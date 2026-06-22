# Queue Workers

Queue workers process items from reliable queues asynchronously. Items are claimed by cron or `drush queue:run`.

## Queue worker plugin — `src/Plugin/QueueWorker/EmailQueueWorker.php`

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

/**
 * Sends queued email notifications.
 */
#[QueueWorker(
  id: 'example_module_email',
  title: new TranslatableMarkup('Example email queue'),
  cron: ['time' => 60],
)]
final class EmailQueueWorker extends QueueWorkerBase implements ContainerFactoryPluginInterface {

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
   * {@inheritdoc}
   *
   * @param array{to: string, subject: string, body: string} $data
   */
  public function processItem($data): void {
    if (!is_array($data) || !isset($data['to'], $data['subject'], $data['body'])) {
      throw new \InvalidArgumentException('Invalid queue item payload.');
    }

    // Replace with MailManagerInterface::mail() in production.
    $this->logger->info('Queued email to @to: @subject', [
      '@to' => $data['to'],
      '@subject' => $data['subject'],
    ]);
  }

}
```

---

## Logger channel — `example_module.services.yml`

```yaml
services:
  logger.channel.example_module:
    parent: logger.channel_base
    arguments: ['example_module']
```

---

## Enqueue items

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Service;

use Drupal\Core\Queue\QueueFactory;

/**
 * Enqueues email notifications.
 */
final class EmailQueue {

  public function __construct(
    private readonly QueueFactory $queueFactory,
  ) {}

  public function enqueue(string $to, string $subject, string $body): void {
    $queue = $this->queueFactory->get('example_module_email');
    $queue->createItem([
      'to' => $to,
      'subject' => $subject,
      'body' => $body,
    ]);
  }

}
```

```yaml
# example_module.services.yml
  example_module.email_queue:
    class: Drupal\example_module\Service\EmailQueue
    arguments:
      - '@queue'
```

---

## Process the queue

```bash
# Process all items in the email queue
ddev drush queue:run example_module_email

# Process with a time limit (seconds)
ddev drush queue:run example_module_email --time-limit=120

# List queues and item counts
ddev drush queue:list
```

Cron automatically processes queues annotated with `cron: ['time' => N]`.

---

## Delayed queues

Use the queue's delay API for retry with backoff:

```php
$queue = \Drupal::queue('example_module_email');
$queue->createItem($data);

// Re-queue failed item with 5-minute delay (if supported by backend).
if (method_exists($queue, 'delayItem')) {
  $queue->delayItem($item, 300);
}
```

!!! note "Reliable vs unreliable"
    `\Drupal::queue()` returns a **reliable** database-backed queue by default. Use `\Drupal::service('queue')->get('name', FALSE)` for memory-only unreliable queues.

!!! warning "Idempotent processing"
    Design `processItem()` to be safe if run twice. Items may be retried after timeouts or crashes.

See also: [Queue API](../queue-api.md) for low-level queue operations.
