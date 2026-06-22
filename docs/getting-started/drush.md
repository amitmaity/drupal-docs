# Drush Basics

[Drush](https://www.drush.org/) is Drupal's command-line shell. It is essential for development, deployment, and site management.

## Installation

```bash
composer require drush/drush
```

## Essential commands

### Site management

```bash
# Check site status
drush status

# Install a fresh site
drush site:install standard --account-name=admin --account-pass=admin -y

# Run database updates after core/module updates
drush updatedb -y

# Clear all caches
drush cr
```

### Module management

```bash
# List all modules
drush pm:list --type=module

# Enable modules
drush en jsonapi rest serialization -y

# Disable modules
drush pm:uninstall devel -y

# Check security advisories
drush pm:security
```

### Configuration

```bash
# Export configuration
drush config:export -y

# Import configuration
drush config:import -y

# Show config differences
drush config:status

# Get a single config value
drush config:get system.site name

# Set a config value
drush config:set system.site name "My Headless Site" -y
```

### Entity operations

```bash
# List content types
drush entity:info

# Create a node
drush php:eval "
  \$node = \Drupal\node\Entity\Node::create([
    'type' => 'article',
    'title' => 'Created via Drush',
  ]);
  \$node->save();
  echo 'Created node ' . \$node->id();
"

# Delete all nodes of a type
drush entity:delete node --bundle=article
```

### User management

```bash
# Create a user
drush user:create apiuser --mail="api@example.com" --password="secret"

# Login link (one-time)
drush user:login

# Reset password
drush user:password admin "newpassword"
```

### Database

```bash
# Open SQL CLI
drush sql:cli

# Dump database
drush sql:dump --result-file=backup.sql

# Import database
drush sql:cli < backup.sql
```

### Development

```bash
# Generate a module
drush generate module

# Generate a service
drush generate service

# Generate a plugin
drush generate plugin:block

# Run PHP code
drush php:eval "echo \Drupal::VERSION;"

# Open PHP REPL
drush php:cli
```

### Routing

```bash
# List all routes
drush route

# Find a specific route
drush route | grep jsonapi

# Debug a route
drush route:debug jsonapi.node--article--canonical
```

### Queue and cron

```bash
# Run cron
drush cron

# List queues
drush queue:list

# Process a queue
drush queue:run my_module_import_queue
```

## Custom Drush commands

See [Drush Commands](../custom-coding/drush-commands.md) for building your own.

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Commands;

use Drush\Commands\DrushCommands;

final class MyModuleCommands extends DrushCommands {

  #[CLI\Command(name: 'my-module:reindex', aliases: ['mmri'])]
  #[CLI\Argument(name: 'type', description: 'Content type to reindex')]
  public function reindex(string $type): void {
    $this->output()->writeln("Reindexing content type: $type");
    // Reindex logic here
    $this->logger()->success("Reindex complete.");
  }
}
```

```bash
drush my-module:reindex article
```

## Aliases for remote sites

```yaml
# drush/sites/self.site.yml
prod:
  host: prod.example.com
  user: deploy
  root: /var/www/drupal/web
  uri: https://www.example.com

staging:
  host: staging.example.com
  user: deploy
  root: /var/www/drupal/web
  uri: https://staging.example.com
```

```bash
# Run command on production
drush @prod cr
drush @prod updatedb -y
drush @prod config:import -y
```

## DDEV integration

```bash
# All drush commands via DDEV
ddev drush status
ddev drush cr
ddev drush en jsonapi -y
ddev drush user:login
```
