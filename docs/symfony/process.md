# Process

The **Process** component runs external system commands with timeout, idle, and environment control. Drupal uses it for ImageMagick, FFmpeg, wkhtmltopdf, and other CLI tools.

## Core concepts

```php
<?php

declare(strict_types=1);

use Symfony\Component\Process\Process;

$process = new Process(['convert', 'input.png', 'output.jpg']);
$process->setTimeout(120);
$process->run();

if (!$process->isSuccessful()) {
  throw new \RuntimeException($process->getErrorOutput());
}

echo $process->getOutput();
```

| Method | Purpose |
|--------|---------|
| `run()` | Blocking execution |
| `mustRun()` | Run or throw `ProcessFailedException` |
| `start()` / `wait()` | Async with callback |
| `setTimeout()` | Kill after N seconds |
| `setIdleTimeout()` | Kill if no output for N seconds |

## How Drupal uses Process

### Image toolkit

ImageAPI Optimize and contrib modules spawn ImageMagick/OptiPNG via Process.

### Cron and queue workers

Long-running external sync jobs often wrap shell commands:

```php
<?php

declare(strict_types=1);

// Conceptual pattern in core/contrib
$process = Process::fromShellCommandline('drush queue:run my_queue');
$process->setTimeout(300);
$process->mustRun();
```

### Settings.php trusted host patterns

Not Process-related, but operational security when spawning shells — always validate arguments and avoid `fromShellCommandline()` with user input.

!!! warning "Shell injection"
    Never pass unsanitized user input to `fromShellCommandline()`. Use array argument form: `new Process(['binary', $safeArg])`.

## Safe process patterns

=== "Array arguments (preferred)"

    ```php
    <?php

    declare(strict_types=1);

    use Symfony\Component\Process\Process;

    $process = new Process([
      '/usr/bin/convert',
      $sourcePath,
      '-resize', '800x600',
      $destinationPath,
    ]);
    $process->setTimeout(60);
    $process->mustRun();
    ```

=== "Environment variables"

    ```php
    <?php

    declare(strict_types=1);

    $process = new Process(['php', 'bin/console', 'cache:clear']);
    $process->setEnv(['APP_ENV' => 'prod']);
    $process->run();
    ```

## Custom module example: PDF generator service

=== "Service"

    ```php
    <?php

    declare(strict_types=1);

    namespace Drupal\my_module\Service;

    use Drupal\Core\File\FileExists;
    use Drupal\Core\File\FileSystemInterface;
    use Drupal\Core\Logger\LoggerChannelInterface;
    use Symfony\Component\Process\Exception\ProcessFailedException;
    use Symfony\Component\Process\Process;

    final class PdfGenerator {

      public function __construct(
        private readonly FileSystemInterface $fileSystem,
        private readonly LoggerChannelInterface $logger,
        private readonly string $wkhtmltopdfPath = '/usr/bin/wkhtmltopdf',
      ) {}

      public function generateFromHtml(string $html, string $destinationUri): string {
        // External binaries cannot read stream wrapper URIs such as
        // temporary://, so pass real filesystem paths.
        $base = $this->fileSystem->getTempDirectory() . '/' . uniqid('pdf_', TRUE);
        $tempHtml = $base . '.html';
        $tempPdf = $base . '.pdf';

        file_put_contents($tempHtml, $html);

        $process = new Process([
          $this->wkhtmltopdfPath,
          '--quiet',
          $tempHtml,
          $tempPdf,
        ]);
        $process->setTimeout(120);

        try {
          $process->mustRun();
        }
        catch (ProcessFailedException $e) {
          $this->logger->error('PDF generation failed: @error', [
            '@error' => $process->getErrorOutput(),
          ]);
          throw $e;
        }
        finally {
          @unlink($tempHtml);
        }

        $this->fileSystem->move($tempPdf, $destinationUri, FileExists::Replace);

        return $destinationUri;
      }

    }
    ```

=== "services.yml"

    ```yaml
    my_module.pdf_generator:
      class: Drupal\my_module\Service\PdfGenerator
      arguments:
        - '@file_system'
        - '@logger.channel.my_module'
        - '%my_module.wkhtmltopdf_path%'

    parameters:
      my_module.wkhtmltopdf_path: '/usr/bin/wkhtmltopdf'
    ```

=== "Controller"

    ```php
    <?php

    declare(strict_types=1);

    namespace Drupal\my_module\Controller;

    use Drupal\Core\Controller\ControllerBase;
    use Drupal\my_module\Entity\Product;
    use Drupal\my_module\Service\PdfGenerator;
    use Symfony\Component\HttpFoundation\BinaryFileResponse;

    final class ProductPdfController extends ControllerBase {

      public function __construct(
        private readonly PdfGenerator $pdfGenerator,
      ) {}

      public static function create($container): static {
        return new static($container->get('my_module.pdf_generator'));
      }

      public function download(Product $product): BinaryFileResponse {
        $html = $this->renderProductHtml($product);
        $uri = 'temporary://product-' . $product->id() . '.pdf';
        $path = $this->pdfGenerator->generateFromHtml($html, $uri);

        return new BinaryFileResponse($path, 200, [
          'Content-Type' => 'application/pdf',
          'Content-Disposition' => 'attachment; filename="product-' . $product->id() . '.pdf"',
        ]);
      }

      private function renderProductHtml(Product $product): string {
        return '<html><body><h1>' . htmlspecialchars($product->label()) . '</h1></body></html>';
      }

    }
    ```

## Async processes

For non-blocking work with progress callbacks:

```php
<?php

declare(strict_types=1);

$process = new Process(['long-running-script.sh']);
$process->start();

while ($process->isRunning()) {
  echo $process->getIncrementalOutput();
  usleep(100_000);
}

$process->wait();
```

## Related reading

- [Filesystem](filesystem.md) — temp files and moves
- [Queue API](../custom-coding/queue-api.md) — offload heavy CLI work
