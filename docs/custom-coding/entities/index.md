# Entity API

Drupal's Entity API provides a unified way to define, load, save, and display structured data. Custom modules typically work with **content entities** (fieldable, revisionable data) and **config entities** (exportable configuration).

## Chapters

| Guide | Entity kind | Storage | Example |
|-------|-------------|---------|---------|
| [Content Entities](content-entities.md) | Content | SQL (`entity_*` tables) | `Task`, `Product` |
| [Config Entities](config-entities.md) | Configuration | Config system | `Importer`, `Rule` |
| [Custom Storage](custom-storage.md) | Either | Custom backend | Remote API, Redis |

## Content vs config entities

| Feature | Content entity | Config entity |
|---------|----------------|---------------|
| UUID | Yes | Yes |
| Fields | Yes (Field API) | Fixed properties in annotation |
| Revisions | Optional | No |
| Config export | No (content) | Yes |
| Typical use | User-generated data | Site structure settings |
| Admin UI | Entity forms, Views | Config sync |

## Loading entities

```php
<?php

declare(strict_types=1);

use Drupal\Core\Entity\EntityTypeManagerInterface;

/** @var EntityTypeManagerInterface $manager */
$manager = \Drupal::entityTypeManager();

// Load by ID.
$node = $manager->getStorage('node')->load(1);

// Load multiple.
$nodes = $manager->getStorage('node')->loadMultiple([1, 2, 3]);

// Query.
$nids = $manager->getStorage('node')->getQuery()
  ->accessCheck(TRUE)
  ->condition('type', 'article')
  ->condition('status', 1)
  ->sort('created', 'DESC')
  ->range(0, 10)
  ->execute();
```

## Creating and saving

```php
<?php

declare(strict_types=1);

$node = \Drupal::entityTypeManager()->getStorage('node')->create([
  'type' => 'article',
  'title' => 'My article',
  'body' => ['value' => 'Body text', 'format' => 'basic_html'],
  'status' => 1,
]);
$node->save();
```

## Entity interface hierarchy

```
EntityInterface
├── ContentEntityInterface
│   └── Your Task, Product, etc.
└── ConfigEntityInterface
    └── Your Importer, Rule, etc.
```

!!! note "Always use entity API"
    Never write directly to `node_field_data` or custom tables when an entity type exists. Use storage handlers and field API.

!!! tip "Entity updates"
    When changing entity schema, implement `hook_update_N()` to apply changes. See [Content Entities](content-entities.md).
