# Debugging Services

The service container holds thousands of wired objects. When a service is missing, wrong, or stale, you need tools to inspect the container at runtime. This page covers Drush commands, `php:eval` snippets, and Devel helpers for service debugging.

## Quick diagnostics

```bash
# Is Drupal bootstrapped?
drush status

# Does the service exist?
drush php:eval "var_export(\Drupal::getContainer()->has('my_module.product_manager'));"

# What class backs a service?
drush php:eval "echo get_class(\Drupal::service('entity_type.manager'));"

# List services matching a pattern (requires devel 5.x)
drush devel:services | grep product
```

## Inspecting the container

### List all service IDs

```bash
drush php:eval "
\$ids = \Drupal::getContainer()->getServiceIds();
sort(\$ids);
foreach (\$ids as \$id) {
  if (str_contains(\$id, 'my_module')) {
    echo \$id . PHP_EOL;
  }
}
"
```

### Check service definition (compile-time metadata)

With Devel installed:

```bash
drush en devel -y
drush devel:services my_module.product_manager
```

Without Devel — inspect via the container builder during development:

```php
<?php

declare(strict_types=1);

// Only works during container build in a ServiceProvider — not at runtime.
// At runtime, use reflection on the instantiated object:

$service = \Drupal::service('my_module.product_manager');
$reflection = new ReflectionClass($service);
echo $reflection->getFileName() . ':' . $reflection->getStartLine();
```

### Container parameters

```bash
drush php:eval "
\$params = \Drupal::getContainer()->getParameterBag()->all();
echo 'Site path: ' . \$params['site.path'] . PHP_EOL;
echo 'Install profile: ' . \$params['install_profile'] . PHP_EOL;
"
```

```php
<?php

declare(strict_types=1);

// Custom parameters from your module
$timeout = \Drupal::getContainer()->getParameter('my_module.api_timeout');
```

## php:eval recipes

### Instantiate and call a service

```bash
drush php:eval "
\$manager = \Drupal::service('my_module.product_manager');
\$products = \$manager->getPublishedProducts();
echo 'Found ' . count(\$products) . ' products' . PHP_EOL;
"
```

### Debug constructor dependencies

```bash
drush php:eval "
\$service = \Drupal::service('my_module.product_manager');
\$ref = new ReflectionClass(\$service);
\$ctor = \$ref->getConstructor();
foreach (\$ctor->getParameters() as \$param) {
  echo \$param->getName() . ': ' . \$param->getType() . PHP_EOL;
}
"
```

### Compare two environments

```bash
# On local
drush php:eval "echo \Drupal::getContainer()->getParameter('container.build_id');" > /tmp/local-build-id.txt

# On staging (SSH)
drush php:eval "echo \Drupal::getContainer()->getParameter('container.build_id');" > /tmp/staging-build-id.txt

diff /tmp/local-build-id.txt /tmp/staging-build-id.txt
```

Different build IDs mean different compiled containers — expected after different module sets or service YAML.

## Common errors and fixes

### ServiceNotFoundException

```
You have requested a non-existent service "my_module.product_manager".
```

```bash
# 1. Check YAML is named correctly
ls web/modules/custom/my_module/my_module.services.yml

# 2. Check module is enabled
drush pm:list --filter=my_module

# 3. Rebuild container
drush cr

# 4. Verify service ID in YAML matches
grep -A3 'product_manager' web/modules/custom/my_module/my_module.services.yml
```

### Circular reference

```
Circular reference detected for service "my_module.service_a"
```

```yaml
# BAD — circular dependency
services:
  my_module.service_a:
    class: Drupal\my_module\ServiceA
    arguments: ['@my_module.service_b']
  my_module.service_b:
    class: Drupal\my_module\ServiceB
    arguments: ['@my_module.service_a']
```

Fix by introducing a third service, using `lazy: true`, or refactoring to break the cycle.

### Service has wrong class

```bash
drush php:eval "echo get_class(\Drupal::service('my_module.product_manager'));"
# Expected: Drupal\my_module\Service\ProductManager
# If wrong: stale container or incorrect services.yml
drush cr
```

## Debugging tagged services

```bash
drush php:eval "
\$container = \Drupal::getContainer();
\$dispatcher = \$container->get('event_dispatcher');
\$listeners = \$dispatcher->getListeners('kernel.request');
echo count(\$listeners) . ' kernel.request listeners' . PHP_EOL;
"
```

See [Tagged Services](../dependency-injection/tagged-services.md) for tag inspection patterns.

## Create a debug command

For repeated debugging, add a custom Drush command:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Commands;

use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drush\Attributes as CLI;
use Drush\Commands\DrushCommands;

final class ProductDebugCommands extends DrushCommands {

  public function __construct(
    private readonly EntityTypeManagerInterface $entityTypeManager,
  ) {
    parent::__construct();
  }

  #[CLI\Command(name: 'my-module:products', aliases: ['mp'])]
  #[CLI\Usage(name: 'drush my-module:products', description: 'List product entity count')]
  public function products(): void {
    $count = $this->entityTypeManager
      ->getStorage('product')
      ->getQuery()
      ->accessCheck(FALSE)
      ->count()
      ->execute();
    $this->io()->success("Products in database: {$count}");
  }

}
```

```yaml
# drush.services.yml
services:
  my_module.commands:
    class: Drupal\my_module\Commands\ProductDebugCommands
    arguments: ['@entity_type.manager']
    tags:
      - { name: drush.command }
```

```bash
drush my-module:products
```

## Debugging in a browser (temporary)

!!! warning "Never leave this in production"
    Remove debug endpoints before deploying. Exposing container internals is a security risk.

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Controller;

use Drupal\Core\Controller\ControllerBase;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpKernel\Exception\AccessDeniedHttpException;

final class DebugController extends ControllerBase {

  public function services(): JsonResponse {
    if (!\Drupal::currentUser()->hasPermission('administer site configuration')) {
      throw new AccessDeniedHttpException();
    }

    $ids = \Drupal::getContainer()->getServiceIds();
    $mine = array_values(array_filter($ids, fn(string $id): bool => str_starts_with($id, 'my_module')));

    return new JsonResponse([
      'build_id' => \Drupal::getContainer()->getParameter('container.build_id'),
      'my_module_services' => $mine,
    ]);
  }

}
```

## Logging service calls

```yaml
# services.yml — decorate or wrap with logging
services:
  my_module.product_manager.logger:
    class: Drupal\my_module\Service\LoggingProductManager
    decorates: my_module.product_manager
    arguments: ['@.inner', '@logger.channel.my_module']
```

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Service;

use Psr\Log\LoggerInterface;

final class LoggingProductManager implements ProductManagerInterface {

  public function __construct(
    private readonly ProductManagerInterface $inner,
    private readonly LoggerInterface $logger,
  ) {}

  public function getPublishedProducts(): array {
    $this->logger->debug('getPublishedProducts() called');
    return $this->inner->getPublishedProducts();
  }

}
```

## Development modules

| Module | Purpose |
|--------|---------|
| [Devel](https://www.drupal.org/project/devel) | Service list, container dump, query log |
| [Webprofiler](https://www.drupal.org/project/webprofiler) | Symfony profiler toolbar (web only) |
| [Tracy](https://www.drupal.org/project/tracy) | PHP debugger bar |

```bash
composer require --dev drupal/devel
drush en devel devel_generate -y
drush devel:services entity_type.manager
```

!!! tip "Prefer constructor injection in new code"
    When debugging your own services, inject dependencies explicitly. `\Drupal::service()` in production code is acceptable in procedural `.module` files but makes unit testing harder.

## Related reading

- [Service Container](service-container.md) — how the container is built
- [Service Definitions](../dependency-injection/service-definitions.md) — YAML reference
- [Route Inspection](route-inspection.md) — debugging routes
