# PHP 8.0 – 8.3

PHP 8.0 was the largest release since 7.0. It added a JIT compiler, but its real impact was on how code is written: named arguments, attributes, `match`, constructor promotion, and the nullsafe operator. It also fixed long-standing type-juggling surprises. 8.1 through 8.3 added enums, readonly data, and stronger guarantees for class hierarchies.

## PHP 8.0

### Named arguments

Pass arguments by parameter name, in any order, and skip optional parameters you don't need to change:

=== "PHP 7.4"

    ```php
    // What do false, 'UTF-8' and false mean? You have to check the manual.
    echo htmlspecialchars($input, ENT_QUOTES, 'UTF-8', false);

    // To change the last option you must repeat every default before it.
    setcookie('session', $id, 0, '', '', true, true);
    ```

=== "PHP 8.0+"

    ```php
    echo htmlspecialchars($input, double_encode: false);

    setcookie('session', $id, secure: true, httponly: true);
    ```

!!! warning "Parameter names are now part of your public API"
    Once callers use named arguments, renaming a parameter in a library is a breaking change. Pick parameter names as carefully as method names.

### Constructor property promotion

=== "PHP 7.4"

    ```php
    class Money
    {
        private int $amount;
        private string $currency;

        public function __construct(int $amount, string $currency)
        {
            $this->amount = $amount;
            $this->currency = $currency;
        }
    }
    ```

=== "PHP 8.0+"

    ```php
    class Money
    {
        public function __construct(
            private int $amount,
            private string $currency,
        ) {}
    }
    ```

### Attributes

Structured, machine-readable metadata replaces docblock annotations that frameworks used to parse as strings:

=== "Docblock annotation (PHP 5/7)"

    ```php
    class OrderController
    {
        /**
         * @Route("/api/orders/{id}", methods={"GET"})
         */
        public function show($id) { /* ... */ }
    }
    ```

=== "Attribute (PHP 8.0+)"

    ```php
    class OrderController
    {
        #[Route('/api/orders/{id}', methods: ['GET'])]
        public function show(int $id): Response { /* ... */ }
    }
    ```

Attributes are ordinary classes, read through Reflection:

```php
#[Attribute(Attribute::TARGET_METHOD)]
final class Route
{
    public function __construct(
        public string $path,
        public array $methods = ['GET'],
    ) {}
}

$method = new ReflectionMethod(OrderController::class, 'show');
foreach ($method->getAttributes(Route::class) as $attribute) {
    $route = $attribute->newInstance();   // Route object
    echo $route->path;                    // /api/orders/{id}
}
```

### `match` expression

`match` is an expression (it returns a value), compares with `===`, has no fall-through, and throws `UnhandledMatchError` when nothing matches:

=== "PHP 7.4"

    ```php
    switch ($status) {
        case 'draft':
        case 'review':
            $label = 'Unpublished';
            break;
        case 'live':
            $label = 'Published';
            break;
        default:
            throw new InvalidArgumentException("Unknown status $status");
    }
    ```

=== "PHP 8.0+"

    ```php
    $label = match ($status) {
        'draft', 'review' => 'Unpublished',
        'live' => 'Published',
        default => throw new InvalidArgumentException("Unknown status $status"),
    };
    ```

Use `match (true)` for range checks:

```php
$band = match (true) {
    $score >= 90 => 'A',
    $score >= 75 => 'B',
    $score >= 50 => 'C',
    default => 'F',
};
```

### Nullsafe operator `?->`

=== "PHP 7.4"

    ```php
    $country = null;
    if ($order !== null) {
        $address = $order->getShippingAddress();
        if ($address !== null) {
            $country = $address->getCountry();
        }
    }
    ```

=== "PHP 8.0+"

    ```php
    $country = $order?->getShippingAddress()?->getCountry();
    ```

The chain stops at the first `null` and the whole expression evaluates to `null`.

### Union types, `mixed`, and `static`

```php
class Collection
{
    public function find(int|string $key): Item|false { /* ... */ }

    public function meta(string $name): mixed { /* ... */ }

    public static function make(array $items): static   // returns the subclass when called on one
    {
        return new static($items);
    }
}
```

### Throw is an expression

```php
$user = $repository->find($id) ?? throw new NotFoundException("User $id not found");

$handler = fn() => throw new LogicException('Not implemented');
```

### New string functions

=== "PHP 7.4"

    ```php
    if (strpos($haystack, 'needle') !== false) { /* ... */ }
    if (substr($path, 0, 1) === '/') { /* ... */ }
    if (substr($file, -4) === '.php') { /* ... */ }
    ```

=== "PHP 8.0+"

    ```php
    if (str_contains($haystack, 'needle')) { /* ... */ }
    if (str_starts_with($path, '/')) { /* ... */ }
    if (str_ends_with($file, '.php')) { /* ... */ }
    ```

### More 8.0 additions

| Feature | Example |
|---------|---------|
| Non-capturing catch | `catch (TimeoutException) { … }` |
| `::class` on objects | `$entity::class` instead of `get_class($entity)` |
| `Stringable` interface | Added automatically to any class with `__toString()` |
| `get_debug_type()` | Returns `int`, `string`, `App\User`, … (better than `gettype()`) |
| `WeakMap` | Caches keyed by objects that don't prevent garbage collection |
| Trailing comma in parameter lists | `function f(int $a, int $b,) {}` |
| `ValueError` | Thrown by internal functions for invalid argument values |
| JIT | `opcache.jit=tracing` with `opcache.jit_buffer_size=64M`; helps CPU-heavy code most |

### Saner comparisons and numeric strings

PHP 8 changed how `==` compares numbers with non-numeric strings. Before 8.0, the string was converted to a number (usually `0`); now the number is converted to a string and the two are compared as strings.

| Expression | PHP 7.4 | PHP 8.0+ |
|------------|---------|----------|
| `0 == "foo"` | `true` | `false` |
| `0 == ""` | `true` | `false` |
| `"1" == "01"` | `true` | `true` |
| `100 == "1e2"` | `true` | `true` |
| `in_array("abc", [0])` | `true` | `false` |
| `"abc" * 1` | `0` with a warning | `TypeError` |
| `"5 apples" + 1` | `6` with a notice | `6` with a warning |

This closes a class of authentication and validation bugs, but it can change behaviour in code that relied on the old rules. See [Breaking Changes](breaking-changes.md#php-74-80).

## PHP 8.1

### Enums

=== "PHP 7.4"

    ```php
    final class Status
    {
        public const DRAFT = 'draft';
        public const PUBLISHED = 'published';
        public const ARCHIVED = 'archived';

        public static function all(): array
        {
            return [self::DRAFT, self::PUBLISHED, self::ARCHIVED];
        }
    }

    function publish(string $status) {}  // any string is accepted
    ```

=== "PHP 8.1+"

    ```php
    enum Status: string
    {
        case Draft = 'draft';
        case Published = 'published';
        case Archived = 'archived';

        public function label(): string
        {
            return ucfirst($this->value);
        }
    }

    function publish(Status $status) {}  // only a Status case is accepted

    Status::from('draft');       // Status::Draft
    Status::tryFrom('deleted');  // null
    Status::cases();             // [Status::Draft, Status::Published, Status::Archived]
    ```

Enums can implement interfaces and declare constants and methods, but they cannot have state. A **pure** enum (`enum Suit { case Hearts; … }`) has no backing value; a **backed** enum maps each case to an `int` or `string`.

### Readonly properties

A `readonly` property can be written once, from inside the class, and never changed:

```php
final class EmailAddress
{
    public function __construct(
        public readonly string $value,
    ) {
        if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidArgumentException("Invalid email: $value");
        }
    }
}

$email = new EmailAddress('ana@example.com');
echo $email->value;          // ana@example.com
$email->value = 'x';         // Error: Cannot modify readonly property EmailAddress::$value
```

### First-class callable syntax

`name(...)` creates a `Closure` from any callable. It is checked by static analysers and IDEs, unlike string or array callables:

=== "PHP 7.4"

    ```php
    $lengths = array_map('strlen', $words);
    $handler = [$this, 'handleEvent'];
    $factory = Closure::fromCallable([UserFactory::class, 'create']);
    ```

=== "PHP 8.1+"

    ```php
    $lengths = array_map(strlen(...), $words);
    $handler = $this->handleEvent(...);
    $factory = UserFactory::create(...);
    ```

### More 8.1 additions

| Feature | Example |
|---------|---------|
| `never` return type | `function redirect(string $url): never { header("Location: $url"); exit; }` |
| `new` in initializers | `public function __construct(private Logger $logger = new NullLogger()) {}` |
| Pure intersection types | `function save(Countable&Traversable $items)` |
| Fibers | Low-level coroutines used by async libraries (Revolt, AMPHP, ReactPHP) |
| `array_is_list()` | `array_is_list([1, 2])` is `true`; `array_is_list([1 => 'a'])` is `false` |
| `final` class constants | `final public const VERSION = '2.0';` |
| Explicit octal prefix | `0o755` (same as `0755`) |
| Spread with string keys | `[...$defaults, ...$options]` now works for associative arrays |

## PHP 8.2

### Readonly classes

Marks every property readonly and forbids dynamic properties, which makes it a natural fit for value objects and DTOs:

```php
readonly class Coordinates
{
    public function __construct(
        public float $latitude,
        public float $longitude,
    ) {}

    public function withLatitude(float $latitude): static
    {
        return new static($latitude, $this->longitude);
    }
}
```

### DNF types and standalone `null`, `false`, `true`

Disjunctive Normal Form types combine union and intersection types:

```php
function cacheKey((Countable&ArrayAccess)|null $items): string { /* ... */ }

function isEnabled(): true { return true; }       // 8.2: true, false and null are standalone types
```

### Dynamic properties are deprecated

Assigning an undeclared property now emits a deprecation:

```php
class User {}

$user = new User();
$user->nickname = 'ana';   // Deprecated: Creation of dynamic property User::$nickname is deprecated
```

Declare the property, use `stdClass`, implement `__get()`/`__set()`, or (as a temporary measure) mark the class with `#[\AllowDynamicProperties]`.

### More 8.2 additions

| Feature | Example |
|---------|---------|
| `Random` extension | `(new Random\Randomizer())->shuffleArray($deck)` with pluggable engines |
| Constants in traits | `trait HasVersion { public const VERSION = 1; }` |
| `#[\SensitiveParameter]` | Hides a parameter's value from stack traces: `function login(string $user, #[\SensitiveParameter] string $password)` |
| `ini_parse_quantity()` | `ini_parse_quantity('256M')` returns `268435456` |

Also **deprecated** in 8.2: `${var}` string interpolation (use `{$var}`) and `utf8_encode()`/`utf8_decode()` (use `mb_convert_encoding()`).

## PHP 8.3

### Typed class constants

```php
interface HasTable
{
    public const string TABLE = '';
}

final class Invoice implements HasTable
{
    public const string TABLE = 'invoices';
}
```

### `#[\Override]`

Declares that a method is meant to override a parent or interface method. If the parent method is renamed or removed, PHP raises a compile-time error instead of silently turning your override into a new, unused method:

```php
class Mailer
{
    protected function buildHeaders(): array { return []; }
}

final class SmtpMailer extends Mailer
{
    #[\Override]
    protected function buildHeaders(): array
    {
        return ['X-Mailer' => 'smtp'];
    }
}
```

### More 8.3 additions

| Feature | Example |
|---------|---------|
| Dynamic class constant fetch | `Status::{$name}` instead of `constant(Status::class . '::' . $name)` |
| `json_validate()` | Checks JSON syntax without decoding it into memory |
| Deep-cloning readonly properties | Readonly properties can be re-initialized inside `__clone()` |
| `mb_str_pad()` | Multibyte-safe `str_pad()` |
| `str_increment()` / `str_decrement()` | `str_increment('Az')` returns `'Ba'` |
| `Randomizer::getBytesFromString()` | Random strings from a custom alphabet |
| Lint several files | `php -l src/A.php src/B.php` |

Next: [PHP 8.4 – 8.6](php-8-4-plus.md).
