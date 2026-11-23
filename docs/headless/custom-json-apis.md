# Custom JSON APIs

Not every API fits JSON:API or REST resource plugins. Custom controllers returning `JsonResponse` are ideal for aggregations, third-party integrations, computed fields, and non-entity payloads.

## When to use custom controllers

| Scenario | Approach |
|----------|----------|
| Entity CRUD | JSON:API |
| Custom aggregation / report | Custom controller |
| Webhook receiver | Custom controller |
| Transform external API data | Custom controller + service |
| Add data to JSON:API output | Computed field (or JSON:API Extras field enhancer) |

## Custom controller with JsonResponse

### Module files

```
modules/custom/my_headless/
├── my_headless.info.yml
├── my_headless.routing.yml
├── my_headless.permissions.yml
├── my_headless.services.yml
└── src/
    ├── Controller/
    │   └── ArticleFeedController.php
    └── Normalizer/
        └── ArticleFeedNormalizer.php
```

### my_headless.routing.yml

```yaml
my_headless.article_feed:
  path: '/api/v1/articles/feed'
  defaults:
    _controller: '\Drupal\my_headless\Controller\ArticleFeedController::feed'
  requirements:
    _permission: 'access article feed'
    _format: 'json'
  methods: [GET]

my_headless.article_by_slug:
  path: '/api/v1/articles/{slug}'
  defaults:
    _controller: '\Drupal\my_headless\Controller\ArticleFeedController::bySlug'
  requirements:
    _permission: 'access content'
    slug: '[a-z0-9-]+'
  methods: [GET]
```

### ArticleFeedController.php

```php
<?php

declare(strict_types=1);

namespace Drupal\my_headless\Controller;

use Drupal\Core\Cache\CacheableJsonResponse;
use Drupal\Core\Cache\CacheableMetadata;
use Drupal\Core\Controller\ControllerBase;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\node\NodeInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;

/**
 * Returns custom JSON feeds for headless frontends.
 */
final class ArticleFeedController extends ControllerBase {

  public function __construct(
    EntityTypeManagerInterface $entityTypeManager,
  ) {
    $this->entityTypeManager = $entityTypeManager;
  }

  /**
   * {@inheritdoc}
   */
  public static function create(ContainerInterface $container): static {
    return new static(
      $container->get('entity_type.manager'),
    );
  }

  /**
   * Returns a curated article feed.
   */
  public function feed(Request $request): CacheableJsonResponse {
    $limit = min((int) $request->query->get('limit', 10), 50);
    $storage = $this->entityTypeManager->getStorage('node');

    $nids = $storage->getQuery()
      ->accessCheck(TRUE)
      ->condition('type', 'article')
      ->condition('status', NodeInterface::PUBLISHED)
      ->sort('sticky', 'DESC')
      ->sort('created', 'DESC')
      ->range(0, $limit)
      ->execute();

    $nodes = $storage->loadMultiple($nids);
    $items = [];
    $cacheTags = ['node_list:article'];

    foreach ($nodes as $node) {
      $cacheTags = array_merge($cacheTags, $node->getCacheTags());
      $items[] = $this->serializeArticle($node);
    }

    $payload = [
      'meta' => [
        'count' => count($items),
        'generated_at' => gmdate('c'),
      ],
      'items' => $items,
    ];

    $response = new CacheableJsonResponse($payload);

    $cacheability = (new CacheableMetadata())
      ->addCacheTags($cacheTags)
      ->addCacheContexts(['url.query_args:limit'])
      ->setCacheMaxAge(300);

    $response->addCacheableDependency($cacheability);

    return $response;
  }

  /**
   * Returns a single article by URL slug field.
   */
  public function bySlug(string $slug): JsonResponse {
    $storage = $this->entityTypeManager->getStorage('node');

    $nids = $storage->getQuery()
      ->accessCheck(TRUE)
      ->condition('type', 'article')
      ->condition('field_slug', $slug)
      ->condition('status', NodeInterface::PUBLISHED)
      ->range(0, 1)
      ->execute();

    if (empty($nids)) {
      throw new NotFoundHttpException("Article not found: {$slug}");
    }

    $node = $storage->load(reset($nids));
    return new JsonResponse($this->serializeArticle($node));
  }

  /**
   * Builds a frontend-friendly article array.
   */
  private function serializeArticle(NodeInterface $node): array {
    $author = $node->get('field_author')->entity;
    $heroMedia = $node->get('field_hero_media')->entity;
    $imageUrl = NULL;

    if ($heroMedia && !$heroMedia->get('field_media_image')->isEmpty()) {
      $file = $heroMedia->get('field_media_image')->entity;
      if ($file) {
        $imageUrl = \Drupal::service('file_url_generator')
          ->generateAbsoluteString($file->getFileUri());
      }
    }

    return [
      'id' => (int) $node->id(),
      'uuid' => $node->uuid(),
      'slug' => $node->get('field_slug')->value,
      'title' => $node->label(),
      'summary' => $node->get('field_summary')->value,
      'body' => $node->get('body')->processed,
      'created' => gmdate('c', (int) $node->getCreatedTime()),
      'author' => $author ? [
        'name' => $author->getDisplayName(),
        'uuid' => $author->uuid(),
      ] : NULL,
      'image' => $imageUrl,
      'tags' => array_map(
        fn ($term) => ['id' => (int) $term->id(), 'name' => $term->label()],
        $node->get('field_tags')->referencedEntities(),
      ),
    ];
  }

}
```

### Test

```bash
curl -s https://my-site.ddev.site/api/v1/articles/feed?limit=5 | jq .
curl -s https://my-site.ddev.site/api/v1/articles/my-article-slug | jq .title
```

## CacheableJsonResponse

Always use `CacheableJsonResponse` (not plain `JsonResponse`) for cacheable endpoints so Drupal's internal cache and cache tag invalidation work:

```php
$response = new CacheableJsonResponse(['data' => $items]);
$response->addCacheableDependency($node);
$response->addCacheableDependency(
  (new CacheableMetadata())->setCacheMaxAge(3600)
);
```

## Custom normalizers

Normalizers transform entities during serialization. Use them for REST responses and your own `serializer` calls. JSON:API builds its resource objects with its own internal (`jsonapi_normalizer`) normalizers, so an entity-level normalizer like this one does not change JSON:API output (see the next section).

### ArticleTeaserNormalizer.php

```php
<?php

declare(strict_types=1);

namespace Drupal\my_headless\Normalizer;

use Drupal\node\NodeInterface;
use Drupal\serialization\Normalizer\NormalizerBase;

/**
 * Normalizes article nodes to teaser format.
 */
final class ArticleTeaserNormalizer extends NormalizerBase {

  /**
   * {@inheritdoc}
   */
  protected $supportedInterfaceOrClass = NodeInterface::class;

  /**
   * {@inheritdoc}
   */
  public function supportsNormalization($data, ?string $format = NULL, array $context = []): bool {
    if (!parent::supportsNormalization($data, $format, $context)) {
      return FALSE;
    }
    assert($data instanceof NodeInterface);
    return $data->bundle() === 'article'
      && !empty($context['teaser_format']);
  }

  /**
   * {@inheritdoc}
   */
  public function normalize($object, $format = NULL, array $context = []): array|string|int|float|bool|\ArrayObject|NULL {
    assert($object instanceof NodeInterface);

    return [
      'id' => (int) $object->id(),
      'uuid' => $object->uuid(),
      'title' => $object->label(),
      'summary' => $object->get('field_summary')->value,
      'created' => (int) $object->getCreatedTime(),
    ];
  }

  /**
   * {@inheritdoc}
   */
  public function getSupportedTypes(?string $format): array {
    return [
      NodeInterface::class => TRUE,
    ];
  }

}
```

### Register the normalizer

```yaml
# my_headless.services.yml
services:
  my_headless.normalizer.article_teaser:
    class: Drupal\my_headless\Normalizer\ArticleTeaserNormalizer
    tags:
      - { name: normalizer, priority: 20 }
```

### Use in a controller

```php
/** @var \Symfony\Component\Serializer\SerializerInterface $serializer */
$serializer = \Drupal::service('serializer');
$teaser = $serializer->normalize($node, 'json', ['teaser_format' => TRUE]);
```

## Adding data to JSON:API output

JSON:API's normalizers are internal (`@internal`) and are not an extension point: they return `CacheableNormalization` value objects rather than arrays, and services tagged `normalizer` are not used to build JSON:API resource objects. Do not extend `ResourceObjectNormalizer` or `ContentEntityNormalizer` to change JSON:API output. Use a supported approach instead:

- **Computed field** (below): JSON:API exposes every field, including computed ones, as an attribute automatically.
- **[JSON:API Extras](jsonapi.md#jsonapi-extras-contrib) field enhancers**: transform or rename existing fields without code.

### Computed field example

Expose `computed_slug` on article resources:

```php
<?php

// my_headless.module

declare(strict_types=1);

use Drupal\Core\Entity\EntityTypeInterface;
use Drupal\Core\Field\BaseFieldDefinition;
use Drupal\Core\StringTranslation\TranslatableMarkup;
use Drupal\my_headless\Plugin\Field\ComputedSlugItemList;

/**
 * Implements hook_entity_base_field_info().
 */
function my_headless_entity_base_field_info(EntityTypeInterface $entity_type): array {
  $fields = [];
  if ($entity_type->id() === 'node') {
    $fields['computed_slug'] = BaseFieldDefinition::create('string')
      ->setLabel(new TranslatableMarkup('Computed slug'))
      ->setComputed(TRUE)
      ->setReadOnly(TRUE)
      ->setClass(ComputedSlugItemList::class);
  }
  return $fields;
}
```

```php
<?php

declare(strict_types=1);

namespace Drupal\my_headless\Plugin\Field;

use Drupal\Core\Field\FieldItemList;
use Drupal\Core\TypedData\ComputedItemListTrait;

/**
 * Computes the `computed_slug` value for article nodes.
 */
final class ComputedSlugItemList extends FieldItemList {

  use ComputedItemListTrait;

  /**
   * {@inheritdoc}
   */
  protected function computeValue(): void {
    $entity = $this->getEntity();
    if ($entity->bundle() === 'article'
      && $entity->hasField('field_slug')
      && !$entity->get('field_slug')->isEmpty()) {
      $this->list[0] = $this->createItem(0, $entity->get('field_slug')->value);
    }
  }

}
```

After `drush cr`, `computed_slug` appears in `attributes` of `node--article` resources. Computed fields have no storage, so no database update is needed.

## Event subscribers for response alteration

For cross-cutting concerns (CORS headers, API versioning), use an event subscriber:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_headless\EventSubscriber;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\ResponseEvent;
use Symfony\Component\HttpKernel\KernelEvents;

/**
 * Adds API version header to custom endpoints.
 */
final class ApiVersionSubscriber implements EventSubscriberInterface {

  /**
   * {@inheritdoc}
   */
  public static function getSubscribedEvents(): array {
    return [KernelEvents::RESPONSE => 'onResponse'];
  }

  public function onResponse(ResponseEvent $event): void {
    $request = $event->getRequest();
    if (!str_starts_with($request->getPathInfo(), '/api/v1')) {
      return;
    }
    $event->getResponse()->headers->set('X-API-Version', '1.0');
  }

}
```

```yaml
# my_headless.services.yml
services:
  my_headless.api_version_subscriber:
    class: Drupal\my_headless\EventSubscriber\ApiVersionSubscriber
    tags:
      - { name: event_subscriber }
```

## Related pages

- [REST API](rest-api.md) — REST resource plugins
- [JSON:API Deep Dive](jsonapi.md) — standard entity API
- [Caching](../caching/index.md) — cache tags on JSON responses
