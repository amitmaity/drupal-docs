# Custom Storage

Replace default SQL storage with a custom backend — remote APIs, key-value stores, or hybrid SQL+cache — by implementing a storage handler.

## When to use custom storage

| Scenario | Approach |
|----------|----------|
| Data lives in external API | Subclass `ContentEntityNullStorage` |
| Extra behavior on core storage | Subclass core storage + `hook_entity_type_alter()` |
| Non-SQL backend | Full custom storage class |
| Sync from remote on read | `load()` override with API fetch |

!!! warning "Complexity cost"
    Custom storage bypasses Field API table mapping. Only use when the entity API benefits outweigh the implementation cost.

---

## Entity with custom storage handler

### Entity attribute (excerpt) — `src/Entity/RemoteItem.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Entity;

use Drupal\Core\Entity\Attribute\ContentEntityType;
use Drupal\Core\Entity\ContentEntityBase;
use Drupal\Core\Entity\EntityTypeInterface;
use Drupal\Core\Field\BaseFieldDefinition;
use Drupal\Core\StringTranslation\TranslatableMarkup;
use Drupal\example_module\RemoteItemInterface;
use Drupal\example_module\Storage\RemoteItemStorage;

#[ContentEntityType(
  id: 'example_remote_item',
  label: new TranslatableMarkup('Remote item'),
  entity_keys: [
    'id' => 'id',
    'uuid' => 'uuid',
    'label' => 'title',
  ],
  handlers: [
    'storage' => RemoteItemStorage::class,
    'list_builder' => 'Drupal\example_module\RemoteItemListBuilder',
    'access' => 'Drupal\Core\Entity\EntityAccessControlHandler',
  ],
  links: [
    'canonical' => '/example/remote-item/{example_remote_item}',
    'collection' => '/admin/content/remote-items',
  ],
  admin_permission: 'administer example module',
  base_table: 'example_remote_item',
)]
final class RemoteItem extends ContentEntityBase implements RemoteItemInterface {

  /**
   * {@inheritdoc}
   */
  public static function baseFieldDefinitions(EntityTypeInterface $entity_type): array {
    $fields = parent::baseFieldDefinitions($entity_type);

    $fields['id'] = BaseFieldDefinition::create('string')
      ->setLabel(t('Remote ID'))
      ->setReadOnly(TRUE);

    $fields['title'] = BaseFieldDefinition::create('string')
      ->setLabel(t('Title'))
      ->setRequired(TRUE);

    $fields['payload'] = BaseFieldDefinition::create('string_long')
      ->setLabel(t('Raw payload'));

    return $fields;
  }

}
```

---

## Custom storage — `src/Storage/RemoteItemStorage.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Storage;

use Drupal\Core\Entity\ContentEntityNullStorage;
use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Entity\EntityTypeInterface;
use Drupal\example_module\RemoteApiClient;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Storage handler that reads/writes via a remote API.
 *
 * ContentEntityNullStorage implements every abstract method of
 * ContentEntityStorageBase (doLoadMultiple(), doSave(), has(), field item
 * storage, etc.) as a no-op, so only the public CRUD methods are overridden.
 */
final class RemoteItemStorage extends ContentEntityNullStorage {

  private RemoteApiClient $apiClient;

  /**
   * {@inheritdoc}
   */
  public static function createInstance(ContainerInterface $container, EntityTypeInterface $entity_type): static {
    // The parent passes the entity field manager, caches and bundle info
    // that ContentEntityStorageBase::__construct() requires.
    $instance = parent::createInstance($container, $entity_type);
    $instance->apiClient = $container->get('example_module.remote_api_client');
    return $instance;
  }

  /**
   * {@inheritdoc}
   */
  public function load($id): ?EntityInterface {
    $entities = $this->loadMultiple([$id]);
    return $entities[$id] ?? NULL;
  }

  /**
   * {@inheritdoc}
   *
   * @param array<int|string, int|string> $ids
   *
   * @return array<string, \Drupal\Core\Entity\EntityInterface>
   */
  public function loadMultiple(?array $ids = NULL): array {
    $entities = [];

    if ($ids === NULL) {
      $records = $this->apiClient->fetchAll();
    }
    else {
      $records = [];
      foreach ($ids as $id) {
        $records[] = $this->apiClient->fetch((string) $id);
      }
    }

    foreach ($records as $record) {
      $entity = $this->create([
        'id' => $record['id'],
        'title' => $record['title'],
        'payload' => json_encode($record, JSON_THROW_ON_ERROR),
      ]);
      $entity->enforceIsNew(FALSE);
      $entities[$record['id']] = $entity;
    }

    return $entities;
  }

  /**
   * {@inheritdoc}
   */
  public function save(EntityInterface $entity): int {
    $is_new = $entity->isNew();
    $data = [
      'title' => $entity->label(),
      'payload' => json_decode((string) $entity->get('payload')->value, TRUE, 512, JSON_THROW_ON_ERROR),
    ];

    if ($is_new) {
      $result = $this->apiClient->create($data);
      $entity->set('id', $result['id']);
      $entity->enforceIsNew(FALSE);
      return SAVED_NEW;
    }

    $this->apiClient->update((string) $entity->id(), $data);
    return SAVED_UPDATED;
  }

  /**
   * {@inheritdoc}
   */
  public function delete(array $entities): void {
    foreach ($entities as $entity) {
      $this->apiClient->delete((string) $entity->id());
    }
  }

}
```

---

## API client service — `src/RemoteApiClient.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module;

use GuzzleHttp\ClientInterface;

/**
 * HTTP client wrapper for remote item API.
 */
final class RemoteApiClient {

  public function __construct(
    private readonly ClientInterface $httpClient,
    private readonly string $baseUrl,
  ) {}

  /**
   * @return array<string, mixed>
   */
  public function fetch(string $id): array {
    $response = $this->httpClient->get($this->baseUrl . '/items/' . $id);
    return json_decode((string) $response->getBody(), TRUE, 512, JSON_THROW_ON_ERROR);
  }

  /**
   * @return list<array<string, mixed>>
   */
  public function fetchAll(): array {
    $response = $this->httpClient->get($this->baseUrl . '/items');
    return json_decode((string) $response->getBody(), TRUE, 512, JSON_THROW_ON_ERROR);
  }

  /**
   * @param array<string, mixed> $data
   *
   * @return array<string, mixed>
   */
  public function create(array $data): array {
    $response = $this->httpClient->post($this->baseUrl . '/items', ['json' => $data]);
    return json_decode((string) $response->getBody(), TRUE, 512, JSON_THROW_ON_ERROR);
  }

  /**
   * @param array<string, mixed> $data
   */
  public function update(string $id, array $data): void {
    $this->httpClient->put($this->baseUrl . '/items/' . $id, ['json' => $data]);
  }

  public function delete(string $id): void {
    $this->httpClient->delete($this->baseUrl . '/items/' . $id);
  }

}
```

```yaml
# example_module.services.yml
  example_module.remote_api_client:
    class: Drupal\example_module\RemoteApiClient
    arguments:
      - '@http_client'
      - 'https://api.example.com'
```

---

## Extend an existing storage handler

Storage handlers are not container services, so they cannot be wrapped with `decorates:` in `services.yml`. To add behavior to core storage, subclass it and swap the class in `hook_entity_type_alter()`:

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Storage;

use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Entity\EntityTypeInterface;
use Drupal\node\NodeStorage;
use Psr\Log\LoggerInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Node storage that logs every save.
 */
final class LoggingNodeStorage extends NodeStorage {

  private LoggerInterface $logger;

  /**
   * {@inheritdoc}
   */
  public static function createInstance(ContainerInterface $container, EntityTypeInterface $entity_type): static {
    $instance = parent::createInstance($container, $entity_type);
    $instance->logger = $container->get('logger.factory')->get('example_module');
    return $instance;
  }

  /**
   * {@inheritdoc}
   */
  protected function doPostSave(EntityInterface $entity, $update) {
    parent::doPostSave($entity, $update);
    $this->logger->info('Saved node @id.', ['@id' => $entity->id()]);
  }

}
```

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Hook;

use Drupal\Core\Hook\Attribute\Hook;
use Drupal\example_module\Storage\LoggingNodeStorage;

/**
 * Entity type alterations.
 */
final class EntityTypeHooks {

  /**
   * Implements hook_entity_type_alter().
   *
   * @param array<string, \Drupal\Core\Entity\EntityTypeInterface> $entity_types
   */
  #[Hook('entity_type_alter')]
  public function entityTypeAlter(array &$entity_types): void {
    $entity_types['node']->setStorageClass(LoggingNodeStorage::class);
  }

}
```

!!! note "Load caching is built in"
    Content entities already use the persistent entity cache (`cache.entity`) and a static memory cache, so a separate cache layer around `load()` is rarely needed. Only one module can own an entity type's storage class — check for conflicts with contrib modules that alter the same entity type.

!!! tip "Entity query limitations"
    `ContentEntityNullStorage` uses the `entity.query.null` service, whose queries always return no results. Override `getQueryServiceName()` with a custom entity query service if filtering is required.

!!! note "Static cache"
    Call `$storage->resetCache([$id])` after external updates so `load()` does not return stale entities.
