# Composer Workflows

Day-to-day Drupal development revolves around Composer. These workflows cover the most common tasks.

## Adding contrib modules

```bash
# Search for a module
composer search drupal/metatag

# Require and install
composer require drupal/metatag
drush en metatag -y

# Require with version constraint
composer require 'drupal/webform:^6.3@beta'
```

## Adding PHP libraries

```bash
# JWT library for custom auth
composer require firebase/php-jwt

# HTTP client
composer require guzzlehttp/guzzle

# Use in custom module
```

```php
<?php

use Firebase\JWT\JWT;
use Firebase\JWT\Key;

$token = JWT::encode($payload, $secret, 'HS256');
$decoded = JWT::decode($token, new Key($secret, 'HS256'));
```

## Updating dependencies

```bash
# Update everything (careful in production)
composer update

# Update only Drupal core
composer update drupal/core-recommended --with-dependencies

# Update a single module
composer update drupal/admin_toolbar

# Check what would be updated (dry run)
composer update --dry-run
```

## Removing packages

```bash
# Uninstall module first
drush pm:uninstall webform -y

# Remove via Composer
composer remove drupal/webform
```

## Managing patches

```bash
composer require cweagans/composer-patches
```

```json
{
  "extra": {
    "patches": {
      "drupal/core": {
        "Fix JSON:API filter: https://drupal.org/node/12345": "https://git.drupalcode.org/project/drupal/-/merge_requests/12345.diff"
      }
    },
    "enable-patching": true
  }
}
```

```bash
# Re-apply patches after update
composer install
```

## Scaffold management

Drupal's scaffold plugin manages files in `web/`:

```json
{
  "extra": {
    "drupal-scaffold": {
      "locations": {
        "web-root": "web/"
      },
      "file-mapping": {
        "[web-root]/robots.txt": false,
        "[web-root]/.htaccess": {
          "mode": "replace",
          "path": "assets/.htaccess",
          "overwrite": false
        }
      }
    }
  }
}
```

## Headless project bootstrap

Complete Composer workflow for a headless project:

```bash
# 1. Create project
composer create-project drupal/recommended-project my-headless
cd my-headless

# 2. Add headless dependencies
composer require \
  drupal/jsonapi_extras \
  drupal/simple_oauth \
  drupal/consumers \
  drupal/decoupled_router \
  drush/drush

# 3. Add dev dependencies
composer require --dev \
  drupal/devel \
  mglaman/phpstan-drupal \
  drupal/coder

# 4. Install site
drush site:install -y

# 5. Enable modules
drush en jsonapi jsonapi_extras simple_oauth consumers decoupled_router -y

# 6. Export config
drush config:export -y
```

## Lock file management

```bash
# Commit composer.lock to version control
git add composer.json composer.lock

# Install exact versions from lock file (CI/production)
composer install --no-dev --optimize-autoloader

# Verify lock file integrity
composer validate --strict
```

## Composer scripts

Add custom scripts to `composer.json`:

```json
{
  "scripts": {
    "post-update-cmd": [
      "Drupal\\Core\\Composer\\Composer::generateMetapackages",
      "@drupal:scaffold"
    ],
    "drupal:cr": "drush cr",
    "drupal:cex": "drush config:export -y",
    "drupal:cim": "drush config:import -y",
    "test": "phpunit --configuration web/core/phpunit.xml.dist"
  }
}
```

```bash
composer drupal:cr
composer drupal:cex
```

## Security auditing

```bash
# Check for known vulnerabilities
composer audit

# Drupal-specific security check
drush pm:security

# Update only security releases
composer update --with-all-dependencies --prefer-stable
```
