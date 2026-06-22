# Custom Storage

Replace default SQL storage with a custom backend — remote APIs, key-value stores, or hybrid SQL+cache — by implementing a storage handler.

## When to use custom storage

| Scenario | Approach |
|----------|----------|
| Data lives in external API | Custom `ContentEntityStorage` |
| Read-heavy, cache-first | Decorator around default storage |
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

use Drupal\Core\Entity\ContentEntityStorageBase;
use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Entity\EntityTypeInterface;
use Drupal\example_module\RemoteApiClient;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Storage handler that reads/writes via a remote API.
 */
final class RemoteItemStorage extends ContentEntityStorageBase {

  public function __construct(
    EntityTypeInterface $entity_type,
    private readonly RemoteApiClient $apiClient,
  ) {
    parent::__construct($entity_type);
  }

  /**
   * {@inheritdoc}
   */
  public static function createInstance(ContainerInterface $container, EntityTypeInterface $entity_type): static {
    return new static(
      $entity_type,
      $container->get('example_module.remote_api_client'),
    );
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

## Storage decorator (cache layer)

Wrap default storage without replacing it entirely:

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Storage;

use Drupal\Core\Cache\CacheBackendInterface;
use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Entity\EntityStorageInterface;
use Drupal\Core\Entity\EntityTypeInterface;

/**
 * Caches entity load results.
 */
final class CachedNodeStorage implements EntityStorageInterface {

  public function __construct(
    private readonly EntityStorageInterface $inner,
    private readonly CacheBackendInterface $cache,
  ) {}

  public function load($id): ?EntityInterface {
    $cid = 'example_node:' . $id;
    if ($cache = $this->cache->get($cid)) {
      return $cache->data;
    }

    $entity = $this->inner->load($id);
    if ($entity !== NULL) {
      $this->cache->set($cid, $entity, CacheBackendInterface::CACHE_PERMANENT, $entity->getCacheTags());
    }
    return $entity;
  }

  // Delegate all other EntityStorageInterface methods to $this->inner...
  public function loadMultiple(?array $ids = NULL): array {
    return $this->inner->loadMultiple($ids);
  }

  public function loadByProperties(array $values = []): array {
    return $this->inner->loadByProperties($values);
  }

  public function loadUnchanged($id): ?EntityInterface {
    return $this->inner->loadUnchanged($id);
  }

  public function create(array $values = []): EntityInterface {
    return $this->inner->create($values);
  }

  public function save(EntityInterface $entity) {
    return $this->inner->save($entity);
  }

  public function delete(array $entities) {
    $this->inner->delete($entities);
  }

  public function restore(EntityInterface $entity) {
    return $this->inner->restore($entity);
  }

  public function resetCache(?array $ids = NULL) {
    $this->inner->resetCache($ids);
  }

  public function getQuery($conjunction = 'AND') {
    return $this->inner->getQuery($conjunction);
  }

  public function getAggregateQuery($conjunction = 'AND') {
    return $this->inner->getAggregateQuery($conjunction);
  }

  public function getEntityTypeId(): string {
    return $this->inner->getEntityTypeId();
  }

  public function hasData(): bool {
    return $this->inner->hasData();
  }

}
```

```yaml
# example_module.services.yml — decorate core node storage
  example_module.cached_node_storage:
    class: Drupal\example_module\Storage\CachedNodeStorage
    decorates: entity_type.manager
    decoration_inner_name: example_module.cached_node_storage.inner
    arguments:
      - '@example_module.cached_node_storage.inner'
      - '@cache.default'
```

!!! tip "Entity query limitations"
    Custom storage may not support `getQuery()`. Throw `UnsupportedEntityStorageException` or implement a custom query class if filtering is required.

!!! note "Static cache"
    Call `$storage->resetCache([$id])` after external updates so `load()` does not return stale entities.
