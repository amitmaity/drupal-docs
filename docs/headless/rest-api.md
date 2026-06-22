# REST API

Drupal's **REST** module exposes entities and custom data through **REST resource plugins**. JSON:API is preferred for entity CRUD, but REST remains the right tool for bespoke endpoints, aggregations, and non-entity payloads.

## Enable REST

```bash
drush en rest serialization basic_auth -y
drush cr
```

!!! warning "Basic Auth"
    `basic_auth` is acceptable for local development only. Never expose it in production without TLS and rate limiting.

## Core REST resources

Drupal ships REST resources for common entity operations:

| Resource | Plugin ID | Methods |
|----------|-----------|---------|
| Node | `entity:node` | GET, POST, PATCH, DELETE |
| User | `entity:user` | GET, POST, PATCH, DELETE |
| Taxonomy term | `entity:taxonomy_term` | GET, POST, PATCH, DELETE |

Enable a resource via configuration:

```bash
# Enable GET + POST for nodes via REST
drush config:set rest.resource.entity.node GET \
  --input-format=yaml \
  '{
    "id": "entity:node",
    "plugin_id": "entity:node",
    "granularity": "resource",
    "configuration": {
      "methods": ["GET", "POST"],
      "formats": ["json"],
      "authentication": ["cookie", "basic_auth"]
    },
    "status": true
  }'
```

Or create `config/install/rest.resource.entity.node.yml` in a custom module.

## Request and response format

REST uses Drupal's serialization stack. Request `json` format:

```bash
# GET a node
curl -s \
  -u admin:admin \
  -H "Accept: application/json" \
  https://my-site.ddev.site/node/1?_format=json \
  | jq '.title[0].value'

# POST a new node
curl -s -X POST \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  https://my-site.ddev.site/entity/node?_format=json \
  -d '{
    "type": [{"target_id": "article"}],
    "title": [{"value": "Created via REST"}],
    "body": [{"value": "<p>Hello REST</p>", "format": "basic_html"}]
  }'
```

## Custom REST resource plugin

Build a REST endpoint that returns aggregated dashboard data.

### Module structure

```
modules/custom/my_api/
├── my_api.info.yml
├── my_api.permissions.yml
├── my_api.routing.yml          # optional — REST uses plugin routes
├── my_api.services.yml
└── src/
    └── Plugin/rest/resource/
        └── DashboardResource.php
```

### my_api.info.yml

```yaml
name: My API
type: module
description: Custom REST resources for headless dashboard.
core_version_requirement: ^11
package: Custom
dependencies:
  - drupal:rest
  - drupal:serialization
```

### my_api.permissions.yml

```yaml
access dashboard api:
  title: 'Access dashboard REST API'
  description: 'Allows access to the custom dashboard REST endpoint.'
```

### DashboardResource.php

```php
<?php

declare(strict_types=1);

namespace Drupal\my_api\Plugin\rest\resource;

use Drupal\Core\Cache\CacheableMetadata;
use Drupal\Core\Session\AccountProxyInterface;
use Drupal\node\NodeInterface;
use Drupal\rest\ModifiedResourceResponse;
use Drupal\rest\Plugin\ResourceBase;
use Drupal\rest\ResourceResponse;
use Psr\Log\LoggerInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Provides a dashboard summary REST resource.
 *
 * @RestResource(
 *   id = "dashboard_summary",
 *   label = @Translation("Dashboard summary"),
 *   uri_paths = {
 *     "canonical" = "/api/v1/dashboard"
 *   }
 * )
 */
final class DashboardResource extends ResourceBase {

  public function __construct(
    array $configuration,
    $plugin_id,
    $plugin_definition,
    array $serializer_formats,
    LoggerInterface $logger,
    private readonly AccountProxyInterface $currentUser,
  ) {
    parent::__construct($configuration, $plugin_id, $plugin_definition, $serializer_formats, $logger);
  }

  /**
   * {@inheritdoc}
   */
  public static function create(ContainerInterface $container, array $configuration, $plugin_id, $plugin_definition): static {
    return new static(
      $configuration,
      $plugin_id,
      $plugin_definition,
      $container->getParameter('serializer.formats'),
      $container->get('logger.factory')->get('my_api'),
      $container->get('current_user'),
    );
  }

  /**
   * Responds to GET requests.
   */
  public function get(): ResourceResponse {
  $storage = \Drupal::entityTypeManager()->getStorage('node');

    $recent = $storage->getQuery()
      ->accessCheck(TRUE)
      ->condition('type', 'article')
      ->condition('status', NodeInterface::PUBLISHED)
      ->sort('created', 'DESC')
      ->range(0, 5)
      ->execute();

    $nodes = $storage->loadMultiple($recent);
    $articles = [];
    $cacheTags = ['node_list:article'];

    foreach ($nodes as $node) {
      $cacheTags = array_merge($cacheTags, $node->getCacheTags());
      $articles[] = [
        'id' => (int) $node->id(),
        'uuid' => $node->uuid(),
        'title' => $node->label(),
        'created' => (int) $node->getCreatedTime(),
        'url' => $node->toUrl('canonical', ['absolute' => TRUE])->toString(),
      ];
    }

    $data = [
      'user' => [
        'uid' => (int) $this->currentUser->id(),
        'name' => $this->currentUser->getAccountName(),
      ],
      'stats' => [
        'article_count' => (int) $storage->getQuery()
          ->accessCheck(TRUE)
          ->condition('type', 'article')
          ->condition('status', NodeInterface::PUBLISHED)
          ->count()
          ->execute(),
      ],
      'recent_articles' => $articles,
    ];

    $response = new ResourceResponse($data, 200);

    $cacheMetadata = (new CacheableMetadata())
      ->addCacheTags($cacheTags)
      ->addCacheContexts(['user.permissions'])
      ->setCacheMaxAge(300);

    $response->addCacheableDependency($cacheMetadata);

    return $response;
  }

  /**
   * Responds to POST requests — example mutation.
   */
  public function post(array $data): ModifiedResourceResponse {
    if (empty($data['title'])) {
      return new ModifiedResourceResponse(['error' => 'title is required'], 400);
    }

    $node = \Drupal::entityTypeManager()->getStorage('node')->create([
      'type' => 'article',
      'title' => $data['title'],
      'body' => [
        'value' => $data['body'] ?? '',
        'format' => 'basic_html',
      ],
    ]);
    $node->save();

    return new ModifiedResourceResponse([
      'id' => (int) $node->id(),
      'uuid' => $node->uuid(),
      'message' => 'Article created.',
    ], 201);
  }

}
```

### Enable the custom resource

Create `config/install/rest.resource.dashboard_summary.yml`:

```yaml
langcode: en
status: true
dependencies:
  module:
    - my_api
    - serialization
id: dashboard_summary
plugin_id: dashboard_summary
granularity: resource
configuration:
  methods:
    - GET
    - POST
  formats:
    - json
  authentication:
    - cookie
```

Install or import config:

```bash
drush en my_api -y
drush cr
```

### Test the endpoint

```bash
# GET dashboard
curl -s \
  -u admin:admin \
  -H "Accept: application/json" \
  https://my-site.ddev.site/api/v1/dashboard?_format=json \
  | jq .

# POST new article via dashboard endpoint
curl -s -X POST \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  https://my-site.ddev.site/api/v1/dashboard?_format=json \
  -d '{"title": "REST Created Article", "body": "<p>From dashboard POST</p>"}'
```

## REST vs JSON:API

| Feature | REST | JSON:API |
|---------|------|----------|
| Discovery | Manual config per resource | Auto-discovery at `/jsonapi` |
| Filtering | Custom queries only | Standard `filter` param |
| Includes | Manual | `include` for relationships |
| Pagination | Custom | `page[offset]`, `page[limit]` |
| Spec compliance | Drupal-specific | JSON:API 1.1 spec |
| Best for | Custom aggregations | Entity CRUD |

## Serialization formats

Register additional formats in `my_api.services.yml` if needed:

```yaml
services:
  my_api.normalizer.dashboard:
    class: Drupal\my_api\Normalizer\DashboardNormalizer
    tags:
      - { name: normalizer, priority: 10 }
```

## Error handling

REST returns standard HTTP status codes:

| Code | Meaning |
|------|---------|
| 200 | Success (GET) |
| 201 | Created (POST) |
| 204 | No content (DELETE) |
| 400 | Validation error |
| 401 | Not authenticated |
| 403 | Access denied |
| 404 | Resource not found |
| 422 | Unprocessable entity |

## Related pages

- [JSON:API Deep Dive](jsonapi.md) — preferred for entity operations
- [Custom JSON APIs](custom-json-apis.md) — controllers without REST plugins
- [Authentication](auth/index.md) — securing REST endpoints
