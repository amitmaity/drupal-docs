# Security

Drupal uses Symfony's **PasswordHasher** component for secure password storage and Drupal's **CSRF** token system (built on session and private key storage) to protect state-changing requests.

## Core concepts

| Feature | Symfony/Drupal class | Purpose |
|---------|---------------------|---------|
| Password hashing | `PasswordHasherInterface` | Bcrypt/argon2 hashing |
| CSRF tokens | `CsrfTokenGenerator` | Validate form and URL tokens |
| Access checks | `_csrf_token`, `form_token` | Prevent cross-site request forgery |

!!! info "Not the full Security component"
    Drupal does not use Symfony's firewall, voters, or authentication bundle. Access control is Drupal's permission and entity access system. This page covers password hashing and CSRF — the primary Symfony Security pieces in core.

## How Drupal uses Security

### Password hashing

User passwords are hashed via `password` service:

```php
<?php

declare(strict_types=1);

/** @var \Drupal\Core\Password\PasswordInterface $password */
$password = \Drupal::service('password');

$hash = $password->hash('my-secure-password');
$valid = $password->check('my-secure-password', $hash);
```

Drupal 11 defaults to PHP's `password_hash()` with `PASSWORD_DEFAULT` (currently bcrypt). Symfony's `NativePasswordHasher` underpins the implementation.

### Checking passwords on login

Never compare plain text — always use `check()`:

```php
<?php

declare(strict_types=1);

use Drupal\user\Entity\User;

$user = user_load_by_name('admin');
$storedHash = $user->getPassword();

if (\Drupal::service('password')->check($submittedPassword, $storedHash)) {
  // Authenticate…
}
```

### Rehashing on login

If the hash algorithm changes, Drupal rehashes on successful login via `user_login_finalize()`.

## CSRF protection

### Forms

Every Form API form gets a `form_token` and `form_build_id` automatically. Do not disable unless you have an alternative.

### Route requirement `_csrf_token`

Protect destructive GET routes (prefer POST, but Drupal admin UIs sometimes use GET with tokens):

```yaml
my_module.product_delete:
  path: '/admin/products/{product}/delete'
  defaults:
    _form: '\Drupal\my_module\Form\ProductDeleteForm'
  requirements:
    _entity_access: 'product.delete'
```

Forms include `confirm_form` with built-in CSRF.

### Manual token validation

```php
<?php

declare(strict_types=1);

use Drupal\Core\Access\CsrfTokenGenerator;

final class WebhookController extends ControllerBase {

  public function __construct(
    private readonly CsrfTokenGenerator $csrfToken,
  ) {}

  public function handle(Request $request): JsonResponse {
    $token = $request->headers->get('X-CSRF-Token');

    if (!$this->csrfToken->validate($token, 'my_module/webhook')) {
      return new JsonResponse(['error' => 'Invalid CSRF token.'], 403);
    }

    // Process webhook…
    return new JsonResponse(['status' => 'ok']);
  }

}
```

### Generating tokens for JavaScript

```php
<?php

declare(strict_types=1);

$token = \Drupal::csrfToken()->get('my_module/api');

// In render array:
$form['#attached']['drupalSettings']['myModule']['csrfToken'] = $token;
```

Headless session auth requires fetching `session/token` (REST) or `X-CSRF-Token` header with cookie authentication.

## Custom module example: secure API registration

=== "Registration service"

    ```php
    <?php

    declare(strict_types=1);

    namespace Drupal\my_module\Service;

    use Drupal\Core\Password\PasswordInterface;
    use Drupal\user\Entity\User;

    final class UserRegistrationService {

      public function __construct(
        private readonly PasswordInterface $password,
      ) {}

      public function register(string $username, string $email, string $plainPassword): User {
        if (strlen($plainPassword) < 12) {
          throw new \InvalidArgumentException('Password must be at least 12 characters.');
        }

        $user = User::create([
          'name' => $username,
          'mail' => $email,
          'status' => 1,
        ]);

        // Never store plain text — hash before setPassword().
        $user->setPassword($plainPassword);
        $user->save();

        return $user;
      }

      public function verifyCredentials(User $user, string $plainPassword): bool {
        return $this->password->check($plainPassword, $user->getPassword());
      }

    }
    ```

=== "Form with CSRF (automatic)"

    ```php
    <?php

    declare(strict_types=1);

    namespace Drupal\my_module\Form;

    use Drupal\Core\Form\FormBase;
    use Drupal\Core\Form\FormStateInterface;
    use Drupal\my_module\Service\UserRegistrationService;

    final class RegisterForm extends FormBase {

      public function __construct(
        private readonly UserRegistrationService $registration,
      ) {}

      public static function create($container): static {
        return new static($container->get('my_module.user_registration'));
      }

      public function getFormId(): string {
        return 'my_module_register';
      }

      public function buildForm(array $form, FormStateInterface $form_state): array {
        $form['username'] = ['#type' => 'textfield', '#title' => $this->t('Username'), '#required' => TRUE];
        $form['email'] = ['#type' => 'email', '#title' => $this->t('Email'), '#required' => TRUE];
        $form['password'] = ['#type' => 'password', '#title' => $this->t('Password'), '#required' => TRUE];
        $form['actions']['submit'] = ['#type' => 'submit', '#value' => $this->t('Register')];

        // form_token added automatically by FormBuilder.
        return $form;
      }

      public function submitForm(array &$form, FormStateInterface $form_state): void {
        $this->registration->register(
          $form_state->getValue('username'),
          $form_state->getValue('email'),
          $form_state->getValue('password'),
        );
        $this->messenger()->addStatus($this->t('Account created.'));
      }

    }
    ```

=== "services.yml"

    ```yaml
    my_module.user_registration:
      class: Drupal\my_module\Service\UserRegistrationService
      arguments: ['@password']
    ```

!!! danger "Never log or cache passwords"
    Do not write plain passwords to logs, cache, or tempstore. Hash immediately and discard the plain string.

## Related reading

- [Headless: Session Auth](../headless/auth/session-auth.md)
- [Headless: Cookie Auth](../headless/auth/cookie-auth.md)
- [Custom Modules: Access & Permissions](../custom-coding/access-permissions.md)
