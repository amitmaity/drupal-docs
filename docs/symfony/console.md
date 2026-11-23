# Console

The **Console** component powers CLI applications with commands, arguments, options, and styled output. **Drush** and Drupal's `drupal` script both build on Symfony Console.

## Core concepts

| Piece | Purpose |
|-------|---------|
| `Application` | Registers and runs commands |
| `Command` | A single CLI task with `configure()` and `execute()` |
| `InputInterface` | Arguments and options from argv |
| `OutputInterface` | stdout/stderr with verbosity levels |
| `Style` | `SymfonyStyle` for tables, progress bars, prompts |

```bash
# Drush is a Symfony Console application wrapping Drupal
drush status
drush cache:rebuild
drush entity:delete node 1
```

## How Drupal uses Console

### Drush commands

Drush 12+ uses annotated commands or PHP attributes. Drupal bootstrap is triggered via `@bootstrap` or `#[CLI\Bootstrap]`:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Commands;

use Drush\Attributes as CLI;
use Drush\Boot\DrupalBootLevels;
use Drush\Commands\DrushCommands;
use Drupal\Core\Entity\EntityTypeManagerInterface;

final class ProductCommands extends DrushCommands {

  public function __construct(
    private readonly EntityTypeManagerInterface $entityTypeManager,
  ) {
    parent::__construct();
  }

  #[CLI\Command(name: 'my-module:product:sync', aliases: ['mps'])]
  #[CLI\Argument(name: 'source', description: 'CSV file path')]
  #[CLI\Bootstrap(level: DrupalBootLevels::FULL)]
  public function sync(string $source): void {
    if (!is_readable($source)) {
      throw new \InvalidArgumentException("Cannot read: {$source}");
    }

    $io = $this->io();
    $io->title('Product sync');

    $handle = fopen($source, 'r');
    $header = fgetcsv($handle);
    $count = 0;

    while (($row = fgetcsv($handle)) !== FALSE) {
      $data = array_combine($header, $row);
      $this->importRow($data);
      $count++;
    }
    fclose($handle);

    $io->success("Imported {$count} products.");
  }

  private function importRow(array $data): void {
    $storage = $this->entityTypeManager->getStorage('product');
    $product = $storage->create([
      'sku' => $data['sku'],
      'title' => $data['title'],
    ]);
    $product->save();
  }

}
```

Register via `drush.services.yml`:

```yaml
# drush.services.yml
services:
  my_module.commands.product:
    class: Drupal\my_module\Commands\ProductCommands
    arguments: ['@entity_type.manager']
    tags:
      - { name: drush.command }
```

### Standalone Symfony Console in custom modules

For scripts outside Drush, bootstrap Drupal and run a command:

```php
<?php

declare(strict_types=1);

// scripts/my-cli.php (simplified)
use Drupal\Core\DrupalKernel;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Console\Application;

$autoloader = require __DIR__ . '/../vendor/autoload.php';
$kernel = DrupalKernel::createFromRequest(Request::createFromGlobals(), $autoloader, 'prod');
$kernel->boot();

$application = new Application('My Module CLI');
// Application::add() is deprecated since Symfony 7.4; use addCommand().
$application->addCommand(new \Drupal\my_module\Command\CleanupCommand());
$application->run();
```

## Command structure

=== "Drush command (recommended)"

    ```php
    <?php

    declare(strict_types=1);

    namespace Drupal\my_module\Commands;

    use Drush\Attributes as CLI;
    use Drush\Boot\DrupalBootLevels;
    use Drush\Commands\DrushCommands;

    final class CacheCommands extends DrushCommands {

      #[CLI\Command(name: 'my-module:cache:warm')]
      #[CLI\Option(name: 'tag', description: 'Cache tag to warm')]
      #[CLI\Bootstrap(level: DrupalBootLevels::FULL)]
      public function warm(array $options = ['tag' => 'my_module:products']): void {
        $tag = $options['tag'];
        $this->logger()->info('Warming cache for tag: {tag}', ['tag' => $tag]);
        // Warm logic…
        $this->io()->success('Cache warmed.');
      }

    }
    ```

=== "Pure Symfony Command"

    ```php
    <?php

    declare(strict_types=1);

    namespace Drupal\my_module\Command;

    use Symfony\Component\Console\Attribute\AsCommand;
    use Symfony\Component\Console\Command\Command;
    use Symfony\Component\Console\Input\InputInterface;
    use Symfony\Component\Console\Input\InputOption;
    use Symfony\Component\Console\Output\OutputInterface;
    use Symfony\Component\Console\Style\SymfonyStyle;

    #[AsCommand(
      name: 'my-module:cleanup',
      description: 'Remove orphaned product records.',
    )]
    final class CleanupCommand extends Command {

      protected function configure(): void {
        $this->addOption('dry-run', NULL, InputOption::VALUE_NONE, 'Simulate only');
      }

      protected function execute(InputInterface $input, OutputInterface $output): int {
        $io = new SymfonyStyle($input, $output);
        $dryRun = (bool) $input->getOption('dry-run');

        $io->note($dryRun ? 'Dry run mode' : 'Live mode');
        // Cleanup via \Drupal::service()…
        $io->success('Done.');

        return Command::SUCCESS;
      }

    }
    ```

## Input/output patterns

```php
<?php

declare(strict_types=1);

// Inside a Drush command
$io = $this->io();

// Table output
$io->table(['SKU', 'Title'], [
  ['W-001', 'Widget'],
  ['W-002', 'Gadget'],
]);

// Progress bar
$items = range(1, 100);
$io->progressStart(count($items));
foreach ($items as $item) {
  // process…
  $io->progressAdvance();
}
$io->progressFinish();

// Confirm destructive action
if (!$io->confirm('Delete all orphaned products?', FALSE)) {
  return;
}
```

!!! warning "Bootstrap level"
    Commands that need the database or entity API must use `#[CLI\Bootstrap(level: DrupalBootLevels::FULL)]` (`use Drush\Boot\DrupalBootLevels;`). Configuration-only commands can use `DrupalBootLevels::CONFIGURATION` or `DrupalBootLevels::NONE` for speed.

## Custom module example: nightly report command

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Commands;

use Drupal\Core\Database\Connection;
use Drush\Attributes as CLI;
use Drush\Boot\DrupalBootLevels;
use Drush\Commands\DrushCommands;

final class ReportCommands extends DrushCommands {

  public function __construct(
    private readonly Connection $database,
  ) {
    parent::__construct();
  }

  #[CLI\Command(name: 'my-module:report:daily')]
  #[CLI\Bootstrap(level: DrupalBootLevels::FULL)]
  public function dailyReport(): void {
    $io = $this->io();
    $io->title('Daily product report');

    $count = (int) $this->database->select('product', 'p')
      ->countQuery()
      ->execute()
      ->fetchField();

    $createdToday = (int) $this->database->select('product', 'p')
      ->condition('created', strtotime('today'), '>=')
      ->countQuery()
      ->execute()
      ->fetchField();

    $io->definitionList(
      ['Total products' => (string) $count],
      ['Created today' => (string) $createdToday],
    );

    $this->logger()->info('Daily report: @total total, @new new', [
      '@total' => $count,
      '@new' => $createdToday,
    ]);
  }

}
```

```yaml
# drush.services.yml
services:
  my_module.commands.report:
    class: Drupal\my_module\Commands\ReportCommands
    arguments: ['@database']
    tags:
      - { name: drush.command }
```

```bash
# Run manually or from cron
drush my-module:report:daily
```

## Related reading

- [Getting Started: Drush](../getting-started/drush.md)
- [Custom Modules: Drush Commands](../custom-coding/drush-commands.md)
- [Cron API](../custom-coding/cron-api.md)
