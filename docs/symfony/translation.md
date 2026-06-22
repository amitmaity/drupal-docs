# Translation

The **Translation** component manages message catalogs, locales, and pluralization. Drupal wraps it with `t()`, `TranslatableMarkup`, and the `string_translation` service for multilingual sites.

## Core concepts

| Symfony | Drupal |
|---------|--------|
| `TranslatorInterface` | `string_translation` service |
| Message ID + parameters | `TranslatableMarkup` |
| Locale | Language code (`en`, `fr`, `de`) |
| Domain | Context (`context` in `t()`) |

```php
<?php

declare(strict_types=1);

use Symfony\Component\Translation\Translator;

$translator = new Translator('en');
$translator->addResource('array', ['Hello %name%' => 'Bonjour %name%'], 'fr');
echo $translator->trans('Hello %name%', ['%name%' => 'World'], NULL, 'fr');
// Bonjour World
```

## How Drupal uses Translation

### The `t()` function and TranslatableMarkup

Strings are wrapped at definition time and translated at render time:

```php
<?php

declare(strict_types=1);

use Drupal\Core\StringTranslation\TranslatableMarkup;

// In controllers, forms, plugins — use $this->t()
$message = $this->t('Product @name saved.', ['@name' => $product->label()]);

// Equivalent object (deferred translation)
$markup = new TranslatableMarkup('Product @name saved.', ['@name' => $product->label()]);
```

### Placeholder safety

| Placeholder | Escaping |
|-------------|----------|
| `@key` | HTML-escaped |
| `%key` | Escapes and wraps in `<em>` |
| `:key` | Not escaped (use only for trusted HTML) |

```php
<?php

declare(strict_types=1);

// Safe — user input is escaped
$this->t('Hello @name', ['@name' => $userInput]);

// Dangerous — only for trusted markup
$this->t('Visit :link', [':link' => '<a href="/admin">admin</a>']);
```

### Interface translation

Custom module strings are extracted to `*.po` files:

```po
# my_module.po
msgid "Product list"
msgstr "Liste des produits"

msgid "Imported @count products."
msgid_plural "Imported @count products."
msgstr[0] "Produit @count importé."
msgstr[1] "@count produits importés."
```

Enable locale module and import translations at `/admin/config/regional/translate`.

### Config translation

Configuration entities can be translated per language (`config_translation` module).

## StringTranslationTrait

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Service;

use Drupal\Core\StringTranslation\StringTranslationTrait;

final class ProductNotifier {

  use StringTranslationTrait;

  public function getSubject(string $productName): string {
    return (string) $this->t('Order confirmation: @product', [
      '@product' => $productName,
    ]);
  }

}
```

!!! note "Inject translation in services"
    Services should use `StringTranslationTrait` and call `$this->t()`, or inject `TranslationInterface` for testability.

## Pluralization

```php
<?php

declare(strict_types=1);

$count = 5;
$message = \Drupal::translation()->formatPlural(
  $count,
  'One item in cart.',
  '@count items in cart.',
  ['@count' => $count],
);
```

In forms and controllers:

```php
<?php

declare(strict_types=1);

$this->formatPlural($deleted, 'Deleted 1 product.', 'Deleted @count products.', [
  '@count' => $deleted,
]);
```

## Custom module example: translatable REST errors

=== "Error response service"

    ```php
    <?php

    declare(strict_types=1);

    namespace Drupal\my_module\Service;

    use Drupal\Core\StringTranslation\TranslatableMarkup;
    use Drupal\Core\StringTranslation\TranslationInterface;

    final class ApiMessageService {

      public function __construct(
        private readonly TranslationInterface $translation,
      ) {}

      public function notFound(string $resource): TranslatableMarkup {
        return new TranslatableMarkup('@type not found.', [
          '@type' => $resource,
        ], [], $this->translation);
      }

      public function validationFailed(array $fields): TranslatableMarkup {
        return new TranslatableMarkup('Validation failed for: @fields', [
          '@fields' => implode(', ', $fields),
        ], [], $this->translation);
      }

      public function toArray(TranslatableMarkup $message): array {
        return [
          'message' => (string) $message,
          'langcode' => $this->translation->getDefaultLangcode(),
        ];
      }

    }
    ```

=== "Controller usage"

    ```php
    <?php

    declare(strict_types=1);

    namespace Drupal\my_module\Controller;

    use Drupal\Core\Controller\ControllerBase;
    use Drupal\my_module\Service\ApiMessageService;
    use Symfony\Component\HttpFoundation\JsonResponse;

    final class ProductApiController extends ControllerBase {

      public function __construct(
        private readonly ApiMessageService $messages,
      ) {}

      public static function create($container): static {
        return new static($container->get('my_module.api_messages'));
      }

      public function get(int $id): JsonResponse {
        $product = $this->entityTypeManager()->getStorage('product')->load($id);

        if ($product === NULL) {
          $error = $this->messages->notFound('product');
          return new JsonResponse($this->messages->toArray($error), 404);
        }

        return new JsonResponse(['id' => $product->id(), 'title' => $product->label()]);
      }

    }
    ```

=== "services.yml"

    ```yaml
    my_module.api_messages:
      class: Drupal\my_module\Service\ApiMessageService
      arguments: ['@string_translation']
    ```

## Custom .po file in module

```yaml
# my_module.info.yml
'interface translation project': my_module
'interface translation server pattern': modules/custom/my_module/translations/my_module-%language.po
```

Place `my_module-fr.po` in `translations/` for French overrides shipped with the module.

## Related reading

- [Controllers](controllers.md) — `$this->t()` in controllers
- [Headless Drupal](../headless/index.md) — multilingual API considerations
