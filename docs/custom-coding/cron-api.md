# Cron API

Drupal cron runs periodic maintenance: queue processing, search indexing, and custom tasks via `hook_cron` or dedicated services.

## OOP cron hook — `src/Hook/CronHooks.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Hook;

use Drupal\Core\Hook\Attribute\Hook;
use Drupal\example_module\Service\CronTasks;

/**
 * Cron hook implementations.
 */
final class CronHooks {

  public function __construct(
    private readonly CronTasks $cronTasks,
  ) {}

  /**
   * Implements hook_cron().
   */
  #[Hook('cron')]
  public function cron(): void {
    $this->cronTasks->run();
  }

}
```

---

## Cron service — `src/Service/CronTasks.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Service;

use Drupal\Core\Config\ConfigFactoryInterface;
use Drupal\Core\Database\Connection;
use Drupal\Core\Queue\QueueFactory;
use Drupal\Core\State\StateInterface;
use Psr\Log\LoggerInterface;

/**
 * Periodic tasks for example_module.
 */
final class CronTasks {

  private const STATE_LAST_RUN = 'example_module.cron_last_run';

  public function __construct(
    private readonly Connection $database,
    private readonly QueueFactory $queueFactory,
    private readonly ConfigFactoryInterface $configFactory,
    private readonly StateInterface $state,
    private readonly LoggerInterface $logger,
  ) {}

  public function run(): void {
    $last_run = (int) $this->state->get(self::STATE_LAST_RUN, 0);
    $now = \Drupal::time()->getRequestTime();

    $this->purgeOldLogs($now);
    $this->enqueueDueImports($now);
    $this->state->set(self::STATE_LAST_RUN, $now);

    $this->logger->info('Cron completed. Last run was @last.', [
      '@last' => $last_run ? date('c', $last_run) : 'never',
    ]);
  }

  /**
   * Deletes log records older than 30 days.
   */
  private function purgeOldLogs(int $now): void {
    $cutoff = $now - (30 * 86400);

    $deleted = $this->database->delete('example_module_log')
      ->condition('created', $cutoff, '<')
      ->execute();

    if ($deleted > 0) {
      $this->logger->info('Purged @count old log entries.', ['@count' => $deleted]);
    }
  }

  /**
   * Enqueues importers whose interval has elapsed.
   */
  private function enqueueDueImports(int $now): void {
    if (!$this->configFactory->get('example_module.settings')->get('enable_logging')) {
      return;
    }

    $storage = \Drupal::entityTypeManager()->getStorage('example_importer');
    $queue = $this->queueFactory->get('example_module_import');

    foreach ($storage->loadMultiple() as $importer) {
      if (!$importer->isEnabled()) {
        continue;
      }

      $key = 'example_module.importer.' . $importer->id() . '.last_run';
      $last = (int) $this->state->get($key, 0);

      if (($now - $last) >= $importer->getInterval()) {
        $queue->createItem(['importer_id' => $importer->id()]);
        $this->state->set($key, $now);
      }
    }
  }

}
```

```yaml
# example_module.services.yml
  example_module.cron_tasks:
    class: Drupal\example_module\Service\CronTasks
    arguments:
      - '@database'
      - '@queue'
      - '@config.factory'
      - '@state'
      - '@logger.channel.example_module'

  Drupal\example_module\Hook\CronHooks:
    arguments:
      - '@example_module.cron_tasks'
```

---

## Legacy procedural cron

```php
<?php

declare(strict_types=1);

/**
 * Implements hook_cron().
 */
function example_module_cron(): void {
  \Drupal::service('example_module.cron_tasks')->run();
}
```

---

## Queue worker for import cron jobs

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Plugin\QueueWorker;

use Drupal\Core\Plugin\ContainerFactoryPluginInterface;
use Drupal\Core\Queue\Attribute\QueueWorker;
use Drupal\Core\Queue\QueueWorkerBase;
use Drupal\Core\StringTranslation\TranslatableMarkup;
use Psr\Log\LoggerInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

#[QueueWorker(
  id: 'example_module_import',
  title: new TranslatableMarkup('Example import queue'),
  cron: ['time' => 120],
)]
final class ImportQueueWorker extends QueueWorkerBase implements ContainerFactoryPluginInterface {

  public function __construct(
    array $configuration,
    $plugin_id,
    $plugin_definition,
    private readonly LoggerInterface $logger,
  ) {
    parent::__construct($configuration, $plugin_id, $plugin_definition);
  }

  /**
   * {@inheritdoc}
   */
  public static function create(ContainerInterface $container, array $configuration, $plugin_id, $plugin_definition): static {
    return new static(
      $configuration,
      $plugin_id,
      $plugin_definition,
      $container->get('logger.channel.example_module'),
    );
  }

  /**
   * @param array{importer_id: string} $data
   */
  public function processItem($data): void {
    $this->logger->info('Running import for @id', ['@id' => $data['importer_id']]);
    // Fetch from $data['importer_id'] and process.
  }

}
```

---

## Run cron manually

```bash
# Run all cron hooks
ddev drush cron

# Run with verbose logging
ddev drush cron -v

# Check last cron run
ddev drush state:get system.cron_last
```

---

## Automated cron settings

In `settings.php` for production:

```php
// Disable poor man's cron (web-triggered); use system cron instead.
$config['automated_cron.settings']['interval'] = 0;
```

System crontab:

```cron
*/15 * * * * cd /var/www/html && vendor/bin/drush cron
```

!!! warning "Long-running cron"
    Keep `hook_cron()` fast. Heavy work belongs in queue workers with `cron: ['time' => N]` limits.

!!! tip "State vs config"
    Use `State` for last-run timestamps (not exported). Use `Config` for operator-tunable intervals.

!!! note "Ultimate Cron module"
    For per-module scheduling and logging UI, consider the contrib **Ultimate Cron** module. Core cron runs all implementations in one pass.
