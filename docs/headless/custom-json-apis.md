# Custom JSON APIs

Not every API fits JSON:API or REST resource plugins. Custom controllers returning `JsonResponse` are ideal for aggregations, third-party integrations, computed fields, and non-entity payloads.

## When to use custom controllers

| Scenario | Approach |
|----------|----------|
| Entity CRUD | JSON:API |
| Custom aggregation / report | Custom controller |
| Webhook receiver | Custom controller |
| Transform external API data | Custom controller + service |
| Alter JSON:API output | Custom normalizer |

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
  methods: [GET]
  options:
    _format: 'json'

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

Normalizers transform entities during serialization. Use them to alter JSON:API output or REST responses.

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

## JSON:API normalizer (alter core output)

To modify JSON:API field output globally, extend `ContentEntityNormalizer`:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_headless\Normalizer;

use Drupal\jsonapi\JsonApiResource\ResourceObject;
use Drupal\jsonapi\Normalizer\ResourceObjectNormalizer;
use Drupal\node\NodeInterface;

/**
 * Adds computed slug alias to JSON:API article resources.
 */
final class ArticleResourceObjectNormalizer extends ResourceObjectNormalizer {

  /**
   * {@inheritdoc}
   */
  public function normalize($object, $format = NULL, array $context = []): array {
    $normalized = parent::normalize($object, $format, $context);

    if ($object instanceof ResourceObject) {
      $entity = $object->getFieldable();
      if ($entity instanceof NodeInterface && $entity->bundle() === 'article') {
        $normalized['attributes']['computed_slug'] = $entity->get('field_slug')->value;
      }
    }

    return $normalized;
  }

}
```

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
