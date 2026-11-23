# Messenger

The **Messenger** component collects flash messages during a request and displays them on the next response — or immediately in the same request. Drupal's status message system (`\Drupal::messenger()`) wraps Symfony Messenger.

## Core concepts

| Method | Drupal equivalent | Purpose |
|--------|-------------------|---------|
| `addMessage()` | `addStatus()` | Neutral/informational |
| `addError()` | `addError()` | Error messages |
| `addWarning()` | `addWarning()` | Warnings |
| `all()` | — | Retrieve all messages by type |
| `deleteAll()` | `deleteAll()` | Clear after display |

Messages are **flash** by default: shown once, then removed from session storage.

## How Drupal uses Messenger

### The `messenger` service

```php
<?php

declare(strict_types=1);

/** @var \Drupal\Core\Messenger\MessengerInterface $messenger */
$messenger = \Drupal::service('messenger');

$messenger->addStatus('Product saved.');
$messenger->addWarning('Stock is low.');
$messenger->addError('Payment failed.');
```

`Messenger` stores messages in the session flash bag (the `session.flash_bag` service, Symfony HttpFoundation's `FlashBag`). The `status_messages` render element displays them in the page template.

### MessengerTrait in controllers and forms

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Controller;

use Drupal\Core\Controller\ControllerBase;
use Symfony\Component\HttpFoundation\RedirectResponse;

final class ProductController extends ControllerBase {

  public function publish(int $id): RedirectResponse {
    // …publish logic…
    $this->messenger()->addStatus($this->t('Product @id published.', ['@id' => $id]));
    return $this->redirect('my_module.product_list');
  }

}
```

### AJAX and API requests

Flash messages do not automatically appear in JSON API responses. For headless apps, return messages in the response body or use a separate notification channel.

!!! note "Not Symfony Messenger (async)"
    Despite the name, this component is **not** the async message bus (`symfony/messenger` queue handlers). Drupal's status messenger is the flash message bag only.

## Message types and rendering

=== "Form submit"

    ```php
    <?php

    declare(strict_types=1);

    namespace Drupal\my_module\Form;

    use Drupal\Core\Form\FormBase;
    use Drupal\Core\Form\FormStateInterface;

    final class ImportForm extends FormBase {

      public function submitForm(array &$form, FormStateInterface $form_state): void {
        $imported = $this->doImport();

        $this->messenger()->addStatus($this->formatPlural(
          $imported,
          'Imported 1 product.',
          'Imported @count products.',
          ['@count' => $imported],
        ));

        if ($imported === 0) {
          $this->messenger()->addWarning($this->t('No rows found in the file.'));
        }

        $form_state->setRedirect('my_module.product_list');
      }

    }
    ```

=== "Batch API integration"

    ```php
    <?php

    declare(strict_types=1);

    function my_module_batch_finished(bool $success, array $results, array $operations): void {
      $messenger = \Drupal::messenger();

      if ($success) {
        $messenger->addStatus(t('Processed @count items.', [
          '@count' => $results['processed'] ?? 0,
        ]));
      }
      else {
        $messenger->addError(t('Batch processing encountered an error.'));
      }
    }
    ```

## Custom messenger service (decorator)

Replace or extend message behavior by decorating the messenger service:

```yaml
# my_module.services.yml
services:
  my_module.messenger:
    class: Drupal\my_module\Messenger\LoggingMessenger
    decorates: messenger
    arguments: ['@my_module.messenger.inner', '@logger.channel.my_module']
```

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Messenger;

use Drupal\Core\Messenger\MessengerInterface;
use Psr\Log\LoggerInterface;

final class LoggingMessenger implements MessengerInterface {

  public function __construct(
    private readonly MessengerInterface $inner,
    private readonly LoggerInterface $logger,
  ) {}

  public function addMessage($message, $type = self::TYPE_STATUS, $repeat = FALSE): static {
    $this->logger->info('[{type}] {message}', [
      'type' => $type,
      'message' => (string) $message,
    ]);
    $this->inner->addMessage($message, $type, $repeat);
    return $this;
  }

  public function addError($message, $repeat = FALSE): static {
    return $this->addMessage($message, self::TYPE_ERROR, $repeat);
  }

  public function addStatus($message, $repeat = FALSE): static {
    return $this->addMessage($message, self::TYPE_STATUS, $repeat);
  }

  public function addWarning($message, $repeat = FALSE): static {
    return $this->addMessage($message, self::TYPE_WARNING, $repeat);
  }

  public function all(): array {
    return $this->inner->all();
  }

  // The status_messages element renders the messages these two return.
  public function deleteByType($type): array {
    return $this->inner->deleteByType($type);
  }

  public function deleteAll(): array {
    return $this->inner->deleteAll();
  }

  public function messagesByType($type): array {
    return $this->inner->messagesByType($type);
  }

}
```

## Custom module example: API-aware messenger

Bridge flash messages to JSON for a decoupled admin UI:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\EventSubscriber;

use Drupal\Core\Messenger\MessengerInterface;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\ResponseEvent;
use Symfony\Component\HttpKernel\KernelEvents;

final class ApiMessengerSubscriber implements EventSubscriberInterface {

  public function __construct(
    private readonly MessengerInterface $messenger,
  ) {}

  public static function getSubscribedEvents(): array {
    return [KernelEvents::RESPONSE => ['attachMessages', -20]];
  }

  public function attachMessages(ResponseEvent $event): void {
    $request = $event->getRequest();
    if ($request->getRequestFormat() !== 'json') {
      return;
    }

    $messages = $this->messenger->all();
    if ($messages === []) {
      return;
    }

    $response = $event->getResponse();
    if (!$response instanceof \Symfony\Component\HttpFoundation\JsonResponse) {
      return;
    }

    $data = json_decode($response->getContent(), TRUE) ?? [];
    $data['messages'] = $messages;
    $response->setData($data);
    $this->messenger->deleteAll();
  }

}
```

```yaml
my_module.api_messenger_subscriber:
  class: Drupal\my_module\EventSubscriber\ApiMessengerSubscriber
  arguments: ['@messenger']
  tags:
    - { name: event_subscriber }
```

## Related reading

- [HttpKernel](http-kernel.md) — RESPONSE event for attaching data
- [Controllers](controllers.md) — `messenger()` in controllers
