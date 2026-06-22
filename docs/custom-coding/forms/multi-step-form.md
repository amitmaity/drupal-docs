# Multi-step Form

Multi-step (wizard) forms store intermediate data in `$form_state` and branch `buildForm()` based on a step counter. Each step validates independently before advancing.

## Route — `example_module.routing.yml`

```yaml
example_module.wizard:
  path: '/example/wizard'
  defaults:
    _form: '\Drupal\example_module\Form\WizardForm'
    _title: 'Registration wizard'
  requirements:
    _permission: 'access content'
```

---

## Wizard form — `src/Form/WizardForm.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Form;

use Drupal\Core\Form\FormBase;
use Drupal\Core\Form\FormStateInterface;

/**
 * Three-step registration wizard.
 */
final class WizardForm extends FormBase {

  private const STEP_ACCOUNT = 1;
  private const STEP_PROFILE = 2;
  private const STEP_CONFIRM = 3;

  /**
   * {@inheritdoc}
   */
  public function getFormId(): string {
    return 'example_module_wizard';
  }

  /**
   * {@inheritdoc}
   */
  public function buildForm(array $form, FormStateInterface $form_state): array {
    $step = $this->getStep($form_state);

    $form['step'] = [
      '#type' => 'value',
      '#value' => $step,
    ];

    $form['progress'] = [
      '#markup' => '<p>' . $this->t('Step @current of @total', [
        '@current' => $step,
        '@total' => self::STEP_CONFIRM,
      ]) . '</p>',
    ];

    switch ($step) {
      case self::STEP_ACCOUNT:
        $form = $this->buildAccountStep($form, $form_state);
        break;

      case self::STEP_PROFILE:
        $form = $this->buildProfileStep($form, $form_state);
        break;

      case self::STEP_CONFIRM:
        $form = $this->buildConfirmStep($form, $form_state);
        break;
    }

    return $form;
  }

  /**
   * Step 1: account credentials.
   */
  private function buildAccountStep(array $form, FormStateInterface $form_state): array {
    $form['email'] = [
      '#type' => 'email',
      '#title' => $this->t('Email'),
      '#default_value' => $form_state->get('email', ''),
      '#required' => TRUE,
    ];

    $form['password'] = [
      '#type' => 'password',
      '#title' => $this->t('Password'),
      '#required' => TRUE,
    ];

    $form['actions']['next'] = [
      '#type' => 'submit',
      '#value' => $this->t('Next'),
      '#submit' => ['::nextStep'],
    ];

    return $form;
  }

  /**
   * Step 2: profile details.
   */
  private function buildProfileStep(array $form, FormStateInterface $form_state): array {
    $form['first_name'] = [
      '#type' => 'textfield',
      '#title' => $this->t('First name'),
      '#default_value' => $form_state->get('first_name', ''),
      '#required' => TRUE,
    ];

    $form['last_name'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Last name'),
      '#default_value' => $form_state->get('last_name', ''),
      '#required' => TRUE,
    ];

    $form['bio'] = [
      '#type' => 'textarea',
      '#title' => $this->t('Bio'),
      '#default_value' => $form_state->get('bio', ''),
    ];

    $form['actions']['back'] = [
      '#type' => 'submit',
      '#value' => $this->t('Back'),
      '#submit' => ['::previousStep'],
      '#limit_validation_errors' => [],
    ];

    $form['actions']['next'] = [
      '#type' => 'submit',
      '#value' => $this->t('Next'),
      '#submit' => ['::nextStep'],
    ];

    return $form;
  }

  /**
   * Step 3: review and confirm.
   */
  private function buildConfirmStep(array $form, FormStateInterface $form_state): array {
    $form['summary'] = [
      '#type' => 'item',
      '#title' => $this->t('Review your information'),
      '#markup' => $this->buildSummaryMarkup($form_state),
    ];

    $form['actions']['back'] = [
      '#type' => 'submit',
      '#value' => $this->t('Back'),
      '#submit' => ['::previousStep'],
      '#limit_validation_errors' => [],
    ];

    $form['actions']['submit'] = [
      '#type' => 'submit',
      '#value' => $this->t('Complete registration'),
    ];

    return $form;
  }

  /**
   * {@inheritdoc}
   */
  public function validateForm(array &$form, FormStateInterface $form_state): void {
    $step = $this->getStep($form_state);

    if ($step === self::STEP_ACCOUNT) {
      $password = (string) $form_state->getValue('password');
      if (strlen($password) < 8) {
        $form_state->setErrorByName('password', $this->t('Password must be at least 8 characters.'));
      }
    }
  }

  /**
   * Advance to the next wizard step.
   */
  public function nextStep(array &$form, FormStateInterface $form_state): void {
    $step = $this->getStep($form_state);

    if ($step === self::STEP_ACCOUNT) {
      $form_state->set('email', $form_state->getValue('email'));
    }
    elseif ($step === self::STEP_PROFILE) {
      $form_state->set('first_name', $form_state->getValue('first_name'));
      $form_state->set('last_name', $form_state->getValue('last_name'));
      $form_state->set('bio', $form_state->getValue('bio'));
    }

    $form_state->set('step', $step + 1);
    $form_state->setRebuild(TRUE);
  }

  /**
   * Return to the previous wizard step.
   */
  public function previousStep(array &$form, FormStateInterface $form_state): void {
    $step = $this->getStep($form_state);
    $form_state->set('step', max(self::STEP_ACCOUNT, $step - 1));
    $form_state->setRebuild(TRUE);
  }

  /**
   * {@inheritdoc}
   */
  public function submitForm(array &$form, FormStateInterface $form_state): void {
    \Drupal::logger('example_module')->info('Wizard completed for @email', [
      '@email' => $form_state->get('email'),
    ]);

    $this->messenger()->addStatus($this->t('Registration complete!'));
    $form_state->setRedirect('<front>');
  }

  /**
   * Returns the current wizard step.
   */
  private function getStep(FormStateInterface $form_state): int {
    return (int) $form_state->get('step', self::STEP_ACCOUNT);
  }

  /**
   * Builds HTML summary for the confirm step.
   */
  private function buildSummaryMarkup(FormStateInterface $form_state): string {
    $lines = [
      $this->t('Email: @email', ['@email' => $form_state->get('email')]),
      $this->t('Name: @first @last', [
        '@first' => $form_state->get('first_name'),
        '@last' => $form_state->get('last_name'),
      ]),
      $this->t('Bio: @bio', ['@bio' => $form_state->get('bio') ?: $this->t('(none)')]),
    ];

    return '<ul><li>' . implode('</li><li>', $lines) . '</li></ul>';
  }

}
```

---

## Key patterns

| Technique | Purpose |
|-----------|---------|
| `$form_state->set('step', N)` | Persist step between rebuilds |
| `$form_state->setRebuild(TRUE)` | Re-run `buildForm()` after navigation |
| `#limit_validation_errors => []` on Back | Skip validation when going backward |
| `$form_state->set('field', $value)` | Store data from earlier steps |
| Custom `#submit` handlers | Separate Next/Back from final submit |

!!! tip "Persist across requests"
    For long wizards, store `$form_state` data in PrivateTempStore:

    ```php
    $store = \Drupal::service('tempstore.private')->get('example_module_wizard');
    $store->set('email', $form_state->getValue('email'));
    ```

!!! warning "Security"
    Never store passwords in tempstore or hidden fields. Re-prompt for credentials on the final step or create the account immediately after step 1.
