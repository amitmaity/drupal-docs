# OOP & Legacy Hooks

Drupal 11 supports two hook implementation styles:

1. **OOP hooks (recommended)** — methods on classes in `src/Hook/` annotated with `#[Hook]`
2. **Legacy procedural hooks** — functions named `example_module_hook_name()` in `.module` or `.install`

!!! note "Drupal 11.1+ OOP hooks"
    OOP hooks are stable in Drupal 11.1+. New custom code should prefer `#[Hook]` classes. Procedural hooks remain fully supported for backward compatibility and hooks without OOP coverage yet.

---

## OOP hook class

Place hook implementations in `src/Hook/`. Drupal discovers them automatically — no service definition required.

### `src/Hook/ExampleHooks.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Hook;

use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\Hook\Attribute\Hook;
use Drupal\Core\Routing\RouteMatchInterface;
use Drupal\Core\StringTranslation\StringTranslationTrait;

/**
 * OOP hook implementations for example_module.
 */
final class ExampleHooks {

  use StringTranslationTrait;

  /**
   * Implements hook_help().
   */
  #[Hook('help')]
  public function help(string $route_name, RouteMatchInterface $route_match): string {
    if ($route_name === 'help.page.example_module') {
      return '<p>' . $this->t('Example module reference documentation.') . '</p>';
    }
    return '';
  }

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
   * Implements hook_form_alter().
   */
  #[Hook('form_alter')]
  public function formAlter(array &$form, FormStateInterface $form_state, string $form_id): void {
    if ($form_id === 'system_site_information_settings') {
      $form['example_module_note'] = [
        '#markup' => '<p>' . $this->t('Altered by example_module.') . '</p>',
        '#weight' => 100,
      ];
    }
  }

  /**
   * Implements hook_entity_presave().
   */
  #[Hook('entity_presave')]
  public function entityPresave(EntityInterface $entity): void {
    if ($entity->getEntityTypeId() === 'node' && $entity->isNew()) {
      $entity->set('title', trim((string) $entity->label()));
    }
  }

  /**
   * Implements hook_cron().
   */
  #[Hook('cron')]
  public function cron(): void {
    \Drupal::logger('example_module')->info('Cron executed at @time', [
      '@time' => date('c'),
    ]);
  }

}
```

### Multiple hook classes

You may split hooks across multiple classes for organization:

```
src/Hook/
├── ExampleFormHooks.php    # form_* hooks
├── ExampleEntityHooks.php  # entity_* hooks
└── ExampleThemeHooks.php   # theme, preprocess hooks
```

Each class follows the same pattern: `final class`, `#[Hook('hook_name')]`, typed parameters matching the procedural signature.

---

## Legacy procedural hooks

### `example_module.module`

```php
<?php

declare(strict_types=1);

/**
 * @file
 * Legacy procedural hooks for example_module.
 */

use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\Routing\RouteMatchInterface;

/**
 * Implements hook_help().
 */
function example_module_help(string $route_name, RouteMatchInterface $route_match): string {
  if ($route_name === 'help.page.example_module') {
    return '<p>' . t('Legacy help implementation.') . '</p>';
  }
  return '';
}

/**
 * Implements hook_theme().
 */
function example_module_theme(): array {
  return [
    'example_module_card' => [
      'variables' => [
        'title' => NULL,
        'body' => NULL,
      ],
    ],
  ];
}

/**
 * Implements hook_form_alter().
 */
function example_module_form_alter(array &$form, FormStateInterface $form_state, string $form_id): void {
  if ($form_id === 'node_article_form') {
    $form['#attributes']['class'][] = 'example-module-article-form';
  }
}

/**
 * Implements hook_preprocess_HOOK() for blocks.
 */
function example_module_preprocess_block(array &$variables): void {
  if (($variables['plugin_id'] ?? '') === 'example_module_greeting_block') {
    $variables['attributes']['class'][] = 'example-module-greeting-block';
  }
}
```

### `example_module.install`

```php
<?php

declare(strict_types=1);

/**
 * @file
 * Install and update hooks for example_module.
 */

/**
 * Implements hook_schema().
 */
function example_module_schema(): array {
  $schema['example_module_log'] = [
    'description' => 'Stores example log entries.',
    'fields' => [
      'id' => [
        'type' => 'serial',
        'unsigned' => TRUE,
        'not null' => TRUE,
      ],
      'message' => [
        'type' => 'varchar',
        'length' => 255,
        'not null' => TRUE,
      ],
      'created' => [
        'type' => 'int',
        'not null' => TRUE,
        'default' => 0,
      ],
    ],
    'primary key' => ['id'],
  ];

  return $schema;
}

/**
 * Implements hook_install().
 */
function example_module_install(): void {
  // Create default config, seed data, etc.
}

/**
 * Implements hook_uninstall().
 */
function example_module_uninstall(): void {
  \Drupal::database()->schema()->dropTable('example_module_log');
}
```

---

## Comparison: OOP vs procedural

| Aspect | OOP `#[Hook]` | Procedural |
|--------|---------------|------------|
| Discovery | Auto-scanned in `src/Hook/` | Function name convention |
| Dependency injection | Constructor injection via autowired services | `\Drupal::service()` |
| Testing | Instantiate class, call method | `module_load_include()` or kernel tests |
| IDE support | Full type hints on parameters | Docblock `@param` required |
| Recommendation | **Use for new code** | Use when OOP hook missing or for `.install` |

---

## Injecting services into OOP hooks

Hook classes support constructor injection when the class is registered as a service. For most hooks, Drupal auto-wires dependencies:

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Hook;

use Drupal\Core\Config\ConfigFactoryInterface;
use Drupal\Core\Hook\Attribute\Hook;
use Drupal\example_module\Service\GreetingGenerator;

/**
 * Hooks that depend on services.
 */
final class ExampleServiceHooks {

  public function __construct(
    private readonly GreetingGenerator $greetingGenerator,
    private readonly ConfigFactoryInterface $configFactory,
  ) {}

  /**
   * Implements hook_cron().
   */
  #[Hook('cron')]
  public function cron(): void {
    $greeting = $this->greetingGenerator->generate('cron');
    \Drupal::logger('example_module')->notice($greeting);
  }

}
```

If autowiring fails, register the hook class explicitly:

```yaml
# example_module.services.yml
services:
  Drupal\example_module\Hook\ExampleServiceHooks:
    arguments:
      - '@example_module.greeting_generator'
      - '@config.factory'
```

---

## Invoking hooks from your code

Use the module handler service — never call procedural functions directly:

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Service;

use Drupal\Core\Extension\ModuleHandlerInterface;

final class Notifier {

  public function __construct(
    private readonly ModuleHandlerInterface $moduleHandler,
  ) {}

  /**
   * Notifies other modules of an event.
   *
   * @param array<string, mixed> $context
   */
  public function notify(string $message, array $context = []): void {
    $this->moduleHandler->invokeAll('example_module_notify', [$message, $context]);
  }

}
```

Other modules implement `hook_example_module_notify()` or `#[Hook('example_module_notify')]` to receive the event.

!!! warning "Do not mix duplicate implementations"
    Implement each hook in **either** OOP or procedural form, not both. If both exist, behavior is undefined.

!!! tip "Find hook definitions"
    ```bash
    ddev drush php:eval "print_r(array_keys(\Drupal::service('module_handler')->getImplementations('cron')));"
    ```
