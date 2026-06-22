# What Changed in Drupal 11

Drupal 11 is a major evolution of the platform introduced in June 2024. It removes legacy baggage while embracing modern PHP and Symfony patterns.

## Key changes

### PHP 8.3+ required

Drupal 11 requires **PHP 8.3 or later**. This unlocks:

- Typed class constants
- `#[\Override]` attribute
- `json_validate()` native function
- Improved performance via JIT

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Service;

final class ReportGenerator {

  private const string DEFAULT_FORMAT = 'json';

  public function generate(array $data, ?string $format = null): string {
    $format ??= self::DEFAULT_FORMAT;
    return match ($format) {
      'json' => json_encode($data, JSON_THROW_ON_ERROR),
      'xml'  => $this->toXml($data),
      default => throw new \InvalidArgumentException("Unknown format: $format"),
    };
  }
}
```

### Symfony 7 components

Drupal 11 ships with **Symfony 7** (up from Symfony 6 in Drupal 10). Major component updates affect routing, serialization, and the HTTP layer.

| Component | Drupal 10 | Drupal 11 |
|-----------|-----------|-----------|
| HttpKernel | 6.x | 7.x |
| Routing | 6.x | 7.x |
| Serializer | 6.x | 7.x |
| DependencyInjection | 6.x | 7.x |
| Validator | 6.x | 7.x |

### OOP Hooks (Drupal 11.1+)

Hooks can now be implemented as class methods with the `#[Hook]` attribute:

=== "OOP Hook (11.1+)"

    ```php
    <?php

    namespace Drupal\my_module\Hook;

    use Drupal\Core\Hook\Attribute\Hook;
    use Drupal\Core\Entity\EntityInterface;

    final class MyModuleEntityHooks {

      #[Hook('entity_insert')]
      public function entityInsert(EntityInterface $entity): void {
        \Drupal::logger('my_module')->info('Entity @type @id created.', [
          '@type' => $entity->getEntityTypeId(),
          '@id' => $entity->id(),
        ]);
      }
    }
    ```

=== "Legacy Procedural"

    ```php
    <?php

    /**
     * Implements hook_entity_insert().
     */
    function my_module_entity_insert(EntityInterface $entity): void {
      \Drupal::logger('my_module')->info('Entity @type @id created.', [
        '@type' => $entity->getEntityTypeId(),
        '@id' => $entity->id(),
      ]);
    }
    ```

!!! note
    Procedural hooks remain supported until at least Drupal 13. Use OOP hooks for all new code on 11.1+.

### Removed modules and themes

These were removed from core (use contrib replacements):

| Removed | Replacement |
|---------|-------------|
| `book` | [Book](https://www.drupal.org/project/book) contrib |
| `forum` | [Forum](https://www.drupal.org/project/forum) contrib |
| `statistics` | [Statistics](https://www.drupal.org/project/statistics) contrib |
| `tour` | [Tour](https://www.drupal.org/project/tour) contrib |
| `help` | Inline help in admin UI |
| `quickedit` | Removed (no direct replacement) |
| `color` | Theme-level color management |
| `rdf` | [RDF](https://www.drupal.org/project/rdf) contrib |

### Single-Directory Components (SDC)

Drupal 11 promotes **Single-Directory Components** for reusable Twig components:

```
my_theme/
  components/
    card/
      card.component.yml
      card.twig
      card.css
```

```yaml
# card.component.yml
name: Card
props:
  type: object
  properties:
    title:
      type: string
    body:
      type: string
slots:
  footer:
    title: Footer
```

### Recipe system improvements

Drupal 11.1 enhanced the **recipe system** for site building:

```bash
# Apply a recipe during installation
drush site:install --recipe=drupal_cms_starter
```

### CKEditor 5 only

CKEditor 4 is removed. All rich text uses **CKEditor 5**.

### Database requirements

- **MySQL 8.0+** or **MariaDB 10.6+**
- **PostgreSQL 16+**
- **SQLite 3.45+** (development only)

## Upgrade path from Drupal 10

```bash
# Update core constraint in composer.json
composer require 'drupal/core-recommended:^11' --update-with-all-dependencies

# Run database updates
drush updatedb -y

# Clear caches
drush cr
```

!!! warning "Check contrib compatibility"
    Run `drush pm:security` and verify all contrib modules support Drupal 11 before upgrading production sites.
