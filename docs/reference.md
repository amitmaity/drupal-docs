# Reference

Quick-reference cheat sheet for Drupal 11 headless and custom development.

## Essential Drush commands

```bash
drush cr                          # Clear caches
drush cex -y                      # Export config
drush cim -y                      # Import config
drush updatedb -y                 # Run DB updates
drush en jsonapi -y               # Enable module
drush pm:uninstall module -y      # Uninstall module
drush user:login                  # One-time login link
drush route | grep jsonapi        # Find routes
drush queue:run queue_name         # Process queue
drush cron                        # Run cron
drush generate module             # Scaffold module
drush php:eval "code"             # Run PHP
```

## Composer commands

```bash
composer require drupal/module_name
composer update "drupal/core-*" --with-all-dependencies
composer remove drupal/module_name
composer audit
composer install --no-dev --optimize-autoloader
```

## File patterns

| File | Purpose |
|------|---------|
| `*.info.yml` | Module/theme metadata |
| `*.services.yml` | Service definitions |
| `*.routing.yml` | Route definitions |
| `*.permissions.yml` | Permission definitions |
| `*.links.menu.yml` | Admin menu links |
| `*.links.task.yml` | Local tasks (tabs) |
| `*.install` | Install/uninstall hooks |
| `config/install/*.yml` | Default config on install |
| `config/schema/*.schema.yml` | Config schema |

## Module skeleton

```
my_module/
├── my_module.info.yml
├── my_module.module
├── my_module.services.yml
├── my_module.routing.yml
├── my_module.permissions.yml
├── config/install/
├── src/
│   ├── Controller/
│   ├── Form/
│   ├── Plugin/
│   ├── Hook/
│   └── Service/
└── tests/src/
```

## Service definition template

```yaml
services:
  my_module.my_service:
    class: Drupal\my_module\Service\MyService
    arguments: ['@entity_type.manager', '@logger.channel.my_module']

  logger.channel.my_module:
    parent: logger.channel_base
    arguments: ['my_module']
```

## Route definition template

```yaml
my_module.api_endpoint:
  path: '/api/v1/items'
  defaults:
    _controller: '\Drupal\my_module\Controller\ApiController::list'
    _format: 'json'
  requirements:
    _permission: 'access content'
  methods: [GET]
```

## OOP Hook template (11.1+)

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Hook;

use Drupal\Core\Hook\Attribute\Hook;

final class MyModuleHooks {

  #[Hook('cron')]
  public function cron(): void {
    // Scheduled task logic.
  }
}
```

## JSON:API quick reference

```bash
# List articles
GET /jsonapi/node/article

# Filter by title
GET /jsonapi/node/article?filter[title-filter][condition][path]=title&filter[title-filter][condition][value]=Hello

# Include relationships
GET /jsonapi/node/article?include=uid,field_image

# Sparse fieldsets
GET /jsonapi/node/article?fields[node--article]=title,created&fields[user--user]=name

# Pagination
GET /jsonapi/node/article?page[limit]=10&page[offset]=20

# Create (POST with CSRF + auth)
POST /jsonapi/node/article
Content-Type: application/vnd.api+json

# Update (PATCH)
PATCH /jsonapi/node/article/{uuid}

# Delete
DELETE /jsonapi/node/article/{uuid}
```

## Cache metadata template

```php
$build = [
  '#markup' => $output,
  '#cache' => [
    'tags' => ['node:1', 'node_list'],
    'contexts' => ['user.permissions', 'languages:language_interface'],
    'max-age' => 3600,
  ],
];
```

## Bootstrap phases

Drupal 8+ has no numbered bootstrap phases (the `DRUPAL_BOOTSTRAP_*` constants are Drupal 7). A request goes through these `DrupalKernel` steps:

| Step | Method | What it does |
|------|--------|--------------|
| 1 | `DrupalKernel::handle()` | Calls `bootEnvironment()` and `initializeSettings()` (loads `settings.php`) |
| 2 | `DrupalKernel::boot()` | Sets up the file cache and builds or loads the service container (`initializeContainer()`) |
| 3 | `http_kernel` middleware stack | `handle()` passes the request to the stack (page cache, reverse proxy, session, …) |
| 4 | `DrupalKernel::preHandle()` | Called by the `KernelPreHandle` middleware: sanitizes the request, loads all modules, registers stream wrappers, pushes the request onto `request_stack` |
| 5 | Symfony `HttpKernel` | Dispatches `kernel.request`, routes, runs the controller, dispatches `kernel.response` |
| 6 | `DrupalKernel::terminate()` | Terminates the HTTP kernel, destructs `needs_destruction` services, pops the request stack |

## Recommended contrib modules (headless)

| Module | Purpose |
|--------|---------|
| `jsonapi_extras` | Customize JSON:API resources |
| `jsonapi_menu_items` | Expose menus via JSON:API |
| `simple_oauth` | OAuth 2.0 authentication |
| `consumers` | API consumer management |
| `decoupled_router` | Path resolution for frontends |
| `graphql` | GraphQL API |
| `subrequests` | Batch API requests |
| `webhooks` | Event-driven cache invalidation |

## Symfony components in Drupal 11

| Component | Service ID (common) |
|-----------|---------------------|
| HttpFoundation | `request_stack` |
| Routing | `router.route_provider` |
| EventDispatcher | `event_dispatcher` |
| Serializer | `serializer` |
| Validator | `validation.constraint` (constraint plugin manager) |

## PHP version features used

| Feature | Minimum PHP |
|---------|-------------|
| Typed properties | 7.4 |
| Constructor promotion | 8.0 |
| Attributes | 8.0 |
| Enums | 8.1 |
| Readonly classes | 8.2 |
| Typed class constants | 8.3 |
| `json_validate()` | 8.3 |

## Useful links

- [Drupal.org API documentation](https://api.drupal.org/api/drupal)
- [Symfony 7 documentation](https://symfony.com/doc/7.0/index.html)
- [JSON:API specification](https://jsonapi.org/format/)
- [next-drupal documentation](https://next-drupal.org/)
- [Drush documentation](https://www.drush.org/latest/)
- [This guide on GitHub Pages](https://amitmaity.github.io/drupal-docs/)
