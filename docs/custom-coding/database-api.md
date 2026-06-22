# Database API

Drupal's database abstraction layer wraps PDO with schema API, query builders, and merge queries. Prefer **entity queries** for entity data; use `\Drupal::database()` for custom tables.

## Connection and simple queries

```php
<?php

declare(strict_types=1);

use Drupal\Core\Database\Connection;

/** @var Connection $database */
$database = \Drupal::database();

// Select with conditions.
$result = $database->select('example_module_log', 'l')
  ->fields('l', ['id', 'message', 'created'])
  ->condition('created', strtotime('-7 days'), '>')
  ->orderBy('created', 'DESC')
  ->range(0, 50)
  ->execute();

foreach ($result as $row) {
  echo $row->message;
}

// Insert.
$database->insert('example_module_log')
  ->fields([
    'message' => 'Import completed',
    'created' => \Drupal::time()->getRequestTime(),
  ])
  ->execute();

// Update.
$database->update('example_module_log')
  ->fields(['message' => 'Updated message'])
  ->condition('id', 42)
  ->execute();

// Delete.
$database->delete('example_module_log')
  ->condition('created', strtotime('-90 days'), '<')
  ->execute();
```

---

## Merge (upsert)

```php
<?php

declare(strict_types=1);

\Drupal::database()->merge('example_module_log')
  ->key('id', 1)
  ->fields([
    'message' => 'Upserted row',
    'created' => \Drupal::time()->getRequestTime(),
  ])
  ->execute();
```

---

## Transactions

```php
<?php

declare(strict_types=1);

use Drupal\Core\Database\Connection;

$database = \Drupal::database();
$transaction = $database->startTransaction();

try {
  $database->insert('example_module_log')
    ->fields(['message' => 'Step 1', 'created' => time()])
    ->execute();

  $database->insert('example_module_log')
    ->fields(['message' => 'Step 2', 'created' => time()])
    ->execute();

  // Commit on success (transaction object destroyed without rollback).
}
catch (\Exception $e) {
  $transaction->rollBack();
  throw $e;
}
```

---

## Schema definition — `example_module.install`

```php
<?php

declare(strict_types=1);

/**
 * Implements hook_schema().
 */
function example_module_schema(): array {
  $schema['example_module_log'] = [
    'description' => 'Activity log for example_module.',
    'fields' => [
      'id' => [
        'type' => 'serial',
        'unsigned' => TRUE,
        'not null' => TRUE,
      ],
      'message' => [
        'type' => 'varchar',
        'length' => 512,
        'not null' => TRUE,
      ],
      'severity' => [
        'type' => 'varchar',
        'length' => 32,
        'not null' => TRUE,
        'default' => 'info',
      ],
      'created' => [
        'type' => 'int',
        'not null' => TRUE,
        'default' => 0,
        'size' => 'big',
      ],
    ],
    'primary key' => ['id'],
    'indexes' => [
      'created' => ['created'],
      'severity' => ['severity'],
    ],
  ];

  return $schema;
}
```

---

## Repository service — `src/Repository/LogRepository.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Repository;

use Drupal\Core\Database\Connection;
use Drupal\Core\Database\Query\SelectInterface;

/**
 * Data access for example_module_log table.
 */
final class LogRepository {

  public function __construct(
    private readonly Connection $database,
  ) {}

  public function insert(string $message, string $severity = 'info'): int {
    return (int) $this->database->insert('example_module_log')
      ->fields([
        'message' => $message,
        'severity' => $severity,
        'created' => \Drupal::time()->getRequestTime(),
      ])
      ->execute();
  }

  /**
   * @return list<object{id: int, message: string, severity: string, created: int}>
   */
  public function findRecent(int $limit = 100, ?string $severity = NULL): array {
    $query = $this->database->select('example_module_log', 'l')
      ->fields('l')
      ->orderBy('created', 'DESC')
      ->range(0, $limit);

    if ($severity !== NULL) {
      $query->condition('severity', $severity);
    }

    return $query->execute()->fetchAll();
  }

  public function countBySeverity(): array {
    return $this->database->select('example_module_log', 'l')
      ->fields('l', ['severity'])
      ->addExpression('COUNT(*)', 'total')
      ->groupBy('severity')
      ->execute()
      ->fetchAllKeyed();
  }

}
```

```yaml
# example_module.services.yml
  example_module.log_repository:
    class: Drupal\example_module\Repository\LogRepository
    arguments:
      - '@database'
```

---

## Entity queries

Always prefer entity queries over raw SQL for entity data:

```php
<?php

declare(strict_types=1);

use Drupal\node\Entity\Node;

// Basic query.
$nids = \Drupal::entityQuery('node')
  ->accessCheck(TRUE)
  ->condition('type', 'article')
  ->condition('status', 1)
  ->condition('created', strtotime('-30 days'), '>=')
  ->sort('created', 'DESC')
  ->range(0, 20)
  ->execute();

$nodes = Node::loadMultiple($nids);

// OR condition group.
$query = \Drupal::entityQuery('node')->accessCheck(TRUE);
$or = $query->orConditionGroup()
  ->condition('uid', 1)
  ->condition('promote', 1);
$nids = $query
  ->condition('type', 'article')
  ->condition($or)
  ->execute();

// EXISTS subquery on field table.
$nids = \Drupal::entityQuery('node')
  ->accessCheck(TRUE)
  ->condition('type', 'article')
  ->exists('field_tags')
  ->execute();

// Count.
$count = \Drupal::entityQuery('node')
  ->accessCheck(TRUE)
  ->condition('type', 'article')
  ->count()
  ->execute();

// Aggregate (Drupal 10.2+).
$result = \Drupal::entityQueryAggregate('node')
  ->accessCheck(TRUE)
  ->condition('status', 1)
  ->aggregate('nid', 'COUNT')
  ->groupBy('type')
  ->execute();
```

---

## Entity query in a service

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Service;

use Drupal\Core\Entity\EntityTypeManagerInterface;

final class ArticleFinder {

  public function __construct(
    private readonly EntityTypeManagerInterface $entityTypeManager,
  ) {}

  /**
   * @return list<int|string>
   */
  public function findRecentPublished(int $limit = 10): array {
    return $this->entityTypeManager->getStorage('node')->getQuery()
      ->accessCheck(TRUE)
      ->condition('type', 'article')
      ->condition('status', 1)
      ->sort('created', 'DESC')
      ->range(0, $limit)
      ->execute();
  }

}
```

---

## Dynamic queries with tags

```php
<?php

declare(strict_types=1);

$query = \Drupal::database()->select('node_field_data', 'n');
$query->fields('n', ['nid', 'title']);
$query->condition('status', 1);
$query->addTag('node_access');  // Applies node access grants.
$nids = $query->execute()->fetchAllKeyed(0, 1);
```

!!! warning "Always set accessCheck"
    Entity queries default changed in Drupal 9.2+. Explicitly pass `->accessCheck(TRUE)` or `FALSE` — omitting it triggers a deprecation.

!!! note "Never concatenate user input"
    Use placeholders via `->condition()` and `->args()`. Raw SQL with variables is a SQL injection risk.

!!! tip "Query logging"
    Enable `database.logging: true` in `development.services.yml` and inspect queries with `drush watchdog:show --filter=Database`.
