# Custom Field Types

A complete custom field requires three plugins: **field type** (data storage), **widget** (form input), and **formatter** (display output).

## Module layout

```
example_module/
├── config/schema/example_module.schema.yml
└── src/Plugin/Field/
    ├── FieldType/
    │   └── ColorFieldType.php
    ├── FieldWidget/
    │   └── ColorWidget.php
    └── FieldFormatter/
        └── ColorSwatchFormatter.php
```

---

## Field type — `src/Plugin/Field/FieldType/ColorFieldType.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Plugin\Field\FieldType;

use Drupal\Core\Field\Attribute\FieldType;
use Drupal\Core\Field\FieldDefinitionInterface;
use Drupal\Core\Field\FieldItemBase;
use Drupal\Core\Field\FieldStorageDefinitionInterface;
use Drupal\Core\StringTranslation\TranslatableMarkup;
use Drupal\Core\TypedData\DataDefinition;

/**
 * Defines the Color field type.
 */
#[FieldType(
  id: 'example_color',
  label: new TranslatableMarkup('Color'),
  description: new TranslatableMarkup('Stores a hex color value.'),
  default_widget: 'example_color_widget',
  default_formatter: 'example_color_swatch',
)]
final class ColorFieldType extends FieldItemBase {

  /**
   * {@inheritdoc}
   */
  public static function propertyDefinitions(FieldStorageDefinitionInterface $field_definition): array {
    $properties['value'] = DataDefinition::create('string')
      ->setLabel(t('Hex color'))
      ->setRequired(TRUE)
      // Checked whenever the entity is validated (entity forms, REST,
      // JSON:API, or an explicit $entity->validate()).
      ->addConstraint('Regex', [
        'pattern' => '/^#[0-9A-Fa-f]{6}$/',
        'message' => 'Enter a valid hex color (e.g. #FF0000).',
      ]);

    return $properties;
  }

  /**
   * {@inheritdoc}
   */
  public static function schema(FieldStorageDefinitionInterface $field_definition): array {
    return [
      'columns' => [
        'value' => [
          'type' => 'varchar',
          'length' => 7,
        ],
      ],
    ];
  }

  /**
   * {@inheritdoc}
   */
  public static function defaultFieldSettings(): array {
    return [
      'default_color' => '#000000',
    ] + parent::defaultFieldSettings();
  }

  /**
   * {@inheritdoc}
   */
  public static function generateSampleValue(FieldDefinitionInterface $field_definition): array {
    return ['value' => '#336699'];
  }

  /**
   * {@inheritdoc}
   */
  public function isEmpty(): bool {
    $value = $this->get('value')->getValue();
    return $value === NULL || $value === '';
  }

  /**
   * {@inheritdoc}
   */
  public static function mainPropertyName(): string {
    return 'value';
  }

}
```

---

## Config schema — `config/schema/example_module.schema.yml`

Field settings and default values are stored in `field.field.*` config, so they need schema. Without it, saving the field fails strict schema checks in Kernel and Functional tests.

```yaml
field.field_settings.example_color:
  type: mapping
  label: 'Color field settings'
  mapping:
    default_color:
      type: string
      label: 'Default color'

field.value.example_color:
  type: mapping
  label: 'Default value'
  mapping:
    value:
      type: string
      label: 'Hex color'
```

The field type has no storage settings, so core's `field.storage_settings.*` fallback covers storage config.

---

## Widget — `src/Plugin/Field/FieldWidget/ColorWidget.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Plugin\Field\FieldWidget;

use Drupal\Core\Field\Attribute\FieldWidget;
use Drupal\Core\Field\FieldItemListInterface;
use Drupal\Core\Field\WidgetBase;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\StringTranslation\TranslatableMarkup;

/**
 * Color picker widget using HTML5 color input.
 */
#[FieldWidget(
  id: 'example_color_widget',
  label: new TranslatableMarkup('Color picker'),
  field_types: ['example_color'],
)]
final class ColorWidget extends WidgetBase {

  /**
   * {@inheritdoc}
   */
  public function formElement(FieldItemListInterface $items, $delta, array $element, array &$form, FormStateInterface $form_state): array {
    $default = $items[$delta]->value
      ?? $this->getFieldSetting('default_color')
      ?? '#000000';

    $element['value'] = [
      '#type' => 'color',
      '#title' => $this->t('Color'),
      '#default_value' => $default,
      '#required' => $this->fieldDefinition->isRequired(),
    ];

    return $element;
  }

  /**
   * {@inheritdoc}
   */
  public function massageFormValues(array $values, array $form, FormStateInterface $form_state): array {
    foreach ($values as &$value) {
      if (isset($value['value'])) {
        $value['value'] = strtoupper((string) $value['value']);
      }
    }
    return $values;
  }

}
```

---

## Formatter — `src/Plugin/Field/FieldFormatter/ColorSwatchFormatter.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Plugin\Field\FieldFormatter;

use Drupal\Core\Field\Attribute\FieldFormatter;
use Drupal\Core\Field\FieldItemListInterface;
use Drupal\Core\Field\FormatterBase;
use Drupal\Core\StringTranslation\TranslatableMarkup;

/**
 * Renders a color swatch with hex label.
 */
#[FieldFormatter(
  id: 'example_color_swatch',
  label: new TranslatableMarkup('Color swatch'),
  field_types: ['example_color'],
)]
final class ColorSwatchFormatter extends FormatterBase {

  /**
   * {@inheritdoc}
   */
  public function viewElements(FieldItemListInterface $items, $langcode): array {
    $elements = [];

    foreach ($items as $delta => $item) {
      $color = (string) $item->value;

      $elements[$delta] = [
        '#type' => 'inline_template',
        '#template' => '<span class="color-swatch" style="background-color: {{ color }};" title="{{ color }}"></span> <code>{{ color }}</code>',
        '#context' => ['color' => $color],
        '#attached' => [
          'library' => ['example_module/color_swatch'],
        ],
      ];
    }

    return $elements;
  }

}
```

---

## CSS library — `example_module.libraries.yml`

```yaml
color_swatch:
  css:
    theme:
      css/color-swatch.css: {}
```

## `css/color-swatch.css`

```css
.color-swatch {
  display: inline-block;
  width: 1.5rem;
  height: 1.5rem;
  border: 1px solid #ccc;
  border-radius: 3px;
  vertical-align: middle;
  margin-right: 0.5rem;
}
```

---

## Attach field to an entity

### Programmatically

```php
<?php

declare(strict_types=1);

use Drupal\field\Entity\FieldConfig;
use Drupal\field\Entity\FieldStorageConfig;

// Storage (shared across bundles).
FieldStorageConfig::create([
  'field_name' => 'field_brand_color',
  'entity_type' => 'node',
  'type' => 'example_color',
  'cardinality' => 1,
])->save();

// Instance on article bundle.
FieldConfig::create([
  'field_name' => 'field_brand_color',
  'entity_type' => 'node',
  'bundle' => 'article',
  'label' => 'Brand color',
  'settings' => [
    'default_color' => '#0066cc',
  ],
])->save();

// Add to form and view displays via entity_display repository or UI.
```

### Via UI

1. **Structure → Content types → Article → Manage fields → Add field**
2. Select **Color** field type
3. Configure widget on **Manage form display**, formatter on **Manage display**

---

## Validation

Validate in the field type, not the widget. The `Regex` constraint added in `propertyDefinitions()` runs during entity validation, so it covers entity forms (errors are mapped back to the widget element) as well as REST, JSON:API and programmatic saves that call `$entity->validate()`. Widgets have no automatically invoked `validateElement()` method; widget-only checks need an `#element_validate` callback.

!!! note "Field type ID"
    The `id` in `#[FieldType]` becomes the `type` value in field storage config (`example_color`).

!!! tip "Multiple values"
    Set `cardinality` > 1 on field storage to allow multiple colors per entity. The widget and formatter loops already support deltas.
