# Service Container (DI)

From a developer's perspective, the service container is the object graph of your application. You define services in YAML, Drupal compiles them into a PHP class, and at runtime you receive fully wired instances through constructor injection or `$container->get()`.

## Container build pipeline

```mermaid
flowchart LR
    A[*.services.yml] --> B[ContainerBuilder]
    B --> C[Compiler passes]
    C --> D[Optimized container class]
    D --> E[sites/.../files/php/]
    E --> F[Runtime: get service]
```

## ContainerBuilder

Symfony's `ContainerBuilder` collects definitions at compile time. Drupal's subclass adds CMS-specific passes:

```php
<?php

declare(strict_types=1);

namespace Symfony\Component\DependencyInjection;

// Conceptual — definitions before compilation
$container = new ContainerBuilder();

$container->register('my_module.api_client', ApiClient::class)
  ->addArgument(30)
  ->addArgument(['https://api.example.com'])
  ->addTag('my_module.http_client');

$container->compile();
```

In Drupal you rarely touch `ContainerBuilder` directly — YAML is the source of truth. Use `ServiceProviderInterface` when programmatic registration is required:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module;

use Drupal\Core\DependencyInjection\ContainerBuilder;
use Drupal\Core\DependencyInjection\ServiceProviderInterface;
use Symfony\Component\DependencyInjection\Reference;

final class MyModuleServiceProvider implements ServiceProviderInterface {

  public function register(ContainerBuilder $container): void {
    $container
      ->register('my_module.dynamic_service')
      ->setClass('Drupal\my_module\Service\DynamicService')
      ->addArgument(new Reference('entity_type.manager'));
  }

}
```

## Compiled container

After compilation, Drupal dumps an optimized PHP file. The generated class:

- Inlines factory logic
- Removes runtime YAML parsing
- Resolves circular dependencies via lazy proxies
- Validates references at compile time (not runtime)

```bash
# Trigger recompilation
drush cr

# Container build ID changes on every rebuild
drush php:eval "echo \Drupal::getContainer()->getParameter('container.build_id');"
```

```php
<?php

declare(strict_types=1);

// Generated container (conceptual — do not edit)
class DrupalContainer implements ContainerInterface {

  protected array $services = [];
  protected array $factories = [];

  public function get(string $id, int $invalidBehavior = self::EXCEPTION_ON_INVALID_REFERENCE): ?object {
    if (isset($this->services[$id])) {
      return $this->services[$id];
    }
    return $this->factories[$id]($this);
  }

  protected function getMyModuleProductManagerService(): ProductManager {
    return new ProductManager($this->get('entity_type.manager'));
  }

}
```

!!! note "Never edit generated files"
    Files under `sites/default/files/php/` are regenerated on cache rebuild. All changes go in `*.services.yml` or service providers.

## Service definition structure

```yaml
services:
  # Minimal definition
  my_module.greeter:
    class: Drupal\my_module\Service\Greeter

  # Full definition
  my_module.product_manager:
    class: Drupal\my_module\Service\ProductManager
    arguments: ['@entity_type.manager', '@logger.channel.my_module']
    calls:
      - [setConfig, ['@config.factory']]
    tags:
      - { name: my_module.processor }
    public: true
    lazy: false
    shared: true
```

| Key | Default | Meaning |
|-----|---------|---------|
| `class` | required | PHP class to instantiate |
| `arguments` | `[]` | Constructor arguments |
| `calls` | `[]` | Method calls after construction |
| `tags` | `[]` | Compiler pass collectors |
| `public` | `true` | Accessible via `$container->get()` |
| `lazy` | `false` | Proxy until first use |
| `shared` | `true` | Singleton per request |
| `decorates` | — | Wrap another service |

## Referencing services and parameters

```yaml
parameters:
  my_module.api_base: 'https://api.example.com/v2'

services:
  my_module.http_client:
    class: GuzzleHttp\Client
    arguments:
      - base_uri: '%my_module.api_base%'
        timeout: 10

  my_module.api:
    class: Drupal\my_module\Http\ProductApi
    arguments:
      - '@my_module.http_client'
      - '%my_module.api_base%'
```

Reference syntax:

| Syntax | Meaning |
|--------|---------|
| `@service.id` | Another service |
| `%parameter.name%` | Container parameter |
| `@?service.id` | Optional service (null if missing) |
| `@=service('id').method()` | Expression (advanced) |

## Autowiring (Drupal 11)

Drupal 11 supports Symfony-style autowiring for some core patterns. Custom modules typically use explicit arguments, but `#[Autowire]` is available:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Service;

use Drupal\Core\Entity\EntityTypeManagerInterface;
use Symfony\Component\DependencyInjection\Attribute\Autowire;

final class AutoProductManager {

  public function __construct(
    #[Autowire(service: 'entity_type.manager')]
    private readonly EntityTypeManagerInterface $entityTypeManager,
  ) {}

}
```

See [Constructor Injection](constructor-injection.md) for full autowire examples.

## Public vs private services

```yaml
services:
  # Public — can call \Drupal::service('my_module.public_service')
  my_module.public_service:
    class: Drupal\my_module\PublicService
    public: true

  # Private — only injectable, not directly retrievable
  my_module.internal_helper:
    class: Drupal\my_module\InternalHelper
    public: false
```

!!! tip "Prefer private services"
    Mark internal helpers `public: false`. Only expose service IDs that other modules legitimately need.

## Synthetic and runtime services

Some services are set after compilation:

```php
<?php

declare(strict_types=1);

// Set by DrupalKernel after boot:
// 'request' — current Request object
// 'current_route_match' — matched route for this request
```

Do not define these in your module's `services.yml`.

## Container debugging

```bash
# Service exists?
drush php:eval "var_dump(\Drupal::hasService('my_module.product_manager'));"

# Service class
drush php:eval "echo get_class(\Drupal::service('my_module.product_manager'));"

# Parameter value
drush php:eval "print_r(\Drupal::getContainer()->getParameter('my_module.api_base'));"
```

## Rebuild triggers

| Action | Container rebuilt? |
|--------|-------------------|
| `drush cr` | Yes |
| Enable/disable module | Yes |
| Edit `*.services.yml` | Yes (on next request) |
| Edit PHP service class only | No (opcache may need refresh) |
| Edit service provider | Yes |

```bash
# After any services.yml change
drush cr
```

## Related reading

- [Service Definitions](service-definitions.md) — complete YAML reference
- [Internals: Service Container](../internals/service-container.md) — kernel bootstrap perspective
- [Tagged Services](tagged-services.md) — compiler passes
