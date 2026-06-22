# Composer-First Development

Drupal has been a Composer-managed project since Drupal 8. In Drupal 11, Composer is the **only** supported installation method.

## Why Composer-first?

- **Reproducible builds** — lock file pins exact versions
- **Dependency management** — core, contrib, and PHP libraries in one place
- **Scaffold files** — `index.php`, `.htaccess` managed automatically
- **Patches** — apply fixes via `cweagans/composer-patches`
- **Autoloading** — PSR-4 autoloading for custom code

## Project template

Always start with the recommended project template:

```bash
composer create-project drupal/recommended-project my-site
cd my-site
```

This creates:

```
my-site/
├── composer.json           # All dependencies
├── composer.lock           # Pinned versions
├── web/                    # Document root (not project root!)
│   ├── core/
│   ├── modules/
│   │   ├── contrib/        # Composer-installed contrib
│   │   └── custom/         # Your custom modules
│   ├── themes/
│   ├── sites/
│   │   └── default/
│   │       ├── settings.php
│   │       └── files/
│   └── index.php
├── vendor/                 # PHP libraries
└── config/                 # Exported configuration
    └── sync/
```

## Essential Composer commands

### Require a contrib module

```bash
composer require drupal/admin_toolbar
drush en admin_toolbar -y
```

### Require a specific version

```bash
composer require 'drupal/webform:^6.3'
```

### Update a single package

```bash
composer update drupal/token --with-dependencies
```

### Update Drupal core

```bash
composer update drupal/core-recommended --with-dependencies
drush updatedb -y
drush cr
```

### Check for security advisories

```bash
composer audit
```

## Composer.json structure

```json
{
  "name": "drupal/recommended-project",
  "description": "Drupal 11 headless project",
  "type": "project",
  "require": {
    "composer/installers": "^2.0",
    "drupal/core-composer-scaffold": "^11",
    "drupal/core-recommended": "^11",
    "drupal/core-project-message": "^11",
    "drupal/jsonapi_extras": "^3.0",
    "drupal/simple_oauth": "^6.0",
    "drush/drush": "^13"
  },
  "conflict": {
    "drupal/drupal": "*"
  },
  "minimum-stability": "stable",
  "prefer-stable": true,
  "config": {
    "sort-packages": true,
    "allow-plugins": {
      "composer/installers": true,
      "drupal/core-composer-scaffold": true,
      "drupal/core-project-message": true,
      "phpstan/extension-installer": true,
      "dealerdirect/phpcodesniffer-composer-installer": true
    }
  },
  "extra": {
    "drupal-scaffold": {
      "locations": {
        "web-root": "web/"
      }
    },
    "installer-paths": {
      "web/core": ["type:drupal-core"],
      "web/modules/contrib/{$name}": ["type:drupal-module"],
      "web/themes/contrib/{$name}": ["type:drupal-theme"],
      "web/profiles/contrib/{$name}": ["type:drupal-profile"]
    }
  }
}
```

## Applying patches

Install the patches plugin:

```bash
composer require cweagans/composer-patches
```

Add patches to `composer.json`:

```json
"extra": {
  "patches": {
    "drupal/some_module": {
      "Fix API response: https://drupal.org/node/12345": "patches/some_module-api-fix.patch"
    }
  }
}
```

## PSR-4 autoloading for custom modules

```json
"autoload": {
  "psr-4": {
    "Drupal\\my_custom_module\\": "web/modules/custom/my_custom_module/src"
  }
}
```

Or define in the module's `composer.json`:

```json
{
  "name": "drupal/my_custom_module",
  "type": "drupal-module",
  "autoload": {
    "psr-4": {
      "Drupal\\my_custom_module\\": "src/"
    }
  }
}
```

## Headless project dependencies

Recommended Composer packages for headless Drupal:

```bash
# API enhancements
composer require drupal/jsonapi_extras
composer require drupal/jsonapi_menu_items
composer require drupal/jsonapi_views

# Authentication
composer require drupal/simple_oauth
composer require drupal/consumers

# CORS (built into Drupal core services)
# Configure in services.yml — no package needed

# Development
composer require --dev drupal/devel
composer require --dev mglaman/phpstan-drupal
```

See [Composer Workflows](../getting-started/composer.md) for detailed workflows.
