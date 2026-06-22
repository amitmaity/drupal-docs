# HttpFoundation

The **HttpFoundation** component provides object-oriented representations of HTTP requests and responses. Every page load, API call, and AJAX request in Drupal flows through Symfony's `Request` and `Response` objects.

## Core concepts

| Class | Purpose |
|-------|---------|
| `Request` | Wraps `$_GET`, `$_POST`, headers, cookies, files, and server variables |
| `Response` | HTTP status, headers, and body sent back to the client |
| `JsonResponse` | `Response` subclass that JSON-encodes data and sets `Content-Type` |
| `Session` | Server-side session storage (Drupal extends this heavily) |
| `Cookie` | Immutable cookie value objects |
| `UploadedFile` | Represents a file from `$_FILES` |

!!! note "Request is the single source of truth"
    In modern Drupal, avoid reading `$_GET` or `$_POST` directly. Inject or type-hint `Symfony\Component\HttpFoundation\Request` in controllers and read from `$request->query`, `$request->request`, and `$request->headers`.

## How Drupal uses HttpFoundation

### DrupalKernel receives every request

`Drupal\Core\DrupalKernel` extends Symfony's `HttpKernel` and accepts a `Request` on every hit:

```php
<?php

declare(strict_types=1);

// core/lib/Drupal/Core/DrupalKernel.php (simplified)
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

final class DrupalKernel extends HttpKernel {

  public function handle(
    Request $request,
    int $type = HttpKernelInterface::MAIN_REQUEST,
    bool $catch = TRUE,
  ): Response {
    // Bootstrap, route matching, controller invocation…
  }
}
```

### Global request via `RequestStack`

Drupal stores the current request in a `RequestStack` service. Subrequests (e.g. internal render calls) push additional requests:

```php
<?php

declare(strict_types=1);

/** @var \Symfony\Component\HttpFoundation\RequestStack $stack */
$stack = \Drupal::service('request_stack');
$request = $stack->getCurrentRequest();
$format = $request->getRequestFormat(); // 'html', 'json', 'api_json', etc.
```

### Sessions and cookies

Drupal's session handling builds on Symfony's session component. The `session` service manages authentication state, CSRF tokens, and tempstore data.

## Request and query parameters

=== "Controller"

    ```php
    <?php

    declare(strict_types=1);

    namespace Drupal\my_module\Controller;

    use Drupal\Core\Controller\ControllerBase;
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Request;

    final class SearchController extends ControllerBase {

      public function search(Request $request): JsonResponse {
        $query = $request->query->getString('q');
        $page = $request->query->getInt('page', 1);
        $limit = min($request->query->getInt('limit', 20), 100);

        $results = $this->searchNodes($query, $page, $limit);

        return new JsonResponse([
          'query' => $query,
          'page' => $page,
          'results' => $results,
        ]);
      }

    }
    ```

=== "Route definition"

    ```yaml
    # my_module.routing.yml
    my_module.search:
      path: '/api/search'
      defaults:
        _controller: '\Drupal\my_module\Controller\SearchController::search'
      requirements:
        _permission: 'access content'
      methods: [GET]
    ```

## JsonResponse for APIs

`JsonResponse` automatically sets status code, `Content-Type: application/json`, and encodes data:

```php
<?php

declare(strict_types=1);

use Symfony\Component\HttpFoundation\JsonResponse;

// Success
return new JsonResponse(['id' => 42, 'title' => 'Hello'], 201);

// Error with custom headers
$response = new JsonResponse(['error' => 'Not found'], 404);
$response->headers->set('X-Request-Id', 'abc-123');
return $response;
```

!!! warning "Cache headers on API responses"
    JSON responses are not cached by Drupal's page cache by default, but you should still set `Cache-Control` explicitly for CDN-facing endpoints. Use Drupal's `cacheable metadata` on render arrays; for raw `JsonResponse`, set headers manually.

## Sessions

=== "Reading session data"

    ```php
    <?php

    declare(strict_types=1);

    namespace Drupal\my_module\Controller;

    use Drupal\Core\Controller\ControllerBase;
    use Symfony\Component\HttpFoundation\Request;

    final class WizardController extends ControllerBase {

      public function step(Request $request): array {
        $session = $request->getSession();
        $step = $session->get('my_module.wizard_step', 1);

        return [
          '#markup' => $this->t('You are on step @step.', ['@step' => $step]),
        ];
      }

    }
    ```

=== "Writing session data"

    ```php
    <?php

    declare(strict_types=1);

    $session = \Drupal::request()->getSession();
    $session->set('my_module.cart', ['sku-1' => 2, 'sku-2' => 1]);
    $session->save();
    ```

!!! tip "Use TempStore for multi-step forms"
    For data that should expire and is keyed by user, prefer `\Drupal\Core\TempStore\PrivateTempStore` over raw session keys. It handles garbage collection and naming collisions.

## Cookies

Symfony's `Cookie` class creates immutable cookie objects. Drupal's `RedirectResponse` and `Response` accept them:

```php
<?php

declare(strict_types=1);

use Symfony\Component\HttpFoundation\Cookie;
use Symfony\Component\HttpFoundation\RedirectResponse;

$response = new RedirectResponse('/admin');
$response->headers->setCookie(Cookie::create(
  name: 'my_module_pref',
  value: 'dark',
  expire: time() + 86400 * 30,
  path: '/',
  secure: TRUE,
  httpOnly: TRUE,
  sameSite: Cookie::SAMESITE_LAX,
));
return $response;
```

## File uploads

`UploadedFile` wraps uploaded files safely:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Form;

use Drupal\Core\Form\FormBase;
use Drupal\Core\Form\FormStateInterface;
use Symfony\Component\HttpFoundation\File\UploadedFile;

final class UploadForm extends FormBase {

  public function submitForm(array &$form, FormStateInterface $form_state): void {
    /** @var \Symfony\Component\HttpFoundation\File\UploadedFile|null $file */
    $file = $form_state->getValue('document');

    if ($file instanceof UploadedFile && $file->isValid()) {
      $destination = 'public://uploads/' . $file->getClientOriginalName();
      $file->move(dirname($destination), basename($destination));
      $this->messenger()->addStatus($this->t('Uploaded @name.', [
        '@name' => $file->getClientOriginalName(),
      ]));
    }
  }

}
```

## Custom module example: headless upload endpoint

Full example accepting a JSON body and a file upload:

=== "Controller"

    ```php
    <?php

    declare(strict_types=1);

    namespace Drupal\my_module\Controller;

    use Drupal\Core\Controller\ControllerBase;
    use Drupal\file\Entity\File;
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Request;

    final class AssetController extends ControllerBase {

      public function upload(Request $request): JsonResponse {
        $upload = $request->files->get('file');

        if ($upload === NULL || !$upload->isValid()) {
          return new JsonResponse(['error' => 'No valid file uploaded.'], 400);
        }

        $data = file_get_contents($upload->getPathname());
        $file = File::create([
          'uri' => 'public://assets/' . $upload->getClientOriginalName(),
          'status' => 1,
        ]);
        file_put_contents($file->getFileUri(), $data);
        $file->save();

        return new JsonResponse([
          'fid' => $file->id(),
          'uri' => $file->getFileUri(),
          'mime' => $upload->getClientMimeType(),
        ], 201);
      }

    }
    ```

=== "Routing"

    ```yaml
    my_module.asset_upload:
      path: '/api/assets/upload'
      defaults:
        _controller: '\Drupal\my_module\Controller\AssetController::upload'
      requirements:
        _permission: 'upload files'
      methods: [POST]
    ```

## Related reading

- [HttpKernel](http-kernel.md) — request lifecycle
- [Controllers](controllers.md) — controller patterns
- [Headless: Custom JSON APIs](../headless/custom-json-apis.md)
