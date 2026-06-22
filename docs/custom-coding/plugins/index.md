# Plugin System

Plugins are discoverable, swappable components managed by **plugin managers**. Drupal core provides dozens of plugin types; custom modules typically create blocks, field formatters, queue workers, and occasionally entire plugin types.

## Chapters

| Guide | Plugin type | Annotation / attribute |
|-------|-------------|------------------------|
| [Block Plugins](block-plugins.md) | `block` | `#[Block]` |
| [Queue Workers](queue-workers.md) | `queue_worker` | `#[QueueWorker]` |
| [Field Formatters](field-formatters.md) | `field_formatter` | `#[FieldFormatter]` |
| [Custom Plugin Types](custom-plugin-types.md) | Your own | Custom attribute |

## Anatomy of a plugin

```
src/Plugin/Block/ExampleBlock.php   ← Plugin class
```

Discovery uses PHP 8 attributes (Drupal 10.2+). Legacy `@Annotation` docblocks still work but attributes are preferred.

### Minimal plugin

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Plugin\Block;

use Drupal\Core\Block\Attribute\Block;
use Drupal\Core\Block\BlockBase;
use Drupal\Core\StringTranslation\TranslatableMarkup;

#[Block(
  id: 'example_hello',
  admin_label: new TranslatableMarkup('Hello block'),
  category: new TranslatableMarkup('Custom'),
)]
final class HelloBlock extends BlockBase {

  /**
   * {@inheritdoc}
   */
  public function build(): array {
    return [
      '#markup' => $this->t('Hello from a block plugin.'),
    ];
  }

}
```

## Using the plugin manager

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Service;

use Drupal\Core\Block\BlockManagerInterface;
use Drupal\Core\Plugin\Context\ContextHandlerInterface;

final class BlockRenderer {

  public function __construct(
    private readonly BlockManagerInterface $blockManager,
    private readonly ContextHandlerInterface $contextHandler,
  ) {}

  /**
   * Renders a block plugin by ID.
   *
   * @return array<string, mixed>
   */
  public function render(string $plugin_id): array {
    $plugin = $this->blockManager->createInstance($plugin_id);
    return $plugin->build();
  }

}
```

## Plugin base classes and interfaces

| Plugin type | Base class / interface | Manager service |
|-------------|------------------------|-----------------|
| Block | `BlockBase` | `plugin.manager.block` |
| Field formatter | `FormatterBase` | `plugin.manager.field.formatter` |
| Field widget | `WidgetBase` | `plugin.manager.field.widget` |
| Queue worker | `QueueWorkerBase` | `plugin.manager.queue_worker` |
| Action | `ActionBase` | `plugin.manager.action` |
| Condition | `ConditionPluginBase` | `plugin.manager.condition` |

## Configuration in plugins

Plugins that need settings implement `PluginFormInterface` or use `configurable` patterns:

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Plugin\Block;

use Drupal\Core\Block\Attribute\Block;
use Drupal\Core\Block\BlockBase;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\StringTranslation\TranslatableMarkup;

#[Block(
  id: 'example_configurable',
  admin_label: new TranslatableMarkup('Configurable example'),
)]
final class ConfigurableBlock extends BlockBase {

  /**
   * {@inheritdoc}
   */
  public function defaultConfiguration(): array {
    return ['label_text' => 'Default'] + parent::defaultConfiguration();
  }

  /**
   * {@inheritdoc}
   */
  public function blockForm(array $form, FormStateInterface $form_state): array {
    $form['label_text'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Label text'),
      '#default_value' => $this->configuration['label_text'],
    ];
    return $form;
  }

  /**
   * {@inheritdoc}
   */
  public function blockSubmit(array $form, FormStateInterface $form_state): void {
    $this->configuration['label_text'] = $form_state->getValue('label_text');
  }

  /**
   * {@inheritdoc}
   */
  public function build(): array {
    return ['#markup' => $this->configuration['label_text']];
  }

}
```

!!! note "Cache metadata"
    Plugins that render output must set cache keys, contexts, and tags in their render arrays. See [Caching](../../caching/index.md).

!!! tip "Clear plugin cache"
    ```bash
    ddev drush cr
  ```
