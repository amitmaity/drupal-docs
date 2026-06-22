# Mime

The **Mime** component detects MIME types, parses email messages, and maps extensions to content types. Drupal uses it in file validation, HTTP `Content-Type` headers, and the file upload pipeline.

## Core concepts

| Class | Purpose |
|-------|---------|
| `MimeTypes` | Extension ↔ MIME type mapping |
| `FileinfoMimeTypeGuesser` | Detect type from file contents |
| `File` | Value object with path and guessed MIME |

```php
<?php

declare(strict_types=1);

use Symfony\Component\Mime\MimeTypes;

$mimeTypes = new MimeTypes();
$mimeTypes->getMimeTypes('pdf');    // ['application/pdf']
$mimeTypes->getExtensions('image/jpeg'); // ['jpeg', 'jpg', 'jpe']
```

## How Drupal uses Mime

### File upload validation

Drupal validates uploaded files against allowed extensions and MIME types in `FileValidator`:

```php
<?php

declare(strict_types=1);

use Drupal\file\Validation\FileValidator;
use Drupal\file\Validation\Constraint\FileExtensionConstraint;
use Symfony\Component\Mime\MimeTypes;

// Core uses symfony/mime internally when validating file entities.
```

### `file.mime_type.guesser` service

```php
<?php

declare(strict_types=1);

/** @var \Symfony\Component\Mime\MimeTypeGuesserInterface $guesser */
$guesser = \Drupal::service('file.mime_type.guesser');
$mime = $guesser->guessMimeType('/path/to/file.png');
// image/png
```

### HTTP responses

Set accurate `Content-Type` when serving files:

```php
<?php

declare(strict_types=1);

use Symfony\Component\HttpFoundation\BinaryFileResponse;
use Symfony\Component\Mime\MimeTypes;

$path = '/var/www/files/document.pdf';
$mimeTypes = new MimeTypes();
$extensions = $mimeTypes->getExtensions('application/pdf');

$response = new BinaryFileResponse($path);
$response->headers->set('Content-Type', 'application/pdf');
```

## Extension vs content detection

!!! warning "Never trust client-provided MIME"
    Browsers send `Content-Type` on uploads — always verify with content-based guessing (`finfo` / `FileinfoMimeTypeGuesser`).

=== "Extension lookup"

    ```php
    <?php

    declare(strict_types=1);

    use Symfony\Component\Mime\MimeTypes;

    $mimeTypes = MimeTypes::getDefault();
    $mimes = $mimeTypes->getMimeTypes('svg');
    // ['image/svg+xml', 'image/svg']
    ```

=== "Content-based guess"

    ```php
    <?php

    declare(strict_types=1);

    use Symfony\Component\Mime\FileinfoMimeTypeGuesser;

    $guesser = new FileinfoMimeTypeGuesser();
    $mime = $guesser->guessMimeType('/tmp/upload.bin');
    ```

## Custom module example: strict upload validator

=== "Constraint validator"

    ```php
    <?php

    declare(strict_types=1);

    namespace Drupal\my_module\FileValidation;

    use Drupal\file\Validation\Constraint\BaseConstraintValidator;
    use Symfony\Component\Mime\FileinfoMimeTypeGuesser;
    use Symfony\Component\Validator\Constraint;

    /**
     * Validates allowed MIME types by file content, not extension.
     *
     * @internal
     */
    final class AllowedMimeTypeValidator extends BaseConstraintValidator {

      public function validate(mixed $value, Constraint $constraint): void {
        assert($constraint instanceof AllowedMimeTypeConstraint);

        if ($value === NULL) {
          return;
        }

        $guesser = new FileinfoMimeTypeGuesser();
        $path = $value->getRealPath();
        $detected = $guesser->guessMimeType($path);

        if ($detected === NULL || !in_array($detected, $constraint->allowedTypes, TRUE)) {
          $this->context->addViolation($constraint->message, [
            '%type' => $detected ?? 'unknown',
            '%allowed' => implode(', ', $constraint->allowedTypes),
          ]);
        }
      }

    }
    ```

=== "Constraint"

    ```php
    <?php

    declare(strict_types=1);

    namespace Drupal\my_module\FileValidation;

    use Symfony\Component\Validator\Constraint;

    /**
     * @Constraint(
     *   id = "MyModuleAllowedMimeType",
     *   label = @Translation("Allowed MIME type by content", context = "Validation"),
     * )
     */
    final class AllowedMimeTypeConstraint extends Constraint {

      /** @var list<string> */
      public array $allowedTypes = [
        'image/jpeg',
        'image/png',
        'image/webp',
      ];

      public string $message = 'File type %type is not allowed. Allowed: %allowed.';

    }
    ```

=== "Form field"

    ```php
    <?php

    declare(strict_types=1);

    namespace Drupal\my_module\Form;

    use Drupal\Core\Form\FormBase;
    use Drupal\Core\Form\FormStateInterface;
    use Drupal\file\Validation\Constraint\FileExtensionConstraint;
    use Drupal\my_module\FileValidation\AllowedMimeTypeConstraint;

    final class ImageUploadForm extends FormBase {

      public function buildForm(array $form, FormStateInterface $form_state): array {
        $form['image'] = [
          '#type' => 'managed_file',
          '#title' => $this->t('Product image'),
          '#upload_location' => 'public://product-images/',
          '#upload_validators' => [
            'FileExtension' => ['extensions' => 'png jpg jpeg webp'],
            'MyModuleAllowedMimeType' => [],
          ],
        ];

        $form['actions']['submit'] = ['#type' => 'submit', '#value' => $this->t('Upload')];
        return $form;
      }

      public function submitForm(array &$form, FormStateInterface $form_state): void {
        $this->messenger()->addStatus($this->t('Image uploaded.'));
      }

      public function getFormId(): string {
        return 'my_module_image_upload';
      }

    }
    ```

## API response Content-Type helper

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Service;

use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Mime\MimeTypes;

final class MimeResponseFactory {

  public function create(string $content, string $extension): Response {
    $mimeTypes = MimeTypes::getDefault();
    $types = $mimeTypes->getMimeTypes($extension);
    $contentType = $types[0] ?? 'application/octet-stream';

    return new Response($content, 200, ['Content-Type' => $contentType]);
  }

}
```

## Related reading

- [HttpFoundation](http-foundation.md) — file uploads
- [Filesystem](filesystem.md) — storing uploaded files
- [Validator](validator.md) — file constraints
