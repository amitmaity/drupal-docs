# Content Entities

Content entities store fieldable data in the database. This example defines a `Task` entity with base fields, admin forms, and permissions.

## Files

```
example_module/
├── example_module.permissions.yml
├── example_module.links.menu.yml
├── example_module.links.action.yml
└── src/
    ├── Entity/Task.php
    ├── TaskInterface.php
    ├── TaskListBuilder.php
    ├── Form/TaskForm.php
    └── TaskAccessControlHandler.php
```

---

## Entity type — `src/Entity/Task.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Entity;

use Drupal\Core\Entity\Attribute\ContentEntityType;
use Drupal\Core\Entity\ContentEntityBase;
use Drupal\Core\Entity\EntityTypeInterface;
use Drupal\Core\Field\BaseFieldDefinition;
use Drupal\Core\StringTranslation\TranslatableMarkup;
use Drupal\example_module\Form\TaskForm;
use Drupal\example_module\TaskAccessControlHandler;
use Drupal\example_module\TaskInterface;
use Drupal\example_module\TaskListBuilder;

/**
 * Defines the Task entity.
 */
#[ContentEntityType(
  id: 'example_task',
  label: new TranslatableMarkup('Task'),
  label_collection: new TranslatableMarkup('Tasks'),
  label_singular: new TranslatableMarkup('task'),
  label_plural: new TranslatableMarkup('tasks'),
  entity_keys: [
    'id' => 'id',
    'uuid' => 'uuid',
    'label' => 'title',
    'owner' => 'uid',
  ],
  handlers: [
    'list_builder' => TaskListBuilder::class,
    'form' => [
      'default' => TaskForm::class,
      'add' => TaskForm::class,
      'edit' => TaskForm::class,
      'delete' => 'Drupal\Core\Entity\ContentEntityDeleteForm',
    ],
    'access' => TaskAccessControlHandler::class,
    'views_data' => 'Drupal\views\EntityViewsData',
  ],
  links: [
    'canonical' => '/admin/content/task/{example_task}',
    'add-form' => '/admin/content/task/add',
    'edit-form' => '/admin/content/task/{example_task}/edit',
    'delete-form' => '/admin/content/task/{example_task}/delete',
    'collection' => '/admin/content/tasks',
  ],
  admin_permission: 'administer example tasks',
  base_table: 'example_task',
  label_count: [
    'singular' => '@count task',
    'plural' => '@count tasks',
  ],
)]
final class Task extends ContentEntityBase implements TaskInterface {

  /**
   * {@inheritdoc}
   */
  public static function baseFieldDefinitions(EntityTypeInterface $entity_type): array {
    $fields = parent::baseFieldDefinitions($entity_type);

    $fields['title'] = BaseFieldDefinition::create('string')
      ->setLabel(t('Title'))
      ->setRequired(TRUE)
      ->setSetting('max_length', 255)
      ->setDisplayOptions('form', [
        'type' => 'string_textfield',
        'weight' => 0,
      ])
      ->setDisplayOptions('view', [
        'label' => 'hidden',
        'type' => 'string',
        'weight' => 0,
      ]);

    $fields['description'] = BaseFieldDefinition::create('text_long')
      ->setLabel(t('Description'))
      ->setDisplayOptions('form', [
        'type' => 'text_textarea',
        'weight' => 1,
      ])
      ->setDisplayOptions('view', [
        'type' => 'text_default',
        'label' => 'above',
        'weight' => 1,
      ]);

    $fields['status'] = BaseFieldDefinition::create('boolean')
      ->setLabel(t('Completed'))
      ->setDefaultValue(FALSE)
      ->setDisplayOptions('form', [
        'type' => 'boolean_checkbox',
        'weight' => 2,
      ])
      ->setDisplayOptions('view', [
        'type' => 'boolean',
        'label' => 'inline',
        'weight' => 2,
      ]);

    $fields['uid'] = BaseFieldDefinition::create('entity_reference')
      ->setLabel(t('Author'))
      ->setSetting('target_type', 'user')
      ->setDefaultValueCallback(self::class . '::getCurrentUserId')
      ->setDisplayOptions('view', [
        'label' => 'inline',
        'type' => 'author',
        'weight' => 3,
      ]);

    $fields['created'] = BaseFieldDefinition::create('created')
      ->setLabel(t('Created'));

    $fields['changed'] = BaseFieldDefinition::create('changed')
      ->setLabel(t('Changed'));

    return $fields;
  }

  /**
   * Default value callback for the uid field.
   */
  public static function getCurrentUserId(): array {
    return [\Drupal::currentUser()->id()];
  }

  /**
   * {@inheritdoc}
   */
  public function isCompleted(): bool {
    return (bool) $this->get('status')->value;
  }

  /**
   * {@inheritdoc}
   */
  public function setCompleted(bool $completed): static {
    $this->set('status', $completed);
    return $this;
  }

}
```

---

## Interface — `src/TaskInterface.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module;

use Drupal\Core\Entity\ContentEntityInterface;

/**
 * Interface for Task entities.
 */
interface TaskInterface extends ContentEntityInterface {

  public function isCompleted(): bool;

  public function setCompleted(bool $completed): static;

}
```

---

## List builder — `src/TaskListBuilder.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module;

use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Entity\EntityListBuilder;

/**
 * Lists Task entities.
 */
final class TaskListBuilder extends EntityListBuilder {

  /**
   * {@inheritdoc}
   */
  public function buildHeader(): array {
    $header['title'] = $this->t('Title');
    $header['status'] = $this->t('Completed');
    $header['created'] = $this->t('Created');
    return $header + parent::buildHeader();
  }

  /**
   * {@inheritdoc}
   */
  public function buildRow(EntityInterface $entity): array {
    /** @var \Drupal\example_module\TaskInterface $entity */
    $row['title'] = $entity->toLink();
    $row['status'] = $entity->isCompleted() ? $this->t('Yes') : $this->t('No');
    $row['created'] = \Drupal::service('date.formatter')->format($entity->get('created')->value);
    return $row + parent::buildRow($entity);
  }

}
```

---

## Form — `src/Form/TaskForm.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Form;

use Drupal\Core\Entity\ContentEntityForm;
use Drupal\Core\Form\FormStateInterface;

/**
 * Form handler for Task entities.
 */
final class TaskForm extends ContentEntityForm {

  /**
   * {@inheritdoc}
   */
  public function save(array $form, FormStateInterface $form_state): int {
    $result = parent::save($form, $form_state);
    $entity = $this->getEntity();

    $message = $result === SAVED_NEW
      ? $this->t('Created task %title.', ['%title' => $entity->label()])
      : $this->t('Updated task %title.', ['%title' => $entity->label()]);

    $this->messenger()->addStatus($message);
    $form_state->setRedirectUrl($entity->toUrl('collection'));

    return $result;
  }

}
```

---

## Access handler — `src/TaskAccessControlHandler.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module;

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Entity\EntityAccessControlHandler;
use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Session\AccountInterface;

/**
 * Access controller for Task entities.
 */
final class TaskAccessControlHandler extends EntityAccessControlHandler {

  /**
   * {@inheritdoc}
   */
  protected function checkAccess(EntityInterface $entity, $operation, AccountInterface $account): AccessResult {
    /** @var \Drupal\example_module\TaskInterface $entity */
    if ($operation === 'view') {
      return AccessResult::allowedIfHasPermission($account, 'view example tasks');
    }

    if (in_array($operation, ['update', 'delete'], TRUE)) {
      return AccessResult::allowedIfHasPermission($account, 'edit example tasks')
        ->orIf(AccessResult::allowedIf($entity->getOwnerId() === $account->id()));
    }

    return AccessResult::neutral();
  }

  /**
   * {@inheritdoc}
   */
  protected function checkCreateAccess(AccountInterface $account, array $context, $entity_bundle = NULL): AccessResult {
    return AccessResult::allowedIfHasPermission($account, 'create example tasks');
  }

}
```

---

## Permissions — `example_module.permissions.yml`

```yaml
administer example tasks:
  title: 'Administer tasks'
  restrict access: true

view example tasks:
  title: 'View tasks'

create example tasks:
  title: 'Create tasks'

edit example tasks:
  title: 'Edit any task'
```

---

## Menu links — `example_module.links.menu.yml`

```yaml
entity.example_task.collection:
  title: 'Tasks'
  route_name: entity.example_task.collection
  parent: system.admin_content
  weight: 50
```

## Action link — `example_module.links.action.yml`

```yaml
example_task.add:
  route_name: entity.example_task.add_form
  title: 'Add task'
  appears_on:
    - entity.example_task.collection
```

---

## Install and use

```bash
ddev drush cr
ddev drush en example_module -y

# Create via Drush
ddev drush php:eval "
  \$task = \Drupal\example_module\Entity\Task::create(['title' => 'Write docs']);
  \$task->save();
  echo 'Created task ' . \$task->id();
"
```

!!! warning "Install entity schema"
    On first enable, Drupal creates `example_task` tables from `baseFieldDefinitions()`. For schema changes on existing sites, add `hook_update_N()` calling `$entity_definition_update_manager->installFieldStorageDefinition()` or equivalent.

!!! tip "Add configurable fields"
    Set `field_ui_base_route` in the entity attribute to expose the **Manage fields** UI for bundle-less entities.
