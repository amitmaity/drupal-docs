# PHP 8.4 – 8.6

The three most recent releases continued the 8.x theme: less boilerplate around objects, more functional-style tools, and steady clean-up of legacy behaviour. If your code already targets 8.3, these are incremental upgrades with few breaking changes.

## PHP 8.4 (November 2024)

### Property hooks

Properties can define `get` and `set` logic directly, so you no longer need a private property plus a getter and setter just to add validation or a computed value:

=== "PHP 8.3"

    ```php
    final class User
    {
        private string $email;

        public function __construct(
            private string $firstName,
            private string $lastName,
            string $email,
        ) {
            $this->setEmail($email);
        }

        public function getFullName(): string
        {
            return $this->firstName . ' ' . $this->lastName;
        }

        public function getEmail(): string
        {
            return $this->email;
        }

        public function setEmail(string $email): void
        {
            if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
                throw new InvalidArgumentException("Invalid email: $email");
            }
            $this->email = strtolower($email);
        }
    }
    ```

=== "PHP 8.4+"

    ```php
    final class User
    {
        // Virtual property: computed on every read, no storage.
        public string $fullName {
            get => $this->firstName . ' ' . $this->lastName;
        }

        // Backed property with validation on write.
        public string $email {
            set (string $value) {
                if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
                    throw new InvalidArgumentException("Invalid email: $value");
                }
                $this->email = strtolower($value);
            }
        }

        public function __construct(
            public string $firstName,
            public string $lastName,
            string $email,
        ) {
            $this->email = $email;   // runs the set hook
        }
    }

    $user = new User('Ana', 'Silva', 'Ana@Example.com');
    echo $user->fullName;   // Ana Silva
    echo $user->email;      // ana@example.com
    ```

Hooks can also be declared on interfaces (`public string $name { get; }`), so an interface can require a readable property instead of a getter method.

### Asymmetric visibility

Readable from outside, writable only from inside:

```php
final class Order
{
    public private(set) string $status = 'new';

    public function ship(): void
    {
        $this->status = 'shipped';
    }
}

$order = new Order();
$order->ship();
echo $order->status;        // shipped
$order->status = 'lost';    // Error: Cannot modify private(set) property Order::$status from global scope
```

`protected(set)` lets subclasses write as well. Unlike `readonly`, the property can change more than once.

### `new` without wrapping parentheses

```php
// PHP 8.3
$name = (new Greeter())->hi();

// PHP 8.4
$name = new Greeter()->hi();
```

### Array search functions

```php
$people = [
    ['name' => 'Ana', 'age' => 31],
    ['name' => 'Ben', 'age' => 17],
];

array_find($people, fn($p) => $p['age'] < 18);       // ['name' => 'Ben', 'age' => 17]
array_find_key(['a' => 1, 'b' => 2], fn($v) => $v === 2);   // 'b'
array_any($people, fn($p) => $p['age'] < 18);        // true
array_all($people, fn($p) => $p['age'] < 18);        // false
```

### More 8.4 additions

| Feature | Example |
|---------|---------|
| `#[\Deprecated]` | Mark your own functions, methods and constants deprecated; callers get an `E_USER_DEPRECATED` |
| Lazy objects | `(new ReflectionClass(Heavy::class))->newLazyGhost($initializer)` defers construction until first use |
| HTML5 DOM parser | `Dom\HTMLDocument::createFromString($html)->querySelector('p.intro')` |
| Multibyte helpers | `mb_trim()`, `mb_ltrim()`, `mb_rtrim()`, `mb_ucfirst()`, `mb_lcfirst()` |
| Arbitrary-precision object | `new BcMath\Number('0.1') + new BcMath\Number('0.2')` is exactly `0.3` |
| Driver-specific PDO classes | `PDO::connect($dsn)` returns `Pdo\Mysql`, `Pdo\Sqlite`, … |
| Stronger default bcrypt | `PASSWORD_DEFAULT` cost raised from 10 to 12 |

### Deprecated in 8.4

- **Implicitly nullable parameters.** `function f(string $s = null)` must become `function f(?string $s = null)`. This is the 8.4 deprecation you are most likely to hit in older code.
- `trigger_error()` with `E_USER_ERROR` (throw an exception instead).
- The `E_STRICT` constant.
- `_` as a class name.
- `lcg_value()` (use `Random\Randomizer::getFloat()`).

The IMAP, OCI8, PDO_OCI and Pspell extensions moved from core to PECL.

## PHP 8.5 (November 2025)

### Pipe operator `|>`

Passes the value on the left as the single argument of the callable on the right, so a chain of transformations reads top to bottom instead of inside out:

=== "Before"

    ```php
    $slug = strtolower(str_replace(' ', '-', trim($title)));
    ```

=== "PHP 8.5+"

    ```php
    $slug = $title
        |> trim(...)
        |> (fn($s) => str_replace(' ', '-', $s))
        |> strtolower(...);
    ```

The right-hand side must be a callable that takes one argument. First-class callables (`trim(...)`) work directly; arrow functions must be wrapped in parentheses.

### Clone with

`clone()` accepts an array of properties to change on the copy, which makes "wither" methods on readonly classes a one-liner:

```php
readonly class Color
{
    public function __construct(
        public int $red,
        public int $green,
        public int $blue,
        public int $alpha = 255,
    ) {}

    public function withAlpha(int $alpha): static
    {
        return clone($this, ['alpha' => $alpha]);
    }
}

$translucent = (new Color(10, 20, 30))->withAlpha(128);
```

### `#[\NoDiscard]` and the `(void)` cast

Marks a function whose return value must not be ignored. Ignoring it raises a warning; cast to `(void)` to ignore it on purpose:

```php
#[\NoDiscard('the result says whether the write succeeded')]
function save(array $record): bool { /* ... */ }

save($record);          // Warning: The return value of function save() should either be used or intentionally ignored…
(void) save($record);   // deliberately ignored, no warning
```

### `array_first()` and `array_last()`

```php
array_first(['x' => 1, 'y' => 2]);   // 1
array_last([1, 2, 3]);               // 3
array_first([]);                     // null
```

### URI extension

Standards-compliant URL parsing, always available. `parse_url()` follows neither RFC 3986 nor the WHATWG URL standard and is easy to fool, so prefer these classes for security-sensitive parsing:

```php
$uri = new Uri\Rfc3986\Uri('https://example.com:8080/path?q=1');
$uri->getHost();   // example.com
$uri->getPort();   // 8080

$url = new Uri\WhatWg\Url('HTTPS://EXAMPLE.com/a/../b');
$url->toAsciiString();   // https://example.com/b (normalized the way browsers do)
```

### More 8.5 additions

| Feature | Example |
|---------|---------|
| Closures in constant expressions | `public const CHECK = static function (int $n): bool { return $n > 0; };` |
| `final` promoted properties | `public function __construct(final public string $name) {}` |
| Attributes on constants | `#[Since('2.0')] const API_VERSION = 2;` |
| `#[\Override]` on properties | Guards a property that redeclares a parent's |
| Fatal error backtraces | Fatal errors (such as timeouts) now include a stack trace |
| Handler getters | `get_error_handler()`, `get_exception_handler()` |
| Persistent cURL share handles | `curl_share_init_persistent()` reuses DNS and connection caches across requests |

### Deprecated in 8.5

- The backtick operator (`` `ls` ``). Use `shell_exec()`, or better, `proc_open()` with an argument array.
- Non-canonical casts `(integer)`, `(boolean)`, `(double)`, `(binary)`. Use `(int)`, `(bool)`, `(float)`, `(string)`.
- Incrementing non-numeric strings with `++`. Use `str_increment()`.
- `null` as an array offset, or as the key in `array_key_exists()`.
- `case` labels terminated with `;` instead of `:`.
- `__sleep()` and `__wakeup()` (soft-deprecated; use `__serialize()` and `__unserialize()`).
- `curl_close()`, `finfo_close()`, `imagedestroy()` and `xml_parser_free()`, which have done nothing since their resources became objects.

## PHP 8.6 (November 2026)

PHP 8.6 was released on 19 November 2026. The samples below follow the syntax in the [official release announcement](https://www.php.net/releases/8.6/en.php). They were not executed for this guide; check the manual before relying on edge cases.

### Partial function application

Call a function with `?` placeholders (or `...` for "all remaining arguments") and you get back a `Closure` that waits for the missing arguments instead of calling the function immediately:

```php title="PHP 8.6"
$isModern = version_compare(?, '8.6', '>=');        // Closure taking one argument

$modern = array_filter(['8.4.1', '8.6.0', '8.5.2'], $isModern);   // [1 => '8.6.0']

$dashed = str_replace(' ', '-', ?);                  // Closure: fn($subject) => str_replace(' ', '-', $subject)
```

This combines naturally with the 8.5 pipe operator, which needs single-argument callables:

```php title="PHP 8.6"
$slug = $title |> trim(...) |> str_replace(' ', '-', ?) |> strtolower(...);
```

### `clamp()`

```php title="PHP 8.6"
$volume = clamp($requested, min: 0, max: 100);   // 0–100, whatever was requested
```

It works with any comparable values, not just numbers.

### Readonly properties with defaults

```php title="PHP 8.6"
final class SlugGenerator
{
    public readonly string $separator = '-';   // a compile-time error before 8.6
}
```

The default counts as the property's one-time initialization, so the constructor cannot assign it again.

### More 8.6 additions

| Feature | Notes |
|---------|-------|
| `Time\Duration` | An immutable length of time with nanosecond precision: `Duration::fromSeconds(30)`, `->add()`, `->multiplyBy()` |
| `SortDirection` enum | Built-in `SortDirection::Ascending` / `SortDirection::Descending` for sort-order parameters in APIs |
| Polling API (`Io\Poll`) | Waits on many streams at once using epoll, kqueue, event ports or WSAPoll |
| Stream error handling | `StreamException`, `StreamErrorMode` and `StreamErrorCode` for structured stream errors |
| `#[\Override]` on constants | Also applies to enum cases |
| URI builders | `Uri\Rfc3986\UriBuilder`, `Uri\WhatWg\UrlBuilder` |

### Deprecated or changed in 8.6

- `return` inside a `finally` block.
- Returning a value from `__construct()` or `__destruct()`.
- `is_double()`, `is_long()`, `is_integer()` and `doubleval()` (use `is_float()`, `is_int()`, `floatval()`).
- `spl_object_hash()` (use `spl_object_id()`).
- The mbstring regex functions (`mb_ereg()`, `mb_split()`, …).
- **Session defaults changed:** `session.use_strict_mode=1`, `session.cookie_httponly=1`, `session.cookie_samesite=Lax`. Applications that relied on JavaScript reading the session cookie, or on cross-site POSTs carrying it, must opt back in explicitly.
- `trim()`, `ltrim()` and `rtrim()` now strip form feed (`\f`) by default.

See [Breaking Changes](breaking-changes.md) for the full upgrade-by-upgrade list, and the official [migration guides](https://www.php.net/manual/en/appendices.php) for every detail.
