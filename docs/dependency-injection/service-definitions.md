# Service Definitions

Every custom module declares services in `{module_name}.services.yml` at the module root. This file is merged with core and contrib definitions at container compile time. This page is a practical reference with complete, copy-pasteable examples.

## File location and naming

```
modules/custom/my_module/
├── my_module.info.yml
├── my_module.services.yml      ← service definitions
├── my_module.routing.yml
└── src/
    └── Service/
        └── ProductManager.php
```

!!! warning "Filename must match module name"
    `my_module.services.yml` for module `my_module`. A typo means services are silently ignored.

## Basic service

```yaml
# my_module.services.yml
services:
  my_module.product_manager:
    class: Drupal\my_module\Service\ProductManager
    arguments: ['@entity_type.manager']
```

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Service;

use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\my_module\Entity\Product;

final class ProductManager {

  public function __construct(
    private readonly EntityTypeManagerInterface $entityTypeManager,
  ) {}

  public function loadBySku(string $sku): ?Product {
    $ids = $this->entityTypeManager
      ->getStorage('product')
      ->getQuery()
      ->accessCheck(FALSE)
      ->condition('sku', $sku)
      ->range(0, 1)
      ->execute();

    return $ids ? $this->entityTypeManager->getStorage('product')->load(reset($ids)) : NULL;
  }

}
```

## Logger channel

Register a dedicated logger channel for your module:

```yaml
services:
  logger.channel.my_module:
    parent: logger.channel_base
    arguments: ['my_module']

  my_module.product_manager:
    class: Drupal\my_module\Service\ProductManager
    arguments:
      - '@entity_type.manager'
      - '@logger.channel.my_module'
```

```php
<?php

declare(strict_types=1);

use Psr\Log\LoggerInterface;

final class ProductManager {

  public function __construct(
    private readonly EntityTypeManagerInterface $entityTypeManager,
    private readonly LoggerInterface $logger,
  ) {}

  public function archive(int $id): void {
    $product = $this->entityTypeManager->getStorage('product')->load($id);
    if ($product) {
      $product->set('status', FALSE);
      $product->save();
      $this->logger->notice('Archived product @id', ['@id' => $id]);
    }
  }

}
```

## Config injection

```yaml
services:
  my_module.sync_service:
    class: Drupal\my_module\Service\ProductSyncService
    arguments:
      - '@entity_type.manager'
      - '@config.factory'
      - '@http_client'
      - '@logger.channel.my_module'
```

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Service;

use Drupal\Core\Config\ConfigFactoryInterface;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use GuzzleHttp\ClientInterface;
use Psr\Log\LoggerInterface;

final class ProductSyncService {

  public function __construct(
    private readonly EntityTypeManagerInterface $entityTypeManager,
    private readonly ConfigFactoryInterface $configFactory,
    private readonly ClientInterface $httpClient,
    private readonly LoggerInterface $logger,
  ) {}

  public function sync(): int {
    $config = $this->configFactory->get('my_module.settings');
    $endpoint = $config->get('sync_endpoint');
    $apiKey = $config->get('api_key');

    $response = $this->httpClient->request('GET', $endpoint, [
      'headers' => ['Authorization' => "Bearer {$apiKey}"],
    ]);

    $items = json_decode($response->getBody()->getContents(), TRUE, 512, JSON_THROW_ON_ERROR);
    $count = 0;

    foreach ($items as $item) {
      $this->upsertProduct($item);
      $count++;
    }

    $this->logger->info('Synced @count products from @endpoint', [
      '@count' => $count,
      '@endpoint' => $endpoint,
    ]);

    return $count;
  }

  private function upsertProduct(array $item): void {
  }

}
```

## Parameters

```yaml
parameters:
  my_module.sync_endpoint: 'https://api.example.com/products'
  my_module.http_timeout: 15

services:
  my_module.sync_service:
    class: Drupal\my_module\Service\ProductSyncService
    arguments:
      - '@entity_type.manager'
      - '@config.factory'
      - '@http_client'
      - '@logger.channel.my_module'
      - '%my_module.http_timeout%'
```

Override parameters per environment with a site-level services file loaded from `settings.php` (there is no `$container` in settings files):

```php
<?php
// settings.local.php
$settings['container_yamls'][] = $app_root . '/' . $site_path . '/services.local.yml';
```

```yaml
# sites/default/services.local.yml
parameters:
  my_module.sync_endpoint: 'https://staging-api.example.com/products'
```

Run `drush cr` after changing either file.

## Event subscribers

```yaml
services:
  my_module.product_event_subscriber:
    class: Drupal\my_module\EventSubscriber\ProductEventSubscriber
    arguments:
      - '@entity_type.manager'
      - '@queue'
      - '@logger.channel.my_module'
    tags:
      - { name: event_subscriber }
```

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\EventSubscriber;

use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Queue\QueueFactory;
use Psr\Log\LoggerInterface;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\TerminateEvent;
use Symfony\Component\HttpKernel\KernelEvents;

final class ProductEventSubscriber implements EventSubscriberInterface {

  public function __construct(
    private readonly EntityTypeManagerInterface $entityTypeManager,
    private readonly QueueFactory $queueFactory,
    private readonly LoggerInterface $logger,
  ) {}

  public function onTerminate(TerminateEvent $event): void {
  }

  public static function getSubscribedEvents(): array {
    return [KernelEvents::TERMINATE => ['onTerminate']];
  }

}
```

## Access checkers

```yaml
services:
  my_module.premium_access_check:
    class: Drupal\my_module\Access\PremiumAccessCheck
    tags:
      - { name: access_check, applies_to: _premium_access }
```

```yaml
# my_module.routing.yml
my_module.premium_products:
  path: '/products/premium'
  defaults:
    _controller: '\Drupal\my_module\Controller\PremiumController::list'
  requirements:
    _premium_access: 'TRUE'
```

## Param converters

```yaml
services:
  my_module.sku_converter:
    class: Drupal\my_module\ParamConverter\SkuParamConverter
    tags:
      - { name: paramconverter, priority: 10 }
    arguments: ['@entity_type.manager']
```

## Plugin managers

```yaml
services:
  plugin.manager.my_module.report:
    class: Drupal\my_module\ReportPluginManager
    arguments:
      - '@container.namespaces'
      - '@cache.discovery'
      - '@module_handler'
```

## Calls (setter injection)

Prefer constructor injection. Use `calls` only when you must integrate with legacy code:

```yaml
services:
  my_module.legacy_consumer:
    class: Drupal\my_module\LegacyConsumer
    calls:
      - [setLogger, ['@logger.channel.my_module']]
      - [setConfig, ['@config.factory']]
```

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module;

use Drupal\Core\Config\ConfigFactoryInterface;
use Psr\Log\LoggerInterface;

final class LegacyConsumer {

  private LoggerInterface $logger;
  private ConfigFactoryInterface $configFactory;

  public function setLogger(LoggerInterface $logger): void {
    $this->logger = $logger;
  }

  public function setConfig(ConfigFactoryInterface $configFactory): void {
    $this->configFactory = $configFactory;
  }

}
```

## Factory services

When construction logic is complex, use a factory:

```yaml
services:
  my_module.http_client_factory:
    class: Drupal\my_module\Http\HttpClientFactory
    arguments: ['@config.factory']

  my_module.api_client:
    class: GuzzleHttp\Client
    factory: ['@my_module.http_client_factory', 'create']
```

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Http;

use Drupal\Core\Config\ConfigFactoryInterface;
use GuzzleHttp\Client;

final class HttpClientFactory {

  public function __construct(
    private readonly ConfigFactoryInterface $configFactory,
  ) {}

  public function create(): Client {
    $config = $this->configFactory->get('my_module.settings');
    return new Client([
      'base_uri' => $config->get('api_base'),
      'timeout' => (float) $config->get('timeout'),
      'headers' => [
        'Accept' => 'application/json',
      ],
    ]);
  }

}
```

## Interface aliases

```yaml
services:
  my_module.price_calculator:
    class: Drupal\my_module\Service\PriceCalculator

  Drupal\my_module\Service\PriceCalculatorInterface: '@my_module.price_calculator'
```

## Complete module example

```yaml
# my_module.services.yml
parameters:
  my_module.queue_name: 'product_sync'

services:
  logger.channel.my_module:
    parent: logger.channel_base
    arguments: ['my_module']

  my_module.price_calculator:
    class: Drupal\my_module\Service\PriceCalculator

  Drupal\my_module\Service\PriceCalculatorInterface: '@my_module.price_calculator'

  my_module.product_manager:
    class: Drupal\my_module\Service\ProductManager
    arguments:
      - '@entity_type.manager'
      - '@my_module.price_calculator'
      - '@logger.channel.my_module'

  my_module.sync_service:
    class: Drupal\my_module\Service\ProductSyncService
    arguments:
      - '@my_module.product_manager'
      - '@queue'
      - '@config.factory'
      - '%my_module.queue_name%'

  my_module.product_event_subscriber:
    class: Drupal\my_module\EventSubscriber\ProductEventSubscriber
    arguments: ['@my_module.sync_service']
    tags:
      - { name: event_subscriber }

  my_module.premium_access_check:
    class: Drupal\my_module\Access\PremiumAccessCheck
    tags:
      - { name: access_check, applies_to: _premium_access }
```

```bash
# Validate after editing
drush cr
drush php:eval "echo get_class(\Drupal::service('my_module.product_manager'));"
```

## Common mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Wrong filename | Service not found | Rename to `{module}.services.yml` |
| Missing `@` on service ref | String passed to constructor | Use `@entity_type.manager` |
| Wrong service ID | `ServiceNotFoundException` | Check spelling against `core.services.yml` |
| Forgot `drush cr` | Old container | Always rebuild after YAML changes |
| Circular arguments | Compile error | Refactor, or use `lazy: true` with a generated proxy class |

## Related reading

- [Constructor Injection](constructor-injection.md) — receiving services in PHP
- [Service Decoration](service-decoration.md) — wrapping existing services
- [Tagged Services](tagged-services.md) — event subscribers and collectors
