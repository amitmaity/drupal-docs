# Module Structure

A well-structured custom module separates configuration (YAML), procedural glue (`.module`), and object-oriented code (`src/`). This page lists **every file** in a complete `example_module` you can use as a template.

## Complete file tree

```
web/modules/custom/example_module/
├── example_module.info.yml
├── example_module.module
├── example_module.install
├── example_module.services.yml
├── example_module.routing.yml
├── example_module.permissions.yml
├── example_module.links.menu.yml
├── example_module.links.task.yml
├── config/
│   ├── install/
│   │   └── example_module.settings.yml
│   └── schema/
│       └── example_module.schema.yml
├── css/
│   └── example-module.admin.css
├── templates/
│   └── example-module-greeting.html.twig
├── src/
│   ├── Controller/
│   │   └── GreetingController.php
│   ├── Form/
│   │   └── SettingsForm.php
│   ├── Hook/
│   │   └── ExampleHooks.php
│   ├── Plugin/
│   │   └── Block/
│   │       └── GreetingBlock.php
│   ├── Access/
│   │   └── GreetingAccessCheck.php
│   ├── Service/
│   │   └── GreetingGenerator.php
│   └── Drush/
│       └── Commands/
│           └── ExampleCommands.php
└── tests/
    └── src/
        ├── Unit/
        │   └── GreetingGeneratorTest.php
        └── Kernel/
            └── GreetingGeneratorKernelTest.php
```

!!! note "PSR-4 autoloading"
    Drupal maps `Drupal\example_module\` → `src/` automatically. You do not need a `composer.json` inside the module unless you publish it on Packagist.

---

## `example_module.info.yml`

```yaml
name: 'Example Module'
type: module
description: 'Reference module demonstrating Drupal 11 custom module structure.'
package: Custom
core_version_requirement: ^11
dependencies:
  - drupal:system
configure: example_module.settings
```

---

## `example_module.module`

Keep this file small. Use it for legacy hooks that have no OOP equivalent yet, or for `hook_help()`.

```php
<?php

declare(strict_types=1);

/**
 * @file
 * Legacy procedural hooks for example_module.
 */

use Drupal\Core\Routing\RouteMatchInterface;

/**
 * Implements hook_help().
 */
function example_module_help(string $route_name, RouteMatchInterface $route_match): string {
  if ($route_name === 'help.page.example_module') {
    return '<p>' . t('Example module demonstrates custom module structure.') . '</p>';
  }
  return '';
}
```

---

## `example_module.install`

```php
<?php

declare(strict_types=1);

/**
 * @file
 * Install, update, and uninstall hooks for example_module.
 */

/**
 * Implements hook_install().
 */
function example_module_install(): void {
  \Drupal::configFactory()->getEditable('example_module.settings')
    ->set('default_greeting', 'Hello')
    ->save();
}

/**
 * Implements hook_uninstall().
 */
function example_module_uninstall(): void {
  \Drupal::configFactory()->getEditable('example_module.settings')->delete();
}
```

---

## `example_module.services.yml`

```yaml
services:
  example_module.greeting_generator:
    class: Drupal\example_module\Service\GreetingGenerator
    arguments:
      - '@config.factory'
      - '@string_translation'

  # Class alias so autowired classes (OOP hooks, Drush commands) can
  # type-hint GreetingGenerator.
  Drupal\example_module\Service\GreetingGenerator: '@example_module.greeting_generator'

  example_module.greeting_access_check:
    class: Drupal\example_module\Access\GreetingAccessCheck
    arguments:
      - '@current_user'
    tags:
      - { name: access_check, applies_to: _example_module_greeting_access }
```

---

## `example_module.routing.yml`

```yaml
example_module.greeting:
  path: '/example/greeting'
  defaults:
    _controller: '\Drupal\example_module\Controller\GreetingController::greeting'
    _title: 'Greeting'
  requirements:
    _example_module_greeting_access: 'TRUE'

example_module.settings:
  path: '/admin/config/system/example-module'
  defaults:
    _form: '\Drupal\example_module\Form\SettingsForm'
    _title: 'Example module settings'
  requirements:
    _permission: 'administer example module'
```

---

## `example_module.permissions.yml`

```yaml
administer example module:
  title: 'Administer example module'
  restrict access: true

view example module greeting:
  title: 'View example module greeting'
```

---

## `example_module.links.menu.yml`

```yaml
example_module.settings:
  title: 'Example module'
  description: 'Configure example module settings.'
  route_name: example_module.settings
  parent: system.admin_config_system
  weight: 100
```

---

## `example_module.links.task.yml`

```yaml
example_module.settings_tab:
  route_name: example_module.settings
  base_route: example_module.settings
  title: 'Settings'
  weight: 0
```

---

## `config/install/example_module.settings.yml`

```yaml
default_greeting: 'Hello'
```

---

## `config/schema/example_module.schema.yml`

```yaml
example_module.settings:
  type: config_object
  label: 'Example module settings'
  mapping:
    default_greeting:
      type: string
      label: 'Default greeting'
```

---

## `src/Service/GreetingGenerator.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Service;

use Drupal\Core\Config\ConfigFactoryInterface;
use Drupal\Core\StringTranslation\TranslationInterface;

/**
 * Builds greeting strings from configuration.
 */
final class GreetingGenerator {

  public function __construct(
    private readonly ConfigFactoryInterface $configFactory,
    private readonly TranslationInterface $stringTranslation,
  ) {}

  /**
   * Returns a localized greeting for the given name.
   */
  public function generate(string $name): string {
    $greeting = $this->configFactory
      ->get('example_module.settings')
      ->get('default_greeting') ?? 'Hello';

    return (string) $this->stringTranslation->translate(
      '@greeting, @name!',
      ['@greeting' => $greeting, '@name' => $name],
    );
  }

}
```

---

## `src/Controller/GreetingController.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Controller;

use Drupal\Core\Controller\ControllerBase;
use Drupal\example_module\Service\GreetingGenerator;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Returns greeting page responses.
 */
final class GreetingController extends ControllerBase {

  public function __construct(
    private readonly GreetingGenerator $greetingGenerator,
  ) {}

  /**
   * {@inheritdoc}
   */
  public static function create(ContainerInterface $container): static {
    return new static(
      $container->get('example_module.greeting_generator'),
    );
  }

  /**
   * Page callback for the greeting route.
   */
  public function greeting(): array {
    $name = $this->currentUser()->getDisplayName();

    return [
      '#theme' => 'example_module_greeting',
      '#greeting' => $this->greetingGenerator->generate($name),
      '#attached' => [
        'library' => ['example_module/admin'],
      ],
    ];
  }

}
```

---

## `src/Form/SettingsForm.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Form;

use Drupal\Core\Form\ConfigFormBase;
use Drupal\Core\Form\FormStateInterface;

/**
 * Settings form for example_module.
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
      '#default_value' => $config->get('default_greeting'),
      '#required' => TRUE,
    ];

    return parent::buildForm($form, $form_state);
  }

  /**
   * {@inheritdoc}
   */
  public function submitForm(array &$form, FormStateInterface $form_state): void {
    $this->config('example_module.settings')
      ->set('default_greeting', $form_state->getValue('default_greeting'))
      ->save();

    parent::submitForm($form, $form_state);
  }

}
```

---

## `src/Hook/ExampleHooks.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Hook;

use Drupal\Core\Hook\Attribute\Hook;
use Drupal\Core\Routing\RouteMatchInterface;
use Drupal\Core\StringTranslation\StringTranslationTrait;

/**
 * OOP hook implementations for example_module.
 */
final class ExampleHooks {

  use StringTranslationTrait;

  /**
   * Implements hook_theme().
   */
  #[Hook('theme')]
  public function theme(): array {
    return [
      'example_module_greeting' => [
        'variables' => [
          'greeting' => NULL,
        ],
      ],
    ];
  }

  /**
   * Implements hook_page_attachments().
   */
  #[Hook('page_attachments')]
  public function pageAttachments(array &$attachments): void {
    $attachments['#attached']['library'][] = 'example_module/admin';
  }

}
```

---

## `src/Plugin/Block/GreetingBlock.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Plugin\Block;

use Drupal\Core\Block\Attribute\Block;
use Drupal\Core\Block\BlockBase;
use Drupal\Core\Plugin\ContainerFactoryPluginInterface;
use Drupal\Core\StringTranslation\TranslatableMarkup;
use Drupal\example_module\Service\GreetingGenerator;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Provides a greeting block.
 */
#[Block(
  id: 'example_module_greeting_block',
  admin_label: new TranslatableMarkup('Example greeting'),
  category: new TranslatableMarkup('Custom'),
)]
final class GreetingBlock extends BlockBase implements ContainerFactoryPluginInterface {

  public function __construct(
    array $configuration,
    $plugin_id,
    $plugin_definition,
    private readonly GreetingGenerator $greetingGenerator,
  ) {
    parent::__construct($configuration, $plugin_id, $plugin_definition);
  }

  /**
   * {@inheritdoc}
   */
  public static function create(ContainerInterface $container, array $configuration, $plugin_id, $plugin_definition): static {
    return new static(
      $configuration,
      $plugin_id,
      $plugin_definition,
      $container->get('example_module.greeting_generator'),
    );
  }

  /**
   * {@inheritdoc}
   */
  public function build(): array {
    return [
      '#markup' => $this->greetingGenerator->generate($this->t('visitor')->render()),
    ];
  }

}
```

---

## `src/Access/GreetingAccessCheck.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Access;

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Routing\Access\AccessInterface;
use Drupal\Core\Session\AccountInterface;
use Symfony\Component\Routing\Route;

/**
 * Access check for the greeting route.
 */
final class GreetingAccessCheck implements AccessInterface {

  public function __construct(
    private readonly AccountInterface $currentUser,
  ) {}

  /**
   * Checks access for the greeting route.
   */
  public function access(Route $route, AccountInterface $account): AccessResult {
  // phpcs:ignore Drupal.Commenting.FunctionComment.Missing
    return AccessResult::allowedIfHasPermission($account, 'view example module greeting')
      ->cachePerPermissions();
  }

}
```

!!! warning "Access check service tag"
    The `applies_to` value in `services.yml` must match the route requirement key (`_example_module_greeting_access`). See [Access & Permissions](access-permissions.md) for details.

---

## `example_module.libraries.yml`

Create this file alongside the others (referenced by the controller and hooks):

```yaml
admin:
  css:
    theme:
      css/example-module.admin.css: {}
```

---

## `templates/example-module-greeting.html.twig`

```twig
{#
/**
 * @file
 * Theme override for the example module greeting.
 *
 * Available variables:
 * - greeting: The greeting string.
 */
#}
<div class="example-module-greeting">
  <p>{{ greeting }}</p>
</div>
```

---

## Enable and verify

```bash
ddev drush cr
ddev drush en example_module -y
ddev drush route:debug | grep example_module
open https://your-site.ddev.site/example/greeting
```

!!! tip "Next steps"
    - Add [OOP hooks](hooks.md) as your module grows
    - Replace inline markup with [Twig templates](twig-theming.md)
    - Cover services with [automated tests](testing.md)
