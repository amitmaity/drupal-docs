# Serializer

The **Serializer** component converts PHP objects to formats (JSON, XML) and back using **normalizers** and **encoders**. Drupal's JSON:API, REST module, and entity serialization all build on this stack with Drupal-specific normalizers.

## Core concepts

```mermaid
flowchart LR
    Object[PHP Object] --> Normalizer[Normalizer]
    Normalizer --> Array["Array / scalar data"]
    Array --> Encoder[Encoder]
    Encoder --> JSON["JSON string"]
```

| Piece | Role |
|-------|------|
| `Serializer` | Orchestrates normalize → encode and decode → denormalize |
| Normalizer | Object ↔ array (e.g. `Node` → `['title' => …]`) |
| Encoder | Array ↔ string (e.g. `JsonEncoder`) |
| Context | Options passed through the chain (`['include' => …]`) |

## How Drupal uses the Serializer

### The `serializer` service

Drupal registers a `Serializer` with tagged normalizers and encoders:

```php
<?php

declare(strict_types=1);

/** @var \Symfony\Component\Serializer\SerializerInterface $serializer */
$serializer = \Drupal::service('serializer');

$node = \Drupal::entityTypeManager()->getStorage('node')->load(1);
$json = $serializer->serialize($node, 'json', ['plugin_id' => 'entity']);
```

### JSON:API normalizer chain

JSON:API does not call `serialize()` on entities directly. It uses resource types and a dedicated normalizer pipeline in the `jsonapi` module. The Symfony Serializer still powers encoding/decoding underneath.

Key Drupal normalizers (simplified):

| Normalizer | Handles |
|------------|---------|
| `ContentEntityNormalizer` | Fieldable entities |
| `EntityReferenceFieldItemNormalizer` | Entity reference fields |
| `TimestampItemNormalizer` | Date/timestamp fields |
| `JsonEncoder` | Final JSON output |

### REST module

REST resources return `ResourceResponse` objects that are serialized using configured formats (`json`, `xml`).

## Serialization example

=== "Serialize entity to JSON"

    ```php
    <?php

    declare(strict_types=1);

    use Drupal\node\Entity\Node;

    $node = Node::load(1);
    $serializer = \Drupal::service('serializer');

    $json = $serializer->serialize($node, 'json', [
      'account' => \Drupal::currentUser(),
    ]);

    header('Content-Type: application/json');
    echo $json;
    ```

=== "Deserialize JSON to array"

    ```php
    <?php

    declare(strict_types=1);

    $serializer = \Drupal::service('serializer');
    $data = $serializer->deserialize(
      '{"title":"Imported article","type":"article"}',
      'array',
      'json',
    );
    ```

## Custom normalizers

Implement `NormalizerInterface` and tag the service:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Normalizer;

use Drupal\my_module\Entity\Product;
use Symfony\Component\Serializer\Normalizer\NormalizerAwareInterface;
use Symfony\Component\Serializer\Normalizer\NormalizerAwareTrait;
use Symfony\Component\Serializer\Normalizer\NormalizerInterface;

final class ProductNormalizer implements NormalizerInterface, NormalizerAwareInterface {

  use NormalizerAwareTrait;

  public function normalize(
    mixed $object,
    ?string $format = NULL,
    array $context = [],
  ): array|string|int|float|bool|\ArrayObject|null {
    assert($object instanceof Product);

    return [
      'id' => $object->id(),
      'sku' => $object->get('sku')->value,
      'title' => $object->label(),
      'price' => [
        'amount' => $object->get('price')->value,
        'currency' => $object->get('price')->currency_code ?? 'USD',
      ],
      'links' => [
        'self' => '/api/products/' . $object->id(),
      ],
    ];
  }

  public function supportsNormalization(mixed $data, ?string $format = NULL, array $context = []): bool {
    return $data instanceof Product;
  }

  public function getSupportedTypes(?string $format): array {
    return [Product::class => TRUE];
  }

}
```

```yaml
# my_module.services.yml
services:
  my_module.normalizer.product:
    class: Drupal\my_module\Normalizer\ProductNormalizer
    tags:
      - { name: normalizer, priority: 10 }
```

!!! note "Priority matters"
    Higher-priority normalizers are checked first. Set priority above generic entity normalizers if your class is more specific.

## JSON:API integration

To expose a custom entity via JSON:API, define an entity type with proper handlers — JSON:API auto-discovers resource types. For field-level customization, use normalizers or JSON:API's `ResourceType` alter hooks.

=== "Entity type annotation (excerpt)"

    ```php
    <?php

    declare(strict_types=1);

    /**
     * @ContentEntityType(
     *   id = "product",
     *   label = @Translation("Product"),
     *   base_table = "product",
     *   entity_keys = {
     *     "id" = "id",
     *     "label" = "title",
     *     "uuid" = "uuid",
     *   },
     *   handlers = {
     *     "view_builder" = "Drupal\Core\Entity\EntityViewBuilder",
     *     "list_builder" = "Drupal\my_module\ProductListBuilder",
     *     "form" = {
     *       "default" = "Drupal\my_module\Form\ProductForm",
     *     },
     *     "access" = "Drupal\my_module\ProductAccessControlHandler",
     *   },
     *   links = {
     *     "canonical" = "/product/{product}",
     *   },
     * )
     */
    ```

=== "JSON:API response (after enabling jsonapi_extras optional)"

    ```bash
    curl -s -H "Accept: application/vnd.api+json" \
      "https://example.com/jsonapi/product/product" | jq .
    ```

For computed fields not stored in the database, add a normalizer or implement `JsonapiResourceBase` patterns in contrib `jsonapi_resources`.

## Custom module example: export service

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Service;

use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\my_module\Entity\Product;
use Symfony\Component\Serializer\SerializerInterface;

final class ProductExportService {

  public function __construct(
    private readonly EntityTypeManagerInterface $entityTypeManager,
    private readonly SerializerInterface $serializer,
  ) {}

  /**
   * Export all products as a JSON array string.
   */
  public function exportJson(): string {
    $storage = $this->entityTypeManager->getStorage('product');
    $products = $storage->loadMultiple();

    $rows = array_map(
      fn (Product $product) => json_decode(
        $this->serializer->serialize($product, 'json'),
        TRUE,
        512,
        JSON_THROW_ON_ERROR,
      ),
      $products,
    );

    return json_encode(['products' => array_values($rows)], JSON_THROW_ON_ERROR | JSON_PRETTY_PRINT);
  }

}
```

```yaml
# my_module.services.yml
services:
  my_module.product_export:
    class: Drupal\my_module\Service\ProductExportService
    arguments: ['@entity_type.manager', '@serializer']
```

## Related reading

- [Headless: JSON:API](../headless/jsonapi.md)
- [Headless: REST API](../headless/rest-api.md)
- [Validator](validator.md) — validate before deserialize
