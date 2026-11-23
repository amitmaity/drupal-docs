# Controllers

In Symfony, a controller is any callable that returns a `Response`. Drupal standardizes on classes extending **`ControllerBase`**, dependency injection via a static `create()` factory, and return types that are either render arrays or `Response` objects.

## Core concepts

| Pattern | Symfony | Drupal |
|---------|---------|--------|
| Base class | `AbstractController` | `ControllerBase` |
| DI | Constructor autowiring | Constructor autowiring by type (`ControllerBase` uses `AutowireTrait`, 10.2+) or an explicit `create()` |
| Return type | `Response` | `array` (render array) or `Response`/`JsonResponse` |
| Registration | Attributes or YAML | `*.routing.yml`, or Symfony `#[Route]` attributes on classes in `src/Controller` (11.4+) |

## How Drupal uses controllers

### ControllerBase provides shortcuts

`Drupal\Core\Controller\ControllerBase` exposes `entityTypeManager()`, `config()`, `currentUser()`, `t()`, and other helpers:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Controller;

use Drupal\Core\Controller\ControllerBase;

abstract class MyModuleControllerBase extends ControllerBase {
  // Shared helpers for this module.
}
```

### Dependency injection via `create()`

`ControllerBase` implements `ContainerInjectionInterface` and, since Drupal 10.2, uses `AutowireTrait`. Constructor arguments are therefore autowired by type wherever a service is aliased to that class or interface. Write an explicit `create()`, as below, when a dependency has no alias (for example a custom service with no class alias):

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Controller;

use Drupal\Core\Controller\ControllerBase;
use Drupal\Core\DependencyInjection\ContainerInjectionInterface;
use Drupal\my_module\Service\ProductRepository;
use Symfony\Component\DependencyInjection\ContainerInterface;

final class ProductController extends ControllerBase implements ContainerInjectionInterface {

  public function __construct(
    private readonly ProductRepository $productRepository,
  ) {}

  public static function create(ContainerInterface $container): static {
    return new static(
      $container->get('my_module.product_repository'),
    );
  }

  public function list(): array {
    $products = $this->productRepository->findPublished();

    return [
      '#theme' => 'item_list',
      '#title' => $this->t('Products'),
      '#items' => array_map(fn ($p) => $p->label(), $products),
    ];
  }

}
```

```yaml
# my_module.services.yml
services:
  my_module.product_repository:
    class: Drupal\my_module\Service\ProductRepository
    arguments: ['@entity_type.manager', '@database']
  # Optional: this alias lets ControllerBase autowire the repository without create().
  Drupal\my_module\Service\ProductRepository: '@my_module.product_repository'
```

!!! note "No controller service needed"
    Drupal instantiates controllers through the class resolver. It uses a service whose ID is the class name if one exists, and otherwise calls `create()`, which autowires in `ControllerBase`. Symfony's `controller.service_arguments` tag has no effect in Drupal, so you do not need to register the controller as a service.

## JSON controllers

For headless APIs, return `JsonResponse` directly:

=== "Simple JSON endpoint"

    ```php
    <?php

    declare(strict_types=1);

    namespace Drupal\my_module\Controller;

    use Drupal\Core\Controller\ControllerBase;
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Request;

    final class StatsController extends ControllerBase {

      public function summary(): JsonResponse {
        $count = $this->entityTypeManager()
          ->getStorage('node')
          ->getQuery()
          ->accessCheck(TRUE)
          ->condition('status', 1)
          ->count()
          ->execute();

        return new JsonResponse([
          'published_nodes' => $count,
          'generated_at' => date('c'),
        ]);
      }

    }
    ```

=== "POST with validation"

    ```php
    <?php

    declare(strict_types=1);

    namespace Drupal\my_module\Controller;

    use Drupal\Core\Controller\ControllerBase;
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Request;

    final class SubscribeController extends ControllerBase {

      public function subscribe(Request $request): JsonResponse {
        $payload = json_decode($request->getContent(), TRUE, 512, JSON_THROW_ON_ERROR);

        if (empty($payload['email']) || !filter_var($payload['email'], FILTER_VALIDATE_EMAIL)) {
          return new JsonResponse(['error' => 'Invalid email.'], 422);
        }

        // Persist subscription…
        return new JsonResponse(['status' => 'subscribed'], 201);
      }

    }
    ```

## REST controllers

Drupal's REST module uses **REST resource plugins**, not plain controllers. For custom REST resources, extend `ResourceBase`:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Plugin\rest\resource;

use Drupal\Core\StringTranslation\TranslatableMarkup;
use Drupal\rest\Attribute\RestResource;
use Drupal\rest\ModifiedResourceResponse;
use Drupal\rest\Plugin\ResourceBase;
use Drupal\rest\ResourceResponse;
use Psr\Log\LoggerInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

#[RestResource(
  id: 'product',
  label: new TranslatableMarkup('Product'),
  uri_paths: [
    'canonical' => '/api/rest/product/{id}',
    'create' => '/api/rest/product',
  ],
)]
final class ProductResource extends ResourceBase {

  public function __construct(
    array $configuration,
    $plugin_id,
    $plugin_definition,
    array $serializer_formats,
    LoggerInterface $logger,
    private readonly \Drupal\Core\Entity\EntityTypeManagerInterface $entityTypeManager,
  ) {
    parent::__construct($configuration, $plugin_id, $plugin_definition, $serializer_formats, $logger);
  }

  public static function create(ContainerInterface $container, array $configuration, $plugin_id, $plugin_definition): static {
    return new static(
      $configuration,
      $plugin_id,
      $plugin_definition,
      $container->getParameter('serializer.formats'),
      $container->get('logger.factory')->get('rest'),
      $container->get('entity_type.manager'),
    );
  }

  public function get(int $id): ResourceResponse {
    $product = $this->entityTypeManager->getStorage('product')->load($id);
    if ($product === NULL) {
      throw new \Symfony\Component\HttpKernel\Exception\NotFoundHttpException();
    }
    return new ResourceResponse($product);
  }

  public function post(array $data): ModifiedResourceResponse {
    $storage = $this->entityTypeManager->getStorage('product');
    $product = $storage->create($data);
    $product->save();
    return new ModifiedResourceResponse($product, 201);
  }

}
```

Enable the resource in the REST UI (`/admin/config/services/rest`) or via config:

```yaml
# rest.resource.entity.product.yml (config entity)
langcode: en
status: true
dependencies:
  module:
    - my_module
    - serialization
id: product
plugin_id: product
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

## Title callbacks

Routes can use `_title_callback` for dynamic page titles:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Controller;

use Drupal\Core\Controller\ControllerBase;
use Drupal\my_module\Entity\Product;

final class ProductController extends ControllerBase {

  public function title(Product $product): string {
    return $product->label();
  }

}
```

## Custom module example: hybrid page + API controller

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Controller;

use Drupal\Core\Controller\ControllerBase;
use Drupal\Core\DependencyInjection\ContainerInjectionInterface;
use Drupal\my_module\Service\InventoryService;
use Symfony\Component\DependencyInjection\ContainerInterface;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;

final class InventoryController extends ControllerBase implements ContainerInjectionInterface {

  public function __construct(
    private readonly InventoryService $inventory,
  ) {}

  public static function create(ContainerInterface $container): static {
    return new static($container->get('my_module.inventory'));
  }

  /**
   * HTML page with embedded JSON bootstrap data.
   */
  public function dashboard(): array {
    $lowStock = $this->inventory->getLowStockItems();

    return [
      '#theme' => 'my_module_dashboard',
      '#low_stock' => $lowStock,
      '#attached' => [
        'drupalSettings' => [
          'myModule' => ['lowStock' => $lowStock],
        ],
      ],
      '#cache' => [
        'tags' => ['my_module:inventory'],
        'contexts' => ['user.permissions'],
      ],
    ];
  }

  /**
   * JSON endpoint for the same data (headless clients).
   */
  public function dashboardApi(Request $request): JsonResponse {
    if ($request->getRequestFormat() !== 'json') {
      $request->setRequestFormat('json');
    }

    return new JsonResponse([
      'low_stock' => $this->inventory->getLowStockItems(),
    ]);
  }

}
```

```yaml
# my_module.routing.yml
my_module.inventory.dashboard:
  path: '/admin/inventory'
  defaults:
    _controller: '\Drupal\my_module\Controller\InventoryController::dashboard'
    _title: 'Inventory'
  requirements:
    _permission: 'administer my_module'

my_module.inventory.api:
  path: '/api/inventory/low-stock'
  defaults:
    _controller: '\Drupal\my_module\Controller\InventoryController::dashboardApi'
  requirements:
    _permission: 'administer my_module'
  methods: [GET]
```

## Related reading

- [HttpFoundation](http-foundation.md) — `Request` and `JsonResponse`
- [Routing](routing.md) — route definitions
- [Headless: REST API](../headless/rest-api.md)
