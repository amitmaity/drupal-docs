# Access & Permissions

Drupal controls access through **permissions** (role-based), **route requirements**, and custom **access check services**.

## Permissions — `example_module.permissions.yml`

```yaml
administer example module:
  title: 'Administer example module'
  description: 'Configure example module settings.'
  restrict access: true

view example module greeting:
  title: 'View greeting page'
  description: 'Access the /example/greeting page.'

create example tasks:
  title: 'Create tasks'

edit example tasks:
  title: 'Edit any task'

view example tasks:
  title: 'View tasks'
```

Assign permissions at **People → Permissions** (`/admin/people/permissions`).

---

## Route requirements

### Permission-based

```yaml
# example_module.routing.yml
example_module.settings:
  path: '/admin/config/system/example-module'
  defaults:
    _form: '\Drupal\example_module\Form\SettingsForm'
    _title: 'Settings'
  requirements:
    _permission: 'administer example module'
```

### Role-based

```yaml
example_module.admin_only:
  path: '/example/admin'
  defaults:
    _controller: '\Drupal\example_module\Controller\AdminController::page'
  requirements:
    _role: 'administrator'
```

### Multiple permissions (OR)

```yaml
example_module.reports:
  path: '/example/reports'
  defaults:
    _controller: '\Drupal\example_module\Controller\ReportController::list'
  requirements:
    _permission: 'view example tasks+administer example module'
```

---

## Custom access check service

### Access checker — `src/Access/GreetingAccessCheck.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Access;

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Routing\Access\AccessInterface;
use Drupal\Core\Session\AccountInterface;
use Symfony\Component\Routing\Route;

/**
 * Determines access to the greeting route.
 */
final class GreetingAccessCheck implements AccessInterface {

  /**
   * Checks access.
   *
   * Route requirement key: _example_module_greeting_access
   */
  public function access(Route $route, AccountInterface $account): AccessResult {
    // Deny blocked users immediately.
    if ($account->isAnonymous()) {
      return AccessResult::forbidden('Anonymous users cannot view greetings.')
        ->cachePerPermissions();
    }

    if ($account->hasPermission('view example module greeting')) {
      return AccessResult::allowed()->cachePerPermissions();
    }

    return AccessResult::forbidden()->cachePerPermissions();
  }

}
```

### Service definition — `example_module.services.yml`

```yaml
services:
  example_module.greeting_access_check:
    class: Drupal\example_module\Access\GreetingAccessCheck
    tags:
      - { name: access_check, applies_to: _example_module_greeting_access }
```

### Route using custom check

```yaml
example_module.greeting:
  path: '/example/greeting'
  defaults:
    _controller: '\Drupal\example_module\Controller\GreetingController::greeting'
    _title: 'Greeting'
  requirements:
    _example_module_greeting_access: 'TRUE'
```

The requirement value (`'TRUE'`) is passed to the access checker but most checks ignore it.

---

## Entity access handler

See [Content Entities](entities/content-entities.md) for `EntityAccessControlHandler` examples. Quick reference:

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module;

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Entity\EntityAccessControlHandler;
use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Session\AccountInterface;

final class TaskAccessControlHandler extends EntityAccessControlHandler {

  protected function checkAccess(EntityInterface $entity, $operation, AccountInterface $account): AccessResult {
    /** @var \Drupal\example_module\TaskInterface $entity */
    if ($operation === 'view') {
      return AccessResult::allowedIfHasPermission($account, 'view example tasks')
        ->cachePerPermissions();
    }

    if ($operation === 'update') {
      // getOwnerId() requires EntityOwnerInterface on the entity.
      return AccessResult::allowedIfHasPermission($account, 'edit example tasks')
        ->orIf(AccessResult::allowedIf((int) $entity->getOwnerId() === (int) $account->id()))
        ->cachePerPermissions()
        ->cachePerUser()
        ->addCacheableDependency($entity);
    }

    return AccessResult::neutral();
  }

  protected function checkCreateAccess(AccountInterface $account, array $context, $entity_bundle = NULL): AccessResult {
    return AccessResult::allowedIfHasPermission($account, 'create example tasks');
  }

}
```

---

## Controller `_access` method

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Controller;

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Controller\ControllerBase;
use Drupal\Core\Session\AccountInterface;

final class SecureController extends ControllerBase {

  public function page(): array {
    return ['#markup' => $this->t('Secret content.')];
  }

  /**
   * Access callback for the route.
   */
  public function access(AccountInterface $account): AccessResult {
    return AccessResult::allowedIf(
      $account->hasPermission('administer example module') && $account->id() !== 0,
    )->cachePerPermissions();
  }

}
```

```yaml
example_module.secure:
  path: '/example/secure'
  defaults:
    _controller: '\Drupal\example_module\Controller\SecureController::page'
  requirements:
    _custom_access: '\Drupal\example_module\Controller\SecureController::access'
```

---

## AccessResult patterns

```php
<?php

declare(strict_types=1);

use Drupal\Core\Access\AccessResult;
use Drupal\node\NodeInterface;

// Allowed / forbidden / neutral.
AccessResult::allowed();
AccessResult::forbidden('Reason string');
AccessResult::neutral();

// Conditional.
AccessResult::allowedIf($condition);
AccessResult::allowedIfHasPermission($account, 'permission');
AccessResult::forbiddenIf($condition);

// Combine.
AccessResult::allowedIfHasPermission($account, 'edit example tasks')
  ->orIf(AccessResult::allowedIf((int) $node->getOwnerId() === (int) $account->id())->cachePerUser());

// Cache metadata (required for performance).
AccessResult::allowed()
  ->cachePerPermissions()
  ->cachePerUser()
  ->addCacheableDependency($node);
```

---

## Testing access

```bash
# Evaluate route access for user 1
ddev drush php:eval "
  \$account = \Drupal\user\Entity\User::load(1);
  \$access = \Drupal::service('router.route_provider')
    ->getRouteByName('example_module.greeting');
  print_r(\Drupal::service('access_manager')->checkNamedRoute(
    'example_module.greeting', [], \$account, TRUE
  ));
"
```

!!! warning "Always add cache metadata"
    `AccessResult` without cache contexts causes incorrect access for other users. Use `cachePerPermissions()`, `cachePerUser()`, or `addCacheableDependency()`.

!!! note "accessCheck on queries"
    `accessCheck(TRUE)` on an entity query does **not** call the entity access handler (`checkAccess()`). It only adds the `{entity_type}_access` query tag, which applies query-level access such as node grants. Check `$entity->access('view')` on loaded entities when per-entity rules matter. Database API queries need `->addTag('node_access')` or equivalent.

!!! tip "Security review"
    Restrict `administer *` permissions with `restrict access: true` so only trusted roles receive them.
