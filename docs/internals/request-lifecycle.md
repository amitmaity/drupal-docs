# Request Lifecycle

Once Drupal reaches `DRUPAL_BOOTSTRAP_FULL`, every HTTP request flows through Symfony's HttpKernel event pipeline. Drupal extends this pipeline with middleware, route access checkers, and render subscribers. Understanding `KernelEvents` is essential for debugging redirects, authentication failures, and API responses.

## High-level flow

```mermaid
sequenceDiagram
    participant C as Client
    participant IP as index.php
    participant DK as DrupalKernel
    participant HK as HttpKernel
    participant MW as Middleware
    participant RT as Router
    participant AC as Access Manager
    participant CT as Controller
    participant RS as Response

    C->>IP: HTTP Request
    IP->>DK: handle($request)
    DK->>DK: bootstrapToPhase(FULL)
    DK->>HK: handle($request)
    HK->>HK: KernelEvents::REQUEST
    HK->>MW: middleware stack
    MW->>RT: match route
    RT->>AC: check access
    AC->>CT: call controller
    CT->>RS: Response / RenderArray
    HK->>HK: KernelEvents::VIEW (if needed)
    HK->>HK: KernelEvents::RESPONSE
    RS->>C: HTTP Response
    DK->>DK: terminate() → KernelEvents::TERMINATE
```

## KernelEvents reference

Symfony's HttpKernel dispatches these events (Drupal adds subscribers to each):

| Event | Constant | When | Common Drupal use |
|-------|----------|------|-------------------|
| Request | `KernelEvents::REQUEST` | Before routing | Language negotiation, maintenance mode |
| Exception | `KernelEvents::EXCEPTION` | On uncaught exception | Custom error pages, JSON error format |
| Controller | `KernelEvents::CONTROLLER` | After controller resolved | Substitute controller (e.g. `_controller` override) |
| Controller arguments | `KernelEvents::CONTROLLER_ARGUMENTS` | Before controller invoked | Inject route parameters |
| View | `KernelEvents::VIEW` | Controller returned non-Response | Convert render array to Response |
| Response | `KernelEvents::RESPONSE` | Before response sent | Add cache headers, CORS |
| Finish request | `KernelEvents::FINISH_REQUEST` | After response created | Cleanup per-request state |
| Terminate | `KernelEvents::TERMINATE` | After response sent to client | Logging, queue dispatch |

```php
<?php

declare(strict_types=1);

namespace Symfony\Component\HttpKernel;

// KernelEvents.php (excerpt)
final class KernelEvents {
  public const REQUEST = 'kernel.request';
  public const EXCEPTION = 'kernel.exception';
  public const CONTROLLER = 'kernel.controller';
  public const CONTROLLER_ARGUMENTS = 'kernel.controller_arguments';
  public const VIEW = 'kernel.view';
  public const RESPONSE = 'kernel.response';
  public const FINISH_REQUEST = 'kernel.finish_request';
  public const TERMINATE = 'kernel.terminate';
}
```

## DrupalKernel::handle()

```php
<?php

declare(strict_types=1);

namespace Drupal\Core;

use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\HttpKernelInterface;

final class DrupalKernel extends HttpKernel implements HttpKernelInterface {

  public function handle(
    Request $request,
    int $type = self::MAIN_REQUEST,
    bool $catch = TRUE,
  ): Response {
    // Ensure full bootstrap before delegating to HttpKernel.
    $this->boot();
    return parent::handle($request, $type, $catch);
  }

  public function terminate(Request $request, Response $response): void {
    parent::terminate($request, $response);
  }
}
```

## Middleware stack

Drupal wraps the HttpKernel in a **middleware stack** (onion model). Each middleware can modify the request before passing inward or the response on the way out:

```mermaid
graph LR
    R[Request] --> A[StackedHttpKernel]
    A --> M1[Session]
    M1 --> M2[KernelPreHandle]
    M2 --> M3[PageCache]
    M3 --> M4[Negotiation]
    M4 --> M5[ReverseProxy]
    M5 --> HK[HttpKernel]
    HK --> M5
    M5 --> M4
    M4 --> M3
    M3 --> M2
    M2 --> M1
    M1 --> A
    A --> Res[Response]
```

Middleware is registered via `core.services.yml` with the `http_middleware` tag:

```yaml
# core/core.services.yml (excerpt)
services:
  http_middleware.session:
    class: Drupal\Core\StackMiddleware\Session
    tags:
      - { name: http_middleware, priority: 300 }
  http_middleware.page_cache:
    class: Drupal\Core\StackMiddleware\PageCache
    tags:
      - { name: http_middleware, priority: 200, responder: true }
```

!!! tip "Responder middleware"
    Middleware tagged with `responder: true` runs on the response path as well as the request path. Page cache uses this to store responses.

## Routing phase

After `KernelEvents::REQUEST` subscribers run, the router matches the path to a route:

```yaml
# my_module.routing.yml
my_module.product_detail:
  path: '/api/products/{product}'
  defaults:
    _controller: '\Drupal\my_module\Controller\ProductApiController::detail'
    _title: 'Product detail'
  requirements:
    _permission: 'access content'
    product: \d+
  methods: [GET]
  options:
    _format: 'json'
```

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Controller;

use Drupal\Core\Controller\ControllerBase;
use Drupal\my_module\Entity\Product;
use Symfony\Component\HttpFoundation\JsonResponse;

final class ProductApiController extends ControllerBase {

  public function detail(Product $product): JsonResponse {
    return new JsonResponse([
      'id' => $product->id(),
      'title' => $product->label(),
      'sku' => $product->get('sku')->value,
      'price' => (float) $product->get('price')->value,
    ]);
  }
}
```

The `{product}` parameter is resolved via **paramconverter** (`entity:product` upcasting).

## Access checking

Before the controller runs, Drupal runs all access checkers registered for the route:

```mermaid
flowchart TD
    A[Route matched] --> B{Access checkers}
    B --> C["_permission"]
    B --> D["_custom_access"]
    B --> E["_entity_access"]
    B --> F["_user_is_logged_in"]
    C --> G{All allowed?}
    D --> G
    E --> G
    F --> G
    G -->|Yes| H[Invoke controller]
    G -->|No| I[403 Forbidden]
```

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Access;

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Routing\Access\AccessInterface;
use Drupal\Core\Session\AccountInterface;
use Symfony\Component\Routing\Route;

final class ProductAccessCheck implements AccessInterface {

  public function access(Route $route, AccountInterface $account): AccessResult {
    $require_premium = (bool) $route->getRequirement('_premium_only');
    if (!$require_premium) {
      return AccessResult::allowed();
    }
    return AccessResult::allowedIfHasPermission($account, 'view premium products')
      ->addCacheContexts(['user.permissions']);
  }

}
```

## View event: render arrays to Response

When a controller returns a **render array** (not a `Response`), the `KernelEvents::VIEW` event converts it:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Controller;

use Drupal\Core\Controller\ControllerBase;

final class ArticleListController extends ControllerBase {

  public function list(): array {
    $nodes = $this->entityTypeManager()
      ->getStorage('node')
      ->loadByProperties(['type' => 'article', 'status' => 1]);

    $items = [];
    foreach ($nodes as $node) {
      $items[] = $node->toLink()->toString();
    }

    return [
      '#theme' => 'item_list',
      '#title' => $this->t('Recent articles'),
      '#items' => $items,
      '#cache' => [
        'tags' => ['node_list:article'],
        'contexts' => ['user.permissions'],
        'max-age' => 3600,
      ],
    ];
  }
}
```

`Drupal\Core\EventSubscriber\MainContentViewSubscriber` listens on `KernelEvents::VIEW` and wraps the render array in an HTML or JSON response based on the requested format.

## Response event

Subscribers can modify headers before the response is sent:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\EventSubscriber;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\ResponseEvent;
use Symfony\Component\HttpKernel\KernelEvents;

final class ApiCacheHeaderSubscriber implements EventSubscriberInterface {

  public function onResponse(ResponseEvent $event): void {
    if (!$event->isMainRequest()) {
      return;
    }
    $request = $event->getRequest();
    if (!str_starts_with($request->getPathInfo(), '/api/')) {
      return;
    }
    $response = $event->getResponse();
    $response->headers->set('Cache-Control', 'public, max-age=300');
    $response->headers->set('X-Drupal-Cache', 'MISS');
  }

  public static function getSubscribedEvents(): array {
    return [KernelEvents::RESPONSE => ['onResponse', -10]];
  }

}
```

## Terminate phase

After `response->send()`, `DrupalKernel::terminate()` fires `KernelEvents::TERMINATE`. Use this for work that must not delay the response:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\EventSubscriber;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\TerminateEvent;
use Symfony\Component\HttpKernel\KernelEvents;

final class RequestLogTerminateSubscriber implements EventSubscriberInterface {

  public function onTerminate(TerminateEvent $event): void {
    $request = $event->getRequest();
    $response = $event->getResponse();
    \Drupal::logger('my_module')->info('@{method} {path} → {status}', [
      '@method' => $request->getMethod(),
      'path' => $request->getPathInfo(),
      'status' => $response->getStatusCode(),
    ]);
  }

  public static function getSubscribedEvents(): array {
    return [KernelEvents::TERMINATE => ['onTerminate']];
  }

}
```

## JSON:API request path

Headless requests follow the same lifecycle with format negotiation:

```bash
# JSON:API request — Accept header triggers json format
curl -s -H "Accept: application/vnd.api+json" \
  https://example.com/jsonapi/node/article/UUID \
  | jq '.data.attributes.title'
```

```mermaid
flowchart LR
    A[GET /jsonapi/node/article/UUID] --> B[Router: jsonapi.entity]
    B --> C[Access: entity access]
    C --> D[JsonApiController]
    D --> E[Serializer + Normalizers]
    E --> F[JsonResponse 200]
```

## Exception handling

Uncaught exceptions trigger `KernelEvents::EXCEPTION`:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\EventSubscriber;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpKernel\Event\ExceptionEvent;
use Symfony\Component\HttpKernel\Exception\HttpExceptionInterface;
use Symfony\Component\HttpKernel\KernelEvents;

final class ApiExceptionSubscriber implements EventSubscriberInterface {

  public function onException(ExceptionEvent $event): void {
    $request = $event->getRequest();
    if (!str_starts_with($request->getPathInfo(), '/api/')) {
      return;
    }
    $exception = $event->getThrowable();
    $status = $exception instanceof HttpExceptionInterface
      ? $exception->getStatusCode()
      : 500;

    $event->setResponse(new JsonResponse([
      'error' => $exception->getMessage(),
      'code' => $status,
    ], $status));
  }

  public static function getSubscribedEvents(): array {
    return [KernelEvents::EXCEPTION => ['onException', 50]];
  }

}
```

!!! warning "Never catch exceptions in controllers for APIs"
    Use exception subscribers instead. They integrate with Drupal's logging and maintain consistent error shapes for API consumers.

## Related reading

- [Bootstrap Phases](bootstrap.md) — what happens before the request event
- [Routing](routing.md) — route matching and builders
- [Tracing a Request](tracing-request.md) — concrete step-by-step example
