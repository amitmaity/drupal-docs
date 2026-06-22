# Config Form

Configuration forms persist settings to the active config storage using `ConfigFormBase`. They require a config schema and are restricted to users with appropriate permissions.

## Files required

```
example_module/
├── example_module.routing.yml
├── example_module.permissions.yml
├── config/
│   ├── install/example_module.settings.yml
│   └── schema/example_module.schema.yml
└── src/Form/SettingsForm.php
```

---

## Default config — `config/install/example_module.settings.yml`

```yaml
default_greeting: 'Hello'
enable_logging: true
max_items: 10
```

---

## Config schema — `config/schema/example_module.schema.yml`

```yaml
example_module.settings:
  type: config_object
  label: 'Example module settings'
  mapping:
    default_greeting:
      type: string
      label: 'Default greeting'
    enable_logging:
      type: boolean
      label: 'Enable logging'
    max_items:
      type: integer
      label: 'Maximum items'
```

---

## Permissions — `example_module.permissions.yml`

```yaml
administer example module:
  title: 'Administer example module'
  restrict access: true
```

---

## Route — `example_module.routing.yml`

```yaml
example_module.settings:
  path: '/admin/config/system/example-module'
  defaults:
    _form: '\Drupal\example_module\Form\SettingsForm'
    _title: 'Example module settings'
  requirements:
    _permission: 'administer example module'
```

---

## Menu link — `example_module.links.menu.yml`

```yaml
example_module.settings:
  title: 'Example module'
  description: 'Configure example module.'
  route_name: example_module.settings
  parent: system.admin_config_system
  weight: 50
```

---

## Form class — `src/Form/SettingsForm.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Form;

use Drupal\Core\Form\ConfigFormBase;
use Drupal\Core\Form\FormStateInterface;

/**
 * Configuration form for example_module settings.
 */
final class SettingsForm extends ConfigFormBase {

  /**
   * {@inheritdoc}
   */
  protected function getEditableConfigNames(): array {
    return ['example_module.settings'];
  }

  /**
   * {@inheritdoc}
   */
  public function getFormId(): string {
    return 'example_module_settings';
  }

  /**
   * {@inheritdoc}
   */
  public function buildForm(array $form, FormStateInterface $form_state): array {
    $config = $this->config('example_module.settings');

    $form['default_greeting'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Default greeting'),
      '#description' => $this->t('Greeting prefix shown to visitors.'),
      '#default_value' => $config->get('default_greeting'),
      '#required' => TRUE,
      '#maxlength' => 64,
    ];

    $form['enable_logging'] = [
      '#type' => 'checkbox',
      '#title' => $this->t('Enable logging'),
      '#default_value' => $config->get('enable_logging'),
    ];

    $form['max_items'] = [
      '#type' => 'number',
      '#title' => $this->t('Maximum items'),
      '#default_value' => $config->get('max_items'),
      '#min' => 1,
      '#max' => 1000,
      '#required' => TRUE,
    ];

    return parent::buildForm($form, $form_state);
  }

  /**
   * {@inheritdoc}
   */
  public function validateForm(array &$form, FormStateInterface $form_state): void {
    parent::validateForm($form, $form_state);

    $greeting = trim((string) $form_state->getValue('default_greeting'));
    if ($greeting === '') {
      $form_state->setErrorByName('default_greeting', $this->t('Greeting cannot be empty.'));
    }
  }

  /**
   * {@inheritdoc}
   */
  public function submitForm(array &$form, FormStateInterface $form_state): void {
    $this->config('example_module.settings')
      ->set('default_greeting', trim((string) $form_state->getValue('default_greeting')))
      ->set('enable_logging', (bool) $form_state->getValue('enable_logging'))
      ->set('max_items', (int) $form_state->getValue('max_items'))
      ->save();

    parent::submitForm($form, $form_state);
  }

}
```

---

## Reading config in a service

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Service;

use Drupal\Core\Config\ConfigFactoryInterface;

final class SettingsReader {

  public function __construct(
    private readonly ConfigFactoryInterface $configFactory,
  ) {}

  public function isLoggingEnabled(): bool {
    return (bool) $this->configFactory
      ->get('example_module.settings')
      ->get('enable_logging');
  }

  public function getMaxItems(): int {
    return (int) $this->configFactory
      ->get('example_module.settings')
      ->get('max_items');
  }

}
```

---

## Export and deploy config

```bash
# Export single config object
ddev drush config:export --destination=/tmp/cfg
cat /tmp/cfg/example_module.settings.yml

# Import on another environment
ddev drush config:import -y
```

!!! note "Config overrides"
    Settings in `settings.php` via `$config['example_module.settings']['max_items'] = 5;` override the database but are not exported. Use for environment-specific values only.

!!! warning "Restrict access"
    Always use `restrict access: true` on admin permissions and never expose config forms to anonymous users.
