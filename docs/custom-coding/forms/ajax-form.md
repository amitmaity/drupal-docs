# AJAX Form

AJAX forms replace part of the page without a full reload using `#ajax` properties on form elements. Drupal returns `AjaxResponse` objects with `ReplaceCommand`, `AppendCommand`, and other commands.

## Route — `example_module.routing.yml`

```yaml
example_module.ajax_demo:
  path: '/example/ajax-demo'
  defaults:
    _form: '\Drupal\example_module\Form\AjaxDemoForm'
    _title: 'AJAX demo'
  requirements:
    _permission: 'access content'
```

---

## AJAX form — `src/Form/AjaxDemoForm.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Form;

use Drupal\Core\Ajax\AjaxResponse;
use Drupal\Core\Ajax\HtmlCommand;
use Drupal\Core\Ajax\InvokeCommand;
use Drupal\Core\Ajax\ReplaceCommand;
use Drupal\Core\Form\FormBase;
use Drupal\Core\Form\FormStateInterface;

/**
 * Demonstrates AJAX form interactions.
 */
final class AjaxDemoForm extends FormBase {

  /**
   * {@inheritdoc}
   */
  public function getFormId(): string {
    return 'example_module_ajax_demo';
  }

  /**
   * {@inheritdoc}
   */
  public function buildForm(array $form, FormStateInterface $form_state): array {
    $form['#prefix'] = '<div id="ajax-demo-wrapper">';
    $form['#suffix'] = '</div>';

    $form['country'] = [
      '#type' => 'select',
      '#title' => $this->t('Country'),
      '#options' => [
        'us' => $this->t('United States'),
        'ca' => $this->t('Canada'),
        'uk' => $this->t('United Kingdom'),
      ],
      '#empty_option' => $this->t('- Select country -'),
      '#ajax' => [
        'callback' => '::updateCityOptions',
        'wrapper' => 'city-wrapper',
        'event' => 'change',
      ],
    ];

    $selected_country = $form_state->getValue('country', '');
    $form['city_wrapper'] = [
      '#type' => 'container',
      '#attributes' => ['id' => 'city-wrapper'],
    ];

    $form['city_wrapper']['city'] = [
      '#type' => 'select',
      '#title' => $this->t('City'),
      '#options' => $this->getCitiesForCountry((string) $selected_country),
      '#empty_option' => $this->t('- Select city -'),
      '#validated' => TRUE,
    ];

    $form['name'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Your name'),
    ];

    $form['actions']['preview'] = [
      '#type' => 'button',
      '#value' => $this->t('Preview greeting'),
      '#ajax' => [
        'callback' => '::previewGreeting',
        'wrapper' => 'greeting-preview',
      ],
    ];

    $form['greeting_preview'] = [
      '#type' => 'container',
      '#attributes' => ['id' => 'greeting-preview'],
    ];

    $form['greeting_preview']['output'] = [
      '#markup' => '<p class="greeting-placeholder">' . $this->t('Click preview to see your greeting.') . '</p>',
    ];

    $form['actions']['submit'] = [
      '#type' => 'submit',
      '#value' => $this->t('Save'),
    ];

    return $form;
  }

  /**
   * AJAX callback: rebuild city options when country changes.
   */
  public function updateCityOptions(array &$form, FormStateInterface $form_state): array {
    return $form['city_wrapper'];
  }

  /**
   * AJAX callback: show a live greeting preview.
   */
  public function previewGreeting(array &$form, FormStateInterface $form_state): AjaxResponse {
    $response = new AjaxResponse();

    $name = trim((string) $form_state->getValue('name'));
    $country = (string) $form_state->getValue('country');
    $city = (string) $form_state->getValue('city');

    if ($name === '') {
      $message = (string) $this->t('Please enter your name.');
      $response->addCommand(new ReplaceCommand(
        '#greeting-preview',
        '<div id="greeting-preview"><p class="messages messages--warning">' . $message . '</p></div>',
      ));
      return $response;
    }

    $greeting = (string) $this->t('Hello @name from @city, @country!', [
      '@name' => $name,
      '@city' => $city ?: $this->t('unknown city'),
      '@country' => $country ?: $this->t('unknown country'),
    ]);

    $response->addCommand(new HtmlCommand('#greeting-preview', '<p>' . $greeting . '</p>'));
    $response->addCommand(new InvokeCommand('#greeting-preview', 'addClass', ['greeting-ready']));

    return $response;
  }

  /**
   * {@inheritdoc}
   */
  public function submitForm(array &$form, FormStateInterface $form_state): void {
    $this->messenger()->addStatus($this->t('Saved selection for @name.', [
      '@name' => $form_state->getValue('name'),
    ]));
  }

  /**
   * Returns city options for a country code.
   *
   * @return array<string, string>
   */
  private function getCitiesForCountry(string $country): array {
    $cities = [
      'us' => ['nyc' => 'New York', 'la' => 'Los Angeles', 'chi' => 'Chicago'],
      'ca' => ['tor' => 'Toronto', 'van' => 'Vancouver', 'mtl' => 'Montreal'],
      'uk' => ['lon' => 'London', 'man' => 'Manchester', 'edi' => 'Edinburgh'],
    ];

    return $cities[$country] ?? [];
  }

}
```

---

## Common AJAX commands

```php
<?php

declare(strict_types=1);

use Drupal\Core\Ajax\AjaxResponse;
use Drupal\Core\Ajax\AppendCommand;
use Drupal\Core\Ajax\CssCommand;
use Drupal\Core\Ajax\HtmlCommand;
use Drupal\Core\Ajax\InvokeCommand;
use Drupal\Core\Ajax\PrependCommand;
use Drupal\Core\Ajax\RemoveCommand;
use Drupal\Core\Ajax\ReplaceCommand;

$response = new AjaxResponse();

// Replace element inner HTML.
$response->addCommand(new HtmlCommand('#my-target', '<p>New content</p>'));

// Replace the entire element.
$response->addCommand(new ReplaceCommand('#my-target', '<div id="my-target">Replaced</div>'));

// jQuery method invocation.
$response->addCommand(new InvokeCommand('#my-field', 'val', ['new value']));

// Add CSS to the page.
$response->addCommand(new CssCommand('#my-target', ['background-color' => '#ffffcc']));

// Append / prepend / remove.
$response->addCommand(new AppendCommand('#list', '<li>Item</li>'));
$response->addCommand(new PrependCommand('#list', '<li>First</li>'));
$response->addCommand(new RemoveCommand('#stale-element'));
```

---

## `#ajax` properties reference

| Property | Description |
|----------|-------------|
| `callback` | Method returning render array or `AjaxResponse` |
| `wrapper` | HTML `id` of element to replace (without `#`) |
| `event` | DOM event (`click`, `change`, `blur`) — default `mousedown` |
| `progress.type` | `throbber` (default), `bar`, or `none` |
| `disable-refocus` | Prevent focus jump after AJAX — useful for accessibility |

---

## Dedicated AJAX controller (non-form)

For endpoints outside the Form API:

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Controller;

use Drupal\Core\Ajax\AjaxResponse;
use Drupal\Core\Ajax\OpenModalDialogCommand;
use Drupal\Core\Controller\ControllerBase;
use Symfony\Component\HttpFoundation\Request;

final class AjaxController extends ControllerBase {

  /**
   * Returns a modal dialog via AJAX.
   */
  public function openDialog(Request $request): AjaxResponse {
    $response = new AjaxResponse();
    $response->addCommand(new OpenModalDialogCommand(
      (string) $this->t('Hello'),
      ['#markup' => '<p>' . $this->t('Modal content.') . '</p>'],
      ['width' => '400'],
    ));
    return $response;
  }

}
```

```yaml
# example_module.routing.yml
example_module.ajax_dialog:
  path: '/example/ajax-dialog'
  defaults:
    _controller: '\Drupal\example_module\Controller\AjaxController::openDialog'
  requirements:
    _permission: 'access content'
```

!!! warning "Wrapper element must exist"
    The `wrapper` id must be on a `#type => container` element in the form. AJAX replaces that container's contents.

!!! tip "Debugging AJAX"
    Open browser DevTools → Network and inspect the POST to `/example/ajax-demo`. The response is JSON with `command` arrays.
