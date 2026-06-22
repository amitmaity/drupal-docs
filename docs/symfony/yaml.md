# Yaml

The **Yaml** component parses and dumps YAML files. Nearly every Drupal configuration file — routing, services, module info, and exported config — is YAML parsed by Symfony's `Yaml` class.

## Core concepts

```php
<?php

declare(strict_types=1);

use Symfony\Component\Yaml\Yaml;

$data = Yaml::parseFile('/path/to/file.yml');
$yaml = Yaml::dump($data, 2, 2, Yaml::DUMP_MULTI_LINE_LITERAL_BLOCK);
```

| Method | Use case |
|--------|----------|
| `parse()` | Parse YAML string |
| `parseFile()` | Parse file from disk |
| `dump()` | Convert PHP array to YAML string |

## How Drupal uses Yaml

### Config export/import

Drupal stores active configuration as YAML in the database and exports to `config/sync/*.yml`:

```yaml
# config/sync/system.site.yml
_core:
  default_config_hash: abc123
langcode: en
name: 'My Headless Site'
mail: admin@example.com
slogan: ''
page:
  403: ''
  404: ''
  front: /node
admin_compact_mode: false
weight_select_max: 100
default_langcode: en
mail_notification: null
```

### Routing and services

Every module ships `*.routing.yml` and `*.services.yml` — compiled into PHP cache on container build.

### Module info

```yaml
# my_module.info.yml
name: 'My Module'
type: module
description: 'Demonstrates Symfony Yaml usage.'
package: Custom
core_version_requirement: ^11
dependencies:
  - drupal:node
```

!!! warning "Never hand-edit exported config blindly"
    Config entities have UUIDs and hash dependencies. Use `drush config:export` and `config:import` for sync workflows.

## Parsing YAML in custom code

=== "Read a data file"

    ```php
    <?php

    declare(strict_types=1);

    namespace Drupal\my_module\Service;

    use Drupal\Core\Extension\ModuleExtensionList;
    use Symfony\Component\Yaml\Yaml;

    final class FixtureLoader {

      public function __construct(
        private readonly ModuleExtensionList $moduleList,
      ) {}

      public function loadProducts(): array {
        $path = $this->moduleList->getPath('my_module') . '/data/products.yml';
        $data = Yaml::parseFile($path);

        return $data['products'] ?? [];
      }

    }
    ```

=== "products.yml"

    ```yaml
    products:
      - sku: WIDGET-001
        title: Premium Widget
        price: 29.99
      - sku: GADGET-002
        title: Basic Gadget
        price: 9.99
    ```

## Dumping YAML

Generate YAML for download or migration:

```php
<?php

declare(strict_types=1);

use Symfony\Component\Yaml\Yaml;

$export = [
  'version' => 1,
  'products' => [
    ['sku' => 'A1', 'title' => 'Item A'],
    ['sku' => 'B2', 'title' => 'Item B'],
  ],
];

$yaml = Yaml::dump($export, 4, 2, Yaml::DUMP_MULTI_LINE_LITERAL_BLOCK);
file_put_contents('public://exports/products.yml', $yaml);
```

## Tabbed: safe parsing options

=== "Strict parsing"

    ```php
    <?php

    declare(strict_types=1);

    use Symfony\Component\Yaml\Yaml;

    try {
      $data = Yaml::parseFile($path, Yaml::PARSE_EXCEPTION_ON_INVALID_TYPE);
    }
    catch (\Symfony\Component\Yaml\Exception\ParseException $e) {
      throw new \RuntimeException("Invalid YAML in {$path}: " . $e->getMessage(), 0, $e);
    }
    ```

=== "Inline for tests"

    ```php
    <?php

    declare(strict_types=1);

    use Symfony\Component\Yaml\Yaml;

    $routing = Yaml::parse(<<<YAML
    my_module.test:
      path: '/test'
      defaults:
        _controller: '\\Drupal\\my_module\\Controller\\TestController::test'
      requirements:
        _access: 'TRUE'
    YAML);
    ```

## Custom module example: config generator command

Generates default config YAML from a template:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Service;

use Drupal\Core\Config\ConfigFactoryInterface;
use Drupal\Core\Config\FileStorage;
use Symfony\Component\Yaml\Yaml;

final class ConfigExporter {

  public function __construct(
    private readonly ConfigFactoryInterface $configFactory,
  ) {}

  /**
   * Export my_module settings as YAML string for version control.
   */
  public function exportSettings(): string {
    $config = $this->configFactory->get('my_module.settings')->getRawData();

    return Yaml::dump(
      ['my_module.settings' => $config],
      4,
      2,
      Yaml::DUMP_MULTI_LINE_LITERAL_BLOCK,
    );
  }

  /**
   * Import settings from a YAML file in the module's config/install directory.
   */
  public function importFromModule(string $modulePath): void {
    $storage = new FileStorage($modulePath . '/config/install');
    $data = $storage->read('my_module.settings');

    if ($data !== FALSE) {
      $this->configFactory->getEditable('my_module.settings')
        ->setData($data)
        ->save();
    }
  }

}
```

```yaml
# config/install/my_module.settings.yml
api_endpoint: 'https://api.example.com/v1'
sync_enabled: true
batch_size: 50
```

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Form;

use Drupal\Core\Form\ConfigFormBase;
use Drupal\Core\Form\FormStateInterface;

final class SettingsForm extends ConfigFormBase {

  protected function getEditableConfigNames(): array {
    return ['my_module.settings'];
  }

  public function getFormId(): string {
    return 'my_module_settings';
  }

  public function buildForm(array $form, FormStateInterface $form_state): array {
    $config = $this->config('my_module.settings');

    $form['api_endpoint'] = [
      '#type' => 'url',
      '#title' => $this->t('API endpoint'),
      '#default_value' => $config->get('api_endpoint'),
    ];

    $form['batch_size'] = [
      '#type' => 'number',
      '#title' => $this->t('Batch size'),
      '#default_value' => $config->get('batch_size'),
      '#min' => 1,
      '#max' => 500,
    ];

    return parent::buildForm($form, $form_state);
  }

  public function submitForm(array &$form, FormStateInterface $form_state): void {
    $this->config('my_module.settings')
      ->set('api_endpoint', $form_state->getValue('api_endpoint'))
      ->set('batch_size', $form_state->getValue('batch_size'))
      ->save();

    parent::submitForm($form, $form_state);
  }

}
```

## Related reading

- [Dependency Injection: Service Definitions](../dependency-injection/service-definitions.md)
- [Getting Started: Composer](../getting-started/composer.md)
