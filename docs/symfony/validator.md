# Validator

The **Validator** component validates objects and arrays against **constraints** (rules). Drupal uses it for entity validation, Form API validation, and custom business rules.

## Core concepts

| Term | Meaning |
|------|---------|
| Constraint | A rule (e.g. `NotBlank`, `Email`) |
| ConstraintValidator | Class that checks the constraint |
| Violation | A single validation failure with message and property path |
| Validation groups | Named sets of constraints applied conditionally |

```php
<?php

declare(strict_types=1);

use Symfony\Component\Validator\Validation;

$validator = Validation::createValidator();
$violations = $validator->validate('not-an-email', new Assert\Email());

foreach ($violations as $violation) {
  echo $violation->getMessage();
}
```

## How Drupal uses the Validator

### Entity constraint plugins

Drupal maps Symfony-style validation to **Constraint plugins** in `Plugin/Validation/Constraint/`:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Plugin\Validation\Constraint;

use Symfony\Component\Validator\Constraint;

/**
 * @Constraint(
 *   id = "UniqueSku",
 *   label = @Translation("Unique SKU", context = "Validation"),
 *   type = "string"
 * )
 */
final class UniqueSkuConstraint extends Constraint {

  public string $message = 'SKU %sku is already in use.';

}
```

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Plugin\Validation\Constraint;

use Drupal\Core\DependencyInjection\ContainerInjectionInterface;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;
use Symfony\Component\Validator\Constraint;
use Symfony\Component\Validator\ConstraintValidator;

final class UniqueSkuConstraintValidator extends ConstraintValidator implements ContainerInjectionInterface {

  public function __construct(
    private readonly EntityTypeManagerInterface $entityTypeManager,
  ) {}

  public static function create(ContainerInterface $container): static {
    return new static($container->get('entity_type.manager'));
  }

  public function validate(mixed $value, Constraint $constraint): void {
    assert($constraint instanceof UniqueSkuConstraint);

    if ($value === NULL || $value === '') {
      return;
    }

    $entity = $this->context->getObject();
    $query = $this->entityTypeManager->getStorage('product')->getQuery()
      ->accessCheck(FALSE)
      ->condition('sku', $value);

    if ($entity->id()) {
      $query->condition('id', $entity->id(), '<>');
    }

    if ($query->count()->execute() > 0) {
      $this->context->addViolation($constraint->message, ['%sku' => $value]);
    }
  }

}
```

Attach to a field via `*.constraints.yml` or entity definition:

```yaml
# my_module.constraints.yml (or field definition)
UniqueSku:
  type: UniqueSku
```

### Entity::validate()

Entities run validation before save:

```php
<?php

declare(strict_types=1);

use Drupal\my_module\Entity\Product;

$product = Product::create(['sku' => 'DUPLICATE-1', 'title' => 'Widget']);
$violations = $product->validate();

if ($violations->count() > 0) {
  foreach ($violations as $violation) {
    \Drupal::messenger()->addError($violation->getMessage());
  }
}
else {
  $product->save();
}
```

### Form validation

Form API `#element_validate` and `#validate` callbacks are separate from Symfony Validator, but entity forms call `$entity->validate()` on submit. For standalone forms, inject the validator service:

```php
<?php

declare(strict_types=1);

/** @var \Symfony\Component\Validator\Validator\ValidatorInterface $validator */
$validator = \Drupal::service('validation.basic_recursive_validator');
```

## Built-in Symfony constraints in Drupal

Drupal ships `drupal/core-validation` bridging common constraints:

=== "Field definition"

    ```yaml
    # field.storage.product.email.yml
    langcode: en
    status: true
    dependencies:
      module:
        - my_module
    id: product.email
    field_name: email
    entity_type: product
    type: email
    settings: {  }
    module: my_module
    locked: false
    cardinality: 1
    translatable: false
    constraints:
      NotBlank: {}
      Email: {}
    ```

=== "Programmatic validation"

    ```php
    <?php

    declare(strict_types=1);

    use Symfony\Component\Validator\Constraints as Assert;

    $constraints = new Assert\Collection([
      'email' => [new Assert\NotBlank(), new Assert\Email()],
      'quantity' => [new Assert\Positive()],
    ]);

    $validator = \Drupal::service('validation.basic_recursive_validator');
    $violations = $validator->validate($payload, $constraints);
    ```

## Custom constraint on entity level

```php
<?php

declare(strict_types=1);

/**
 * @ContentEntityType(
 *   id = "product",
 *   constraints = {
 *     "ProductPricePositive" = {},
 *   },
 * )
 */
```

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Plugin\Validation\Constraint;

use Symfony\Component\Validator\Constraint;

/**
 * @Constraint(
 *   id = "ProductPricePositive",
 *   label = @Translation("Positive price", context = "Validation"),
 * )
 */
final class ProductPricePositiveConstraint extends Constraint {

  public string $message = 'Price must be greater than zero.';

}
```

## Custom module example: API payload validation

=== "Controller"

    ```php
    <?php

    declare(strict_types=1);

    namespace Drupal\my_module\Controller;

    use Drupal\Core\Controller\ControllerBase;
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\Validator\Constraints as Assert;
    use Symfony\Component\Validator\Validator\ValidatorInterface;

    final class ProductApiController extends ControllerBase {

      public function __construct(
        private readonly ValidatorInterface $validator,
      ) {}

      public static function create($container): static {
        return new static($container->get('validation.basic_recursive_validator'));
      }

      public function createProduct(Request $request): JsonResponse {
        $payload = json_decode($request->getContent(), TRUE, 512, JSON_THROW_ON_ERROR);

        $violations = $this->validator->validate($payload, new Assert\Collection([
          'sku' => [new Assert\NotBlank(), new Assert\Length(max: 64)],
          'title' => [new Assert\NotBlank()],
          'price' => [new Assert\Positive()],
        ]));

        if ($violations->count() > 0) {
          $errors = [];
          foreach ($violations as $violation) {
            $errors[$violation->getPropertyPath()] = $violation->getMessage();
          }
          return new JsonResponse(['errors' => $errors], 422);
        }

        $storage = $this->entityTypeManager()->getStorage('product');
        $product = $storage->create($payload);
        $entityViolations = $product->validate();

        if ($entityViolations->count() > 0) {
          return new JsonResponse(['errors' => (string) $entityViolations], 422);
        }

        $product->save();
        return new JsonResponse(['id' => $product->id()], 201);
      }

    }
    ```

=== "Routing"

    ```yaml
    my_module.api.product_create:
      path: '/api/products'
      defaults:
        _controller: '\Drupal\my_module\Controller\ProductApiController::createProduct'
      requirements:
        _permission: 'create product entities'
      methods: [POST]
    ```

!!! tip "Validate twice for APIs"
    Validate the raw payload with Symfony constraints for shape/type, then call `$entity->validate()` for Drupal entity/field rules.

## Related reading

- [Serializer](serializer.md) — deserialize then validate
- [Custom Modules: Forms](../custom-coding/forms/index.md)
