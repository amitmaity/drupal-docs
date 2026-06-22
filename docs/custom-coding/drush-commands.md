# Drush Commands

Custom Drush commands extend Drush's CLI using PHP 8 attributes. Commands live in `src/Commands/` and are registered via `drush.services.yml`.

## File structure

```
example_module/
├── drush.services.yml
└── src/Commands/
    └── ExampleCommands.php
```

---

## `drush.services.yml`

```yaml
services:
  example_module.commands:
    class: Drupal\example_module\Commands\ExampleCommands
    arguments:
      - '@entity_type.manager'
      - '@example_module.greeting_generator'
    tags:
      - { name: drush.command }
```

---

## Command class — `src/Commands/ExampleCommands.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Commands;

use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\example_module\Service\GreetingGenerator;
use Drush\Attributes as CLI;
use Drush\Commands\DrushCommands;
use Symfony\Component\Console\Input\InputOption;

/**
 * Drush commands for example_module.
 */
final class ExampleCommands extends DrushCommands {

  public function __construct(
    private readonly EntityTypeManagerInterface $entityTypeManager,
    private readonly GreetingGenerator $greetingGenerator,
  ) {
    parent::__construct();
  }

  /**
   * Print a greeting message.
   */
  #[CLI\Command(name: 'example:greet', aliases: ['exg'])]
  #[CLI\Argument(name: 'name', description: 'Name to greet.')]
  #[CLI\Option(name: 'shout', description: 'Output in uppercase.')]
  #[CLI\Usage(name: 'example:greet World', description: 'Greet World.')]
  #[CLI\Usage(name: 'example:greet World --shout', description: 'Greet World loudly.')]
  public function greet(string $name, array $options = ['shout' => FALSE]): void {
    $message = $this->greetingGenerator->generate($name);

    if ($options['shout']) {
      $message = mb_strtoupper($message);
    }

    $this->io()->success($message);
  }

  /**
   * Import articles by ID.
   */
  #[CLI\Command(name: 'example:import-articles', aliases: ['exia'])]
  #[CLI\Argument(name: 'ids', description: 'Comma-separated node IDs.')]
  #[CLI\Option(name: 'dry-run', description: 'Simulate without saving.')]
  public function importArticles(string $ids, array $options = ['dry-run' => FALSE]): void {
    $nid_list = array_map('intval', explode(',', $ids));
    $storage = $this->entityTypeManager->getStorage('node');
    $dry_run = (bool) $options['dry-run'];

    $progress = $this->io()->createProgressBar(count($nid_list));
    $progress->start();

    foreach ($nid_list as $nid) {
      $node = $storage->load($nid);
      if ($node === NULL) {
        $this->logger()->warning("Node $nid not found.");
        $progress->advance();
        continue;
      }

      if (!$dry_run) {
        $node->set('field_imported', TRUE);
        $node->save();
      }

      $progress->advance();
    }

    $progress->finish();
    $this->io()->newLine(2);
    $this->io()->success($dry_run
      ? 'Dry run complete.'
      : 'Imported ' . count($nid_list) . ' articles.');
  }

  /**
   * Delete old log entries.
   */
  #[CLI\Command(name: 'example:purge-logs')]
  #[CLI\Option(name: 'days', description: 'Delete logs older than N days.')]
  #[CLI\ValidateModulesEnabled(modules: ['example_module'])]
  public function purgeLogs(array $options = ['days' => 30]): void {
    $days = (int) $options['days'];
    $cutoff = \Drupal::time()->getRequestTime() - ($days * 86400);

    $deleted = \Drupal::database()->delete('example_module_log')
      ->condition('created', $cutoff, '<')
      ->execute();

    $this->io()->success("Deleted $deleted log entries older than $days days.");
  }

}
```

---

## Run commands

```bash
# Discover commands
ddev drush list | grep example

# Run greeting command
ddev drush example:greet "Drupal Developer"
ddev drush exg World --shout

# Import with dry run
ddev drush example:import-articles 1,2,3 --dry-run

# Purge logs
ddev drush example:purge-logs --days=7
```

---

## Input validation attribute

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Commands;

use Drush\Attributes as CLI;
use Drush\Commands\DrushCommands;

final class ValidatedCommands extends DrushCommands {

  #[CLI\Command(name: 'example:require-article')]
  #[CLI\Argument(name: 'nid', description: 'Article node ID.')]
  public function requireArticle(int $nid): void {
    $node = \Drupal::entityTypeManager()->getStorage('node')->load($nid);

    if ($node === NULL || $node->bundle() !== 'article') {
      throw new \InvalidArgumentException("Node $nid is not an article.");
    }

    $this->io()->writeln($node->label());
  }

}
```

---

## Rebuild Drush cache after adding commands

```bash
ddev drush cr
```

!!! note "Drush 12+ attributes"
    Drush 12 uses PHP 8 `#[CLI\Command]` attributes. Legacy `@command` annotations are deprecated.

!!! tip "Inject services"
    List constructor dependencies in `drush.services.yml` arguments. Do not use `\Drupal::` inside commands except for quick prototypes.

!!! warning "Batch and queue from Drush"
    Long operations should delegate to [Batch API](batch-api.md) or [Queue API](queue-api.md) rather than blocking the terminal.
