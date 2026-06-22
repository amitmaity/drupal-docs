# Project Layout

Understanding the Drupal 11 project structure is essential for custom development.

## Top-level structure

```
my-drupal-site/
├── composer.json              # Project dependencies
├── composer.lock              # Pinned versions
├── vendor/                    # PHP libraries (Composer)
├── web/                       # Document root (public)
│   ├── index.php              # Front controller
│   ├── autoload.php           # Composer autoloader
│   ├── core/                  # Drupal core (do not edit)
│   ├── modules/
│   │   ├── contrib/           # Composer-installed modules
│   │   └── custom/            # YOUR custom modules
│   ├── themes/
│   │   ├── contrib/
│   │   └── custom/
│   ├── profiles/
│   ├── sites/
│   │   └── default/
│   │       ├── settings.php   # Site settings
│   │       ├── services.yml   # Service overrides
│   │       └── files/         # Public uploads
│   └── libraries/             # Front-end libraries
├── config/
│   └── sync/                  # Exported configuration
└── patches/                   # Composer patches
```

!!! warning "Never edit core"
    Files in `web/core/` are managed by Composer. Never modify them directly. Use hooks, plugins, and service overrides instead.

## Custom module location

All custom code lives in `web/modules/custom/`:

```
web/modules/custom/my_api/
├── my_api.info.yml
├── my_api.module
├── my_api.services.yml
├── my_api.routing.yml
├── my_api.permissions.yml
├── config/
│   └── install/
│       └── my_api.settings.yml
├── src/
│   ├── Controller/
│   ├── Form/
│   ├── Plugin/
│   ├── Hook/
│   └── Service/
└── tests/
    └── src/
        ├── Unit/
        └── Kernel/
```

## Key configuration files

### settings.php

```php
// web/sites/default/settings.php

// Database connection
$databases['default']['default'] = [
  'database' => 'drupal',
  'username' => 'drupal',
  'password' => 'drupal',
  'host' => 'db',
  'driver' => 'mysql',
];

// Config sync directory
$settings['config_sync_directory'] = '../config/sync';

// Trusted host patterns (security)
$settings['trusted_host_patterns'] = [
  '^my-site\.ddev\.site$',
  '^www\.myproductionsite\.com$',
];

// Hash salt (unique per site)
$settings['hash_salt'] = 'your-unique-random-string-here';

// File paths
$settings['file_public_path'] = 'sites/default/files';
$settings['file_private_path'] = '../private';
```

### services.yml

Override core services for development or headless:

```yaml
# web/sites/default/services.yml
parameters:
  cors.config:
    enabled: true
    allowedHeaders: ['*']
    allowedMethods: ['GET', 'POST', 'PATCH', 'DELETE', 'OPTIONS']
    allowedOrigins: ['http://localhost:3000']
    supportsCredentials: true
    maxAge: 3600

  http.response.debug_cacheability_headers: true

services:
  cache.backend.null:
    class: Drupal\Core\Cache\NullBackendFactory
```

### Module info file

```yaml
# my_api.info.yml
name: 'My API'
type: module
description: 'Custom API endpoints for headless frontend.'
package: Custom
core_version_requirement: ^11
dependencies:
  - drupal:serialization
  - drupal:rest
```

## Config sync directory

Exported configuration for version control:

```
config/sync/
├── core.entity_form_display.node.article.default.yml
├── core.entity_view_display.node.article.default.yml
├── field.field.node.article.body.yml
├── field.storage.node.body.yml
├── node.type.article.yml
├── system.site.yml
└── views.view.content.yml
```

```bash
# Export all config
drush config:export -y

# Import config (e.g., after deployment)
drush config:import -y

# Diff config before import
drush config:status
```

## Multisite structure

For multiple sites from one codebase:

```
web/sites/
├── default/           # Default site
├── site1/             # site1.example.com
│   ├── settings.php
│   └── files/
├── site2/             # site2.example.com
│   ├── settings.php
│   └── files/
└── sites.php          # Domain mapping
```

```php
// web/sites/sites.php
$sites['site1.example.com'] = 'site1';
$sites['site2.example.com'] = 'site2';
```

## Private files

Store sensitive files outside the web root:

```
my-drupal-site/
├── private/           # Not web-accessible
│   └── exports/
└── web/
    └── sites/default/files/  # Public uploads
```

```php
$settings['file_private_path'] = '../private';
```

```php
// Generate private file URL (requires access check)
$url = \Drupal::service('file_url_generator')
  ->generateAbsoluteString('private://exports/report.pdf');
```
