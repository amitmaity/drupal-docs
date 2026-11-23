# Config Entities

Config entities store exportable configuration — importers, rules, display profiles — in the config system with UUIDs.

## Files

```
example_module/
├── config/schema/example_module.schema.yml
└── src/
    ├── Entity/Importer.php
    ├── ImporterInterface.php
    ├── ImporterListBuilder.php
    └── Form/ImporterForm.php
```

---

## Config schema — `config/schema/example_module.schema.yml`

```yaml
example_module.importer.*:
  type: config_entity
  label: 'Importer config'
  mapping:
    id:
      type: string
      label: 'Machine name'
    label:
      type: label
      label: 'Label'
    uuid:
      type: string
      label: 'UUID'
    source_url:
      type: uri
      label: 'Source URL'
    enabled:
      type: boolean
      label: 'Enabled'
    interval:
      type: integer
      label: 'Run interval in seconds'
```

---

## Entity type — `src/Entity/Importer.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Entity;

use Drupal\Core\Config\Entity\ConfigEntityBase;
use Drupal\Core\Entity\Attribute\ConfigEntityType;
use Drupal\Core\Entity\Routing\AdminHtmlRouteProvider;
use Drupal\Core\StringTranslation\TranslatableMarkup;
use Drupal\example_module\Form\ImporterForm;
use Drupal\example_module\ImporterInterface;
use Drupal\example_module\ImporterListBuilder;

/**
 * Defines the Importer config entity.
 */
#[ConfigEntityType(
  id: 'example_importer',
  label: new TranslatableMarkup('Importer'),
  label_collection: new TranslatableMarkup('Importers'),
  label_singular: new TranslatableMarkup('importer'),
  label_plural: new TranslatableMarkup('importers'),
  config_prefix: 'importer',
  entity_keys: [
    'id' => 'id',
    'label' => 'label',
    'uuid' => 'uuid',
  ],
  handlers: [
    'list_builder' => ImporterListBuilder::class,
    'form' => [
      'add' => ImporterForm::class,
      'edit' => ImporterForm::class,
      'delete' => 'Drupal\Core\Entity\EntityDeleteForm',
    ],
    // Generates the add/edit/delete form and collection routes from the
    // link templates below.
    'route_provider' => [
      'html' => AdminHtmlRouteProvider::class,
    ],
  ],
  links: [
    'add-form' => '/admin/config/system/example-module/importers/add',
    'edit-form' => '/admin/config/system/example-module/importers/{example_importer}',
    'delete-form' => '/admin/config/system/example-module/importers/{example_importer}/delete',
    'collection' => '/admin/config/system/example-module/importers',
  ],
  admin_permission: 'administer example module',
  config_export: [
    'id',
    'label',
    'uuid',
    'source_url',
    'enabled',
    'interval',
  ],
)]
final class Importer extends ConfigEntityBase implements ImporterInterface {

  protected string $id = '';

  protected string $label = '';

  protected string $source_url = '';

  protected bool $enabled = TRUE;

  protected int $interval = 3600;

  /**
   * {@inheritdoc}
   */
  public function getSourceUrl(): string {
    return $this->source_url;
  }

  /**
   * {@inheritdoc}
   */
  public function isEnabled(): bool {
    return $this->enabled;
  }

  /**
   * {@inheritdoc}
   */
  public function getInterval(): int {
    return $this->interval;
  }

}
```

---

## Interface — `src/ImporterInterface.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module;

use Drupal\Core\Config\Entity\ConfigEntityInterface;

/**
 * Interface for Importer config entities.
 */
interface ImporterInterface extends ConfigEntityInterface {

  public function getSourceUrl(): string;

  public function isEnabled(): bool;

  public function getInterval(): int;

}
```

---

## List builder — `src/ImporterListBuilder.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module;

use Drupal\Core\Config\Entity\ConfigEntityListBuilder;
use Drupal\Core\Entity\EntityInterface;

/**
 * Lists Importer config entities.
 */
final class ImporterListBuilder extends ConfigEntityListBuilder {

  /**
   * {@inheritdoc}
   */
  public function buildHeader(): array {
    $header['label'] = $this->t('Label');
    $header['source_url'] = $this->t('Source URL');
    $header['enabled'] = $this->t('Enabled');
    return $header + parent::buildHeader();
  }

  /**
   * {@inheritdoc}
   */
  public function buildRow(EntityInterface $entity): array {
    /** @var \Drupal\example_module\ImporterInterface $entity */
    $row['label'] = $entity->label();
    $row['source_url'] = $entity->getSourceUrl();
    $row['enabled'] = $entity->isEnabled() ? $this->t('Yes') : $this->t('No');
    return $row + parent::buildRow($entity);
  }

}
```

---

## Form — `src/Form/ImporterForm.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Form;

use Drupal\Core\Entity\EntityForm;
use Drupal\Core\Form\FormStateInterface;
use Drupal\example_module\Entity\Importer;

/**
 * Form for Importer config entities.
 */
final class ImporterForm extends EntityForm {

  /**
   * {@inheritdoc}
   */
  public function form(array $form, FormStateInterface $form_state): array {
    $form = parent::form($form, $form_state);
    /** @var \Drupal\example_module\ImporterInterface $importer */
    $importer = $this->entity;

    $form['label'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Label'),
      '#default_value' => $importer->label(),
      '#required' => TRUE,
    ];

    $form['id'] = [
      '#type' => 'machine_name',
      '#default_value' => $importer->id(),
      '#machine_name' => [
        // Must be the concrete class: static methods cannot be called on an
        // interface.
        'exists' => Importer::class . '::load',
      ],
      '#disabled' => !$importer->isNew(),
    ];

    $form['source_url'] = [
      '#type' => 'url',
      '#title' => $this->t('Source URL'),
      '#default_value' => $importer->getSourceUrl(),
      '#required' => TRUE,
    ];

    $form['enabled'] = [
      '#type' => 'checkbox',
      '#title' => $this->t('Enabled'),
      '#default_value' => $importer->isEnabled(),
    ];

    $form['interval'] = [
      '#type' => 'number',
      '#title' => $this->t('Interval (seconds)'),
      '#default_value' => $importer->getInterval(),
      '#min' => 60,
      '#required' => TRUE,
    ];

    return $form;
  }

  /**
   * {@inheritdoc}
   */
  public function save(array $form, FormStateInterface $form_state): int {
    $importer = $this->entity;
    $status = $importer->save();

    $message = $status === SAVED_NEW
      ? $this->t('Created importer %label.', ['%label' => $importer->label()])
      : $this->t('Updated importer %label.', ['%label' => $importer->label()]);

    $this->messenger()->addStatus($message);
    $form_state->setRedirectUrl($importer->toUrl('collection'));

    return $status;
  }

}
```

---

## Default config — `config/install/example_module.importer.default.yml`

```yaml
id: default
label: 'Default importer'
source_url: 'https://example.com/feed.json'
enabled: true
interval: 3600
```

Do not include `uuid` (or `_core`) in `config/install` files — Drupal generates them when the config is installed.

---

## Load and iterate

```php
<?php

declare(strict_types=1);

$storage = \Drupal::entityTypeManager()->getStorage('example_importer');
$importers = $storage->loadMultiple();

foreach ($importers as $importer) {
  if ($importer->isEnabled()) {
    // Run import for $importer->getSourceUrl().
  }
}
```

!!! note "Config export"
    Config entities export as `example_module.importer.{id}.yml` and deploy via `drush config:import`.

!!! warning "config_export list"
    Every property persisted to config must appear in `config_export` on the entity attribute. Omitted properties are dropped on export.
