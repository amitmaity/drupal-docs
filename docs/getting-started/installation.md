# Fresh Drupal 11 Installation

This guide installs Drupal 11 using DDEV and Composer — the recommended approach for development.

## Step 1: Install DDEV

=== "macOS"

    ```bash
    brew install ddev/ddev/ddev
    ```

=== "Linux"

    ```bash
    curl -fsSL https://ddev.com/install.sh | bash
    ```

=== "Windows (WSL2)"

    ```bash
    curl -fsSL https://ddev.com/install.sh | bash
    ```

## Step 2: Create the project

```bash
mkdir my-drupal-site && cd my-drupal-site

# Configure DDEV
ddev config \
  --project-type=drupal11 \
  --docroot=web \
  --create-docroot \
  --php-version=8.3

# Start containers
ddev start

# Create Drupal project via Composer
ddev composer create "drupal/recommended-project:^11" --no-install
ddev composer install
```

## Step 3: Install Drupal

```bash
# Copy default settings
cp web/sites/default/default.settings.php web/sites/default/settings.php
cp web/sites/default/default.services.yml web/sites/default/services.yml
chmod 666 web/sites/default/settings.php

# Install via Drush
ddev drush site:install standard \
  --site-name="My Headless Site" \
  --account-name=admin \
  --account-pass=admin \
  -y
```

## Step 4: Verify installation

```bash
# Open in browser
ddev launch

# Check Drupal version
ddev drush status

# Expected output includes:
# Drupal version   : 11.x.x
# Database         : Connected
# Drupal bootstrap : Successful
```

## Step 5: Enable headless modules

```bash
ddev drush en jsonapi rest serialization -y

# Verify JSON:API
curl -s "$(ddev describe -j | jq -r '.raw.primary_url')/jsonapi" | head -20
```

## Step 6: Configure for development

```bash
# Enable development services
ddev drush config:set system.logging error_level verbose -y

# Disable CSS/JS aggregation for debugging
ddev drush config:set system.performance css.preprocess 0 -y
ddev drush config:set system.performance js.preprocess 0 -y
ddev drush cr
```

## Alternative: Manual Composer install (no DDEV)

```bash
composer create-project drupal/recommended-project my-site
cd my-site

# Configure database in web/sites/default/settings.php
# Then install:
cd web
../vendor/bin/drush site:install standard -y
```

### Database settings example

```php
// web/sites/default/settings.php
$databases['default']['default'] = [
  'database' => 'drupal',
  'username' => 'drupal',
  'password' => 'drupal',
  'host' => '127.0.0.1',
  'port' => '3306',
  'driver' => 'mysql',
  'prefix' => '',
  'collation' => 'utf8mb4_general_ci',
];
```

## Troubleshooting

!!! warning "Permission errors"
    ```bash
    # Fix files directory permissions
    chmod -R 775 web/sites/default/files
    chown -R www-data:www-data web/sites/default/files
    ```

!!! warning "Composer memory limit"
    ```bash
    COMPOSER_MEMORY_LIMIT=-1 composer install
    ```

!!! tip "DDEV xdebug"
    Enable Xdebug for debugging:
    ```bash
    ddev xdebug on
    ```
