# Twig & Theming

Drupal renders HTML through render arrays, theme hooks, and Twig templates. Custom modules register templates, override suggestions, and preprocess variables before Twig renders them.

## Register a theme hook

### OOP — `src/Hook/ThemeHooks.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Hook;

use Drupal\Core\Hook\Attribute\Hook;

/**
 * Theme hook registrations.
 */
final class ThemeHooks {

  /**
   * Implements hook_theme().
   */
  #[Hook('theme')]
  public function theme(): array {
    return [
      'example_module_greeting' => [
        'variables' => [
          'greeting' => NULL,
          'name' => NULL,
        ],
        'template' => 'example-module-greeting',
      ],
      'example_module_card' => [
        'variables' => [
          'title' => NULL,
          'body' => NULL,
          'url' => NULL,
        ],
      ],
    ];
  }

}
```

---

## Twig template — `templates/example-module-greeting.html.twig`

```twig
{#
/**
 * @file
 * Greeting template.
 *
 * Available variables:
 * - greeting: The greeting text.
 * - name: The recipient name.
 */
#}
<div{{ attributes.addClass('example-module-greeting') }}>
  <h2>{{ greeting }}</h2>
  {% if name %}
    <p class="example-module-greeting__name">{{ 'For @name'|t({'@name': name}) }}</p>
  {% endif %}
</div>
```

Drupal maps `example_module_greeting` → `example-module-greeting.html.twig` automatically when `template` is set, or by naming convention.

---

## Render from a controller

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Controller;

use Drupal\Core\Controller\ControllerBase;

final class GreetingController extends ControllerBase {

  public function greeting(): array {
    return [
      '#theme' => 'example_module_greeting',
      '#greeting' => $this->t('Welcome back'),
      '#name' => $this->currentUser()->getDisplayName(),
      '#attached' => [
        'library' => ['example_module/greeting'],
      ],
      '#cache' => [
        'contexts' => ['user'],
        'tags' => ['user:' . $this->currentUser()->id()],
      ],
    ];
  }

}
```

---

## Preprocess hook — `src/Hook/PreprocessHooks.php`

!!! note "Drupal 11.2+"
    Modules can implement preprocess hooks as OOP `#[Hook('preprocess_HOOK')]` methods from Drupal 11.2. On older versions, use procedural `example_module_preprocess_HOOK()` functions in `example_module.module` (see [OOP & Legacy Hooks](hooks.md)). Themes keep procedural preprocess functions.

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Hook;

use Drupal\Core\Hook\Attribute\Hook;

/**
 * Preprocess hooks for example_module templates.
 */
final class PreprocessHooks {

  /**
   * Implements hook_preprocess_HOOK() for example_module_greeting.
   */
  #[Hook('preprocess_example_module_greeting')]
  public function preprocessGreeting(array &$variables): void {
    $variables['attributes']['class'][] = 'greeting--themed';

    if (!empty($variables['name'])) {
      $variables['greeting'] .= ' (' . $variables['name'] . ')';
    }
  }

  /**
   * Implements hook_preprocess_HOOK() for blocks.
   */
  #[Hook('preprocess_block')]
  public function preprocessBlock(array &$variables): void {
    if (($variables['plugin_id'] ?? '') === 'example_module_greeting_block') {
      $variables['attributes']['class'][] = 'block--example-greeting';
    }
  }

  /**
   * Implements hook_preprocess_HOOK() for nodes.
   */
  #[Hook('preprocess_node')]
  public function preprocessNode(array &$variables): void {
    /** @var \Drupal\node\NodeInterface $node */
    $node = $variables['node'];

    if ($node->bundle() === 'article') {
      $variables['attributes']['class'][] = 'node--article-custom';
    }
  }

}
```

---

## Template suggestions

Add suggestions dynamically in preprocess:

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Hook;

use Drupal\Core\Hook\Attribute\Hook;

final class SuggestionHooks {

  /**
   * Implements hook_theme_suggestions_HOOK() for example_module_greeting.
   *
   * @param array<string, mixed> $variables
   *
   * @return list<string>
   */
  #[Hook('theme_suggestions_example_module_greeting')]
  public function greetingSuggestions(array $variables): array {
    $suggestions = [];

    if (!empty($variables['name'])) {
      $suggestions[] = 'example_module_greeting__named';
    }

    return $suggestions;
  }

}
```

Create `templates/example-module-greeting--named.html.twig` for the suggestion-specific override.

---

## Override core templates from a module

Only themes pick up template overrides just by file name. A module must register the template suggestion as a theme hook with a `base hook`, then ship the template in its `templates/` directory:

```php
  /**
   * Implements hook_theme().
   */
  #[Hook('theme')]
  public function theme(): array {
    return [
      // ...hooks from the ThemeHooks example above...
      'block__example_module_greeting_block' => [
        'base hook' => 'block',
      ],
      'node__article__teaser' => [
        'base hook' => 'node',
      ],
    ];
  }
```

```
templates/
├── block--example-module-greeting-block.html.twig
└── node--article--teaser.html.twig
```

The active theme can still override these templates. To replace a core template outright (not only a suggestion), alter its `path` in `hook_theme_registry_alter()`.

```twig
{# templates/block--example-module-greeting-block.html.twig #}
<div{{ attributes.addClass('custom-block') }}>
  {{ title_prefix }}
  {% if label %}
    <h3>{{ label }}</h3>
  {% endif %}
  {{ title_suffix }}
  {% block content %}
    {{ content }}
  {% endblock %}
</div>
```

---

## Libraries — `example_module.libraries.yml`

```yaml
greeting:
  css:
    theme:
      css/greeting.css: {}
  js:
    js/greeting.js: {}
  dependencies:
    - core/drupal
    - core/once
```

## `js/greeting.js`

```javascript
(function (Drupal, once) {
  Drupal.behaviors.exampleModuleGreeting = {
    attach(context) {
      once('example-greeting', '.example-module-greeting', context).forEach((el) => {
        el.classList.add('is-attached');
      });
    },
  };
})(Drupal, once);
```

---

## Debugging template suggestions

Enable Twig debug in `sites/development.services.yml`:

```yaml
parameters:
  twig.config:
    debug: true
    auto_reload: true
    cache: false
```

HTML comments in output show available suggestions:

```html
<!-- THEME DEBUG -->
<!-- FILE NAME SUGGESTIONS:
   ▪️ example-module-greeting--named.html.twig
   ✅ example-module-greeting.html.twig
-->
```

!!! tip "Theme vs module templates"
    Put **default** templates in the module. Subthemes override them in `themes/custom/my_theme/templates/`.

!!! warning "Cache metadata in render arrays"
    Always set `#cache` keys when preprocess adds user-specific data. Missing contexts cause stale or leaked output.

!!! note "render element vs variables"
    Theme hooks using `render element` pass a single `$elements` variable. Most custom hooks use `variables` as shown above.
