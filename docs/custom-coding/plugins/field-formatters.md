# Field Formatter Plugins

Field formatters control how field values appear on entity displays (teasers, full view modes, etc.).

## Formatter plugin — `src/Plugin/Field/FieldFormatter/PriceFormatter.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Plugin\Field\FieldFormatter;

use Drupal\Core\Field\Attribute\FieldFormatter;
use Drupal\Core\Field\FieldDefinitionInterface;
use Drupal\Core\Field\FieldItemListInterface;
use Drupal\Core\Field\FormatterBase;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\StringTranslation\TranslatableMarkup;

/**
 * Formats decimal price fields with currency symbol.
 */
#[FieldFormatter(
  id: 'example_module_price',
  label: new TranslatableMarkup('Price with currency'),
  field_types: ['decimal', 'float'],
)]
final class PriceFormatter extends FormatterBase {

  /**
   * {@inheritdoc}
   */
  public static function defaultSettings(): array {
    return [
      'currency_symbol' => '$',
      'decimals' => 2,
    ] + parent::defaultSettings();
  }

  /**
   * {@inheritdoc}
   */
  public function settingsForm(array $form, FormStateInterface $form_state): array {
    $form = parent::settingsForm($form, $form_state);

    $form['currency_symbol'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Currency symbol'),
      '#default_value' => $this->getSetting('currency_symbol'),
      '#size' => 4,
    ];

    $form['decimals'] = [
      '#type' => 'number',
      '#title' => $this->t('Decimal places'),
      '#default_value' => $this->getSetting('decimals'),
      '#min' => 0,
      '#max' => 4,
    ];

    return $form;
  }

  /**
   * {@inheritdoc}
   */
  public function settingsSummary(): array {
    return [
      $this->t('Symbol: @symbol, decimals: @dec', [
        '@symbol' => $this->getSetting('currency_symbol'),
        '@dec' => $this->getSetting('decimals'),
      ]),
    ];
  }

  /**
   * {@inheritdoc}
   */
  public function viewElements(FieldItemListInterface $items, $langcode): array {
    $symbol = (string) $this->getSetting('currency_symbol');
    $decimals = (int) $this->getSetting('decimals');
    $elements = [];

    foreach ($items as $delta => $item) {
      $value = (float) $item->value;
      $formatted = $symbol . number_format($value, $decimals);

      $elements[$delta] = [
        '#type' => 'inline_template',
        '#template' => '<span class="price">{{ price }}</span>',
        '#context' => ['price' => $formatted],
      ];
    }

    return $elements;
  }

}
```

---

## Enable on a display

1. **Structure → Content types → Article → Manage display**
2. Find the price field → select **Price with currency**
3. Click the gear icon to configure symbol and decimals

Or via config export (`core.entity_view_display.node.article.default.yml`):

```yaml
content:
  field_price:
    type: example_module_price
    label: above
    settings:
      currency_symbol: '€'
      decimals: 2
    third_party_settings: {}
    weight: 5
    region: content
```

---

## Formatter with entity access to parent

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Plugin\Field\FieldFormatter;

use Drupal\Core\Field\Attribute\FieldFormatter;
use Drupal\Core\Field\FieldItemListInterface;
use Drupal\Core\Field\FormatterBase;
use Drupal\Core\StringTranslation\TranslatableMarkup;

#[FieldFormatter(
  id: 'example_module_author_link',
  label: new TranslatableMarkup('Author link'),
  field_types: ['entity_reference'],
)]
final class AuthorLinkFormatter extends FormatterBase {

  /**
   * {@inheritdoc}
   */
  public function viewElements(FieldItemListInterface $items, $langcode): array {
    $elements = [];
    $entity = $items->getEntity();

    foreach ($items as $delta => $item) {
      $referenced = $item->entity;
      if ($referenced === NULL) {
        continue;
      }

      $elements[$delta] = [
        '#type' => 'link',
        '#title' => $referenced->label(),
        '#url' => $referenced->toUrl(),
        '#cache' => [
          'tags' => $referenced->getCacheTags(),
        ],
      ];
    }

    if ($entity !== NULL) {
      $elements['#cache']['tags'] = array_merge(
        $elements['#cache']['tags'] ?? [],
        $entity->getCacheTags(),
      );
    }

    return $elements;
  }

}
```

!!! tip "Pair with custom field types"
    See [Custom Field Types](../custom-field-types.md) for a complete field type + widget + formatter example.

!!! note "Multiple field types"
    List all compatible types in `field_types:`. The formatter appears only for those field types in the manage display UI.
