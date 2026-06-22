# Custom Plugin Types

When contrib plugin types do not fit your domain, define a custom plugin type with an attribute, interface, base class, and plugin manager.

## File structure

```
example_module/
├── example_module.services.yml
└── src/
    ├── Attribute/
    │   └── Processor.php
    ├── ProcessorPluginManager.php
    ├── ProcessorPluginInterface.php
    ├── ProcessorPluginBase.php
    ├── Annotation/
    │   └── Processor.php          # Optional legacy annotation
    └── Plugin/Processor/
        └── UppercaseProcessor.php
```

---

## Plugin attribute — `src/Attribute/Processor.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Attribute;

use Drupal\Component\Plugin\Attribute\Plugin;
use Drupal\Core\StringTranslation\TranslatableMarkup;

/**
 * Defines a Processor plugin attribute.
 */
#[\Attribute(\Attribute::TARGET_CLASS)]
final class Processor extends Plugin {

  public function __construct(
    public readonly string $id,
    public readonly TranslatableMarkup $label,
    public readonly int $weight = 0,
  ) {}

}
```

---

## Plugin interface — `src/ProcessorPluginInterface.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module;

use Drupal\Component\Plugin\PluginInspectionInterface;

/**
 * Interface for data processor plugins.
 */
interface ProcessorPluginInterface extends PluginInspectionInterface {

  /**
   * Processes the input string and returns the result.
   */
  public function process(string $input): string;

}
```

---

## Base class — `src/ProcessorPluginBase.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module;

use Drupal\Core\Plugin\PluginBase;

/**
 * Base class for processor plugins.
 */
abstract class ProcessorPluginBase extends PluginBase implements ProcessorPluginInterface {

}
```

---

## Plugin manager — `src/ProcessorPluginManager.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module;

use Drupal\Core\Cache\CacheBackendInterface;
use Drupal\Core\Extension\ModuleHandlerInterface;
use Drupal\Core\Plugin\DefaultPluginManager;
use Drupal\example_module\Attribute\Processor;

/**
 * Manages Processor plugins.
 */
final class ProcessorPluginManager extends DefaultPluginManager {

  public function __construct(
    \Traversable $namespaces,
    CacheBackendInterface $cache_backend,
    ModuleHandlerInterface $module_handler,
  ) {
    parent::__construct(
      'Plugin/Processor',
      $namespaces,
      $module_handler,
      ProcessorPluginInterface::class,
      Processor::class,
    );

    $this->alterInfo('example_module_processor_info');
    $this->setCacheBackend($cache_backend, 'example_module_processors');
  }

}
```

---

## Service definition — `example_module.services.yml`

```yaml
services:
  plugin.manager.example_module.processor:
    class: Drupal\example_module\ProcessorPluginManager
    arguments:
      - '@container.namespaces'
      - '@cache.discovery'
      - '@module_handler'
```

---

## Plugin implementation — `src/Plugin/Processor/UppercaseProcessor.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Plugin\Processor;

use Drupal\Core\StringTranslation\TranslatableMarkup;
use Drupal\example_module\Attribute\Processor;
use Drupal\example_module\ProcessorPluginBase;

/**
 * Converts input to uppercase.
 */
#[Processor(
  id: 'uppercase',
  label: new TranslatableMarkup('Uppercase'),
  weight: 0,
)]
final class UppercaseProcessor extends ProcessorPluginBase {

  /**
   * {@inheritdoc}
   */
  public function process(string $input): string {
    return mb_strtoupper($input);
  }

}
```

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Plugin\Processor;

use Drupal\Core\StringTranslation\TranslatableMarkup;
use Drupal\example_module\Attribute\Processor;
use Drupal\example_module\ProcessorPluginBase;

/**
 * Reverses the input string.
 */
#[Processor(
  id: 'reverse',
  label: new TranslatableMarkup('Reverse'),
  weight: 10,
)]
final class ReverseProcessor extends ProcessorPluginBase {

  /**
   * {@inheritdoc}
   */
  public function process(string $input): string {
    return strrev($input);
  }

}
```

---

## Consumer service — `src/Service/ProcessorPipeline.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Service;

use Drupal\example_module\ProcessorPluginManager;

/**
 * Runs all processor plugins in weight order.
 */
final class ProcessorPipeline {

  public function __construct(
    private readonly ProcessorPluginManager $processorManager,
  ) {}

  public function run(string $input): string {
    $definitions = $this->processorManager->getDefinitions();
    uasort($definitions, static fn(array $a, array $b): int => ($a['weight'] ?? 0) <=> ($b['weight'] ?? 0));

    $result = $input;
    foreach (array_keys($definitions) as $plugin_id) {
      $plugin = $this->processorManager->createInstance($plugin_id);
      $result = $plugin->process($result);
    }

    return $result;
  }

}
```

```yaml
# example_module.services.yml
  example_module.processor_pipeline:
    class: Drupal\example_module\Service\ProcessorPipeline
    arguments:
      - '@plugin.manager.example_module.processor'
```

---

## Alter plugin definitions

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Hook;

use Drupal\Core\Hook\Attribute\Hook;

/**
 * Alters processor plugin definitions.
 */
final class ProcessorHooks {

  /**
   * Implements hook_example_module_processor_info_alter().
   */
  #[Hook('example_module_processor_info_alter')]
  public function processorInfoAlter(array &$definitions): void {
    if (isset($definitions['reverse'])) {
      $definitions['reverse']['weight'] = 100;
    }
  }

}
```

---

## Usage

```php
$pipeline = \Drupal::service('example_module.processor_pipeline');
echo $pipeline->run('hello'); // "OLLEH" (uppercase then reverse)
```

!!! note "Discovery subdirectory"
    The second argument to `DefaultPluginManager` (`Plugin/Processor`) is the PSR-4 subdirectory under `src/` where plugins live.

!!! tip "Reference implementation"
    Read core's `BlockManager` or `ActionManager` for production patterns including contextual plugins and derivatives.
