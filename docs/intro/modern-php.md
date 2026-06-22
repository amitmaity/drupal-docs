# Modern PHP Standards

Drupal 11 embraces modern PHP 8.3+ features. Writing idiomatic PHP makes your custom modules cleaner, safer, and easier to test.

## Strict typing

Always declare strict types in new files:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Service;

final class PriceCalculator {

  public function calculate(float $basePrice, float $taxRate): float {
    return round($basePrice * (1 + $taxRate), 2);
  }
}
```

## Typed properties

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Entity;

use Drupal\Core\Entity\ContentEntityBase;

final class Product extends ContentEntityBase {

  // Typed properties on the entity wrapper
  public function getSku(): string {
    return $this->get('sku')->value;
  }

  public function setSku(string $sku): static {
    $this->set('sku', $sku);
    return $this;
  }

  public function getPrice(): float {
    return (float) $this->get('price')->value;
  }
}
```

## Constructor property promotion

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Service;

use Drupal\Core\Entity\EntityTypeManagerInterface;
use Psr\Log\LoggerInterface;

final class ProductManager {

  public function __construct(
    private readonly EntityTypeManagerInterface $entityTypeManager,
    private readonly LoggerInterface $logger,
  ) {}

  public function getActiveProducts(): array {
    return $this->entityTypeManager
      ->getStorage('product')
      ->loadByProperties(['status' => TRUE]);
  }
}
```

## PHP Attributes

Drupal 11 uses PHP 8 attributes extensively:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Plugin\Block;

use Drupal\Core\Block\Attribute\Block;
use Drupal\Core\Block\BlockBase;
use Drupal\Core\StringTranslation\TranslatableMarkup;

#[Block(
  id: "featured_products",
  admin_label: new TranslatableMarkup("Featured Products"),
  category: new TranslatableMarkup("Commerce"),
)]
final class FeaturedProductsBlock extends BlockBase {

  public function build(): array {
    return [
      '#markup' => '<p>Featured products here.</p>',
      '#cache' => ['max-age' => 3600],
    ];
  }
}
```

### Common Drupal attributes

| Attribute | Purpose |
|-----------|---------|
| `#[Block]` | Block plugin |
| `#[FieldFormatter]` | Field formatter plugin |
| `#[FieldWidget]` | Field widget plugin |
| `#[Hook]` | OOP hook implementation |
| `#[QueueWorker]` | Queue worker plugin |
| `#[RestResource]` | REST resource plugin |

## Enums

Use backed enums for fixed value sets:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Enum;

enum OrderStatus: string {
  case Pending = 'pending';
  case Processing = 'processing';
  case Shipped = 'shipped';
  case Delivered = 'delivered';
  case Cancelled = 'cancelled';

  public function label(): string {
    return match ($this) {
      self::Pending => 'Pending',
      self::Processing => 'Processing',
      self::Shipped => 'Shipped',
      self::Delivered => 'Delivered',
      self::Cancelled => 'Cancelled',
    };
  }

  public function isFinal(): bool {
    return in_array($this, [self::Delivered, self::Cancelled], TRUE);
  }
}
```

```php
// Usage in a service
$status = OrderStatus::from($entity->get('status')->value);
if ($status->isFinal()) {
  throw new \LogicException('Cannot modify a final order.');
}
```

## Readonly classes and properties

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\ValueObject;

readonly class Address {
  public function __construct(
    public string $street,
    public string $city,
    public string $postalCode,
    public string $country,
  ) {}

  public function toString(): string {
    return "{$this->street}, {$this->city} {$this->postalCode}, {$this->country}";
  }
}
```

## Match expressions

Prefer `match` over `switch`:

```php
public function getCacheMaxAge(string $contentType): int {
  return match ($contentType) {
    'breaking_news' => 60,
    'article'       => 3600,
    'page'          => 86400,
    default         => 300,
  };
}
```

## Named arguments

```php
$node = Node::create([
  'type' => 'article',
  'title' => 'Headless Drupal',
  'uid' => 1,
  'status' => NodeInterface::PUBLISHED,
  'promote' => FALSE,
]);
```

## Union and intersection types

```php
public function resolveEntity(string|int $id): NodeInterface|Null {
  return $this->entityTypeManager
    ->getStorage('node')
    ->load($id);
}
```

## Final classes

Mark classes `final` unless designed for extension:

```php
final class ApiResponseBuilder {
  // Cannot be extended — intentional
}
```

!!! tip "Coding standards"
    Follow [Drupal PHP coding standards](https://www.drupal.org/docs/develop/standards/coding-standards) and run `phpcs` with the Drupal standard on your custom modules.
