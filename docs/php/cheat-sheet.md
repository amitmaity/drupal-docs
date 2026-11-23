# Cheat Sheet

PHP 5 idioms on the left, their modern replacements on the right. The **Since** column is the first PHP version that supports the modern form.

## Arrays and strings

| PHP 5 | Modern PHP | Since |
|-------|------------|-------|
| `array(1, 2, 3)` | `[1, 2, 3]` | 5.4 |
| `isset($a['k']) ? $a['k'] : 'default'` | `$a['k'] ?? 'default'` | 7.0 |
| `if (!isset($a['k'])) { $a['k'] = 'default'; }` | `$a['k'] ??= 'default';` | 7.4 |
| `list($id, $name) = $row;` | `[$id, $name] = $row;` | 7.1 |
| `$email = $r['email']; $role = $r['role'];` | `['email' => $email, 'role' => $role] = $r;` | 7.1 |
| `array_merge($defaults, $options)` | `[...$defaults, ...$options]` | 8.1 |
| `$keys = array_keys($a); $first = $keys[0];` | `array_key_first($a)` | 7.3 |
| `reset($a)` / `end($a)` to read a value | `array_first($a)` / `array_last($a)` | 8.5 |
| `foreach` + `break` to find an item | `array_find($a, fn($x) => …)` | 8.4 |
| `strpos($s, 'x') !== false` | `str_contains($s, 'x')` | 8.0 |
| `substr($s, 0, 4) === 'http'` | `str_starts_with($s, 'http')` | 8.0 |
| `substr($s, -4) === '.php'` | `str_ends_with($s, '.php')` | 8.0 |
| `substr($s, -1)` | `$s[-1]` | 7.1 |
| `$s{0}` | `$s[0]` | always; `{}` removed in 8.0 |
| `"Hello ${name}"` | `"Hello {$name}"` | always; `${}` deprecated in 8.2 |
| `1000000` | `1_000_000` | 7.4 |

## Functions and closures

| PHP 5 | Modern PHP | Since |
|-------|------------|-------|
| `create_function('$x', 'return $x * 2;')` | `fn($x) => $x * 2` | 7.4 |
| `function ($x) use ($rate) { return $x * $rate; }` | `fn($x) => $x * $rate` | 7.4 |
| `call_user_func_array(array($obj, 'send'), $args)` | `$obj->send(...$args)` | 5.6 |
| `$args = func_get_args();` | `function log(string ...$messages)` | 5.6 |
| `array_map('trim', $lines)` | `array_map(trim(...), $lines)` | 8.1 |
| `Closure::fromCallable(array($this, 'handle'))` | `$this->handle(...)` | 8.1 |
| `htmlspecialchars($s, ENT_QUOTES, 'UTF-8', false)` | `htmlspecialchars($s, double_encode: false)` | 8.0 |
| `strtolower(trim($s))` | `$s |> trim(...) |> strtolower(...)` | 8.5 |

## Types and classes

| PHP 5 | Modern PHP | Since |
|-------|------------|-------|
| `/** @param int $id */ function find($id)` | `function find(int $id)` | 7.0 |
| `/** @return string */` | `function name(): string` | 7.0 |
| `/** @var string */ private $name;` | `private string $name;` | 7.4 |
| `function f(Foo $x = null)` | `function f(?Foo $x = null)` (required from 8.4) | 7.1 |
| Constructor that only copies arguments into properties | Constructor property promotion | 8.0 |
| `const STATUS_DRAFT = 'draft';` constants used as an enum | `enum Status: string { case Draft = 'draft'; }` | 8.1 |
| Private property + getter, never changed after construction | `public readonly string $id` | 8.1 |
| Private property + getter, changed only internally | `public private(set) string $status` | 8.4 |
| Getter and setter pair with validation | Property hook: `public string $email { set { … } }` | 8.4 |
| `get_class($obj)` | `$obj::class` | 8.0 |
| `/** @Route("/x") */` docblock annotation | `#[Route('/x')]` attribute | 8.0 |
| `function __autoload($class)` | Composer PSR-4 autoloading | 5.3+ (Composer) |
| A comment saying "overrides parent" | `#[\Override]` | 8.3 |
| `$c = clone $this; $c->alpha = $a; return $c;` | `return clone($this, ['alpha' => $a]);` | 8.5 |

## Control flow and errors

| PHP 5 | Modern PHP | Since |
|-------|------------|-------|
| `switch` that assigns a value in each `case` | `$x = match ($y) { … };` | 8.0 |
| `$a !== null ? $a->getB() : null` | `$a?->getB()` | 8.0 |
| `if (!$user) { throw new NotFound(); }` | `$user ?? throw new NotFound();` | 8.0 |
| Sort callback with `if`/`else` returning -1, 0, 1 | `fn($a, $b) => $a <=> $b` | 7.0 |
| `catch (Exception $e)` as a catch-all | `catch (Throwable $e)` | 7.0 |
| Two `catch` blocks with the same body | One `catch` listing both classes, separated by `|` | 7.1 |
| `catch (Foo $e)` where `$e` is unused | `catch (Foo)` | 8.0 |
| Checking `json_last_error()` after `json_decode()` | `json_decode($s, flags: JSON_THROW_ON_ERROR)` | 7.3 |
| `json_decode($s) !== null` to validate | `json_validate($s)` | 8.3 |

## Security and the standard library

| PHP 5 | Modern PHP | Since |
|-------|------------|-------|
| `mysql_query("… WHERE id = $id")` | PDO prepared statement with placeholders | PDO since 5.1; `mysql_*` removed in 7.0 |
| `md5($password)` or `sha1($password)` | `password_hash($password, PASSWORD_DEFAULT)` and `password_verify()` | 5.5 |
| `rand()`, `mt_rand()` or `uniqid()` for tokens | `bin2hex(random_bytes(16))`, `random_int(0, 999999)` | 7.0 |
| `mcrypt_encrypt()` | `sodium_crypto_secretbox()` | 7.2 |
| `ereg('^[a-z]+$', $s)` | `preg_match('/^[a-z]+$/', $s)` | always; `ereg` removed in 7.0 |
| `utf8_encode($s)` | `mb_convert_encoding($s, 'UTF-8', 'ISO-8859-1')` | always; deprecated in 8.2 |
| `strftime('%d %B %Y')` | `(new IntlDateFormatter('en_GB', IntlDateFormatter::LONG, IntlDateFormatter::NONE))->format($date)` | intl; `strftime` deprecated in 8.1 |
| `parse_url($url)` for security decisions | `new Uri\Rfc3986\Uri($url)` | 8.5 |
| `` `ls -la` `` (backticks) | `shell_exec()` or `proc_open(['ls', '-la'], …)` | backticks deprecated in 8.5 |

## One class, two eras

The same class written for PHP 5.6 and for PHP 8.5.

=== "PHP 5.6"

    ```php
    <?php

    class Invoice
    {
        const STATUS_DRAFT = 'draft';
        const STATUS_SENT = 'sent';
        const STATUS_PAID = 'paid';

        /** @var string */
        private $number;

        /** @var array */
        private $lines = array();

        /** @var string */
        private $status;

        /** @var string|null */
        private $customerEmail;

        /**
         * @param string $number
         * @param string|null $customerEmail
         */
        public function __construct($number, $customerEmail = null)
        {
            $this->number = $number;
            $this->customerEmail = $customerEmail;
            $this->status = self::STATUS_DRAFT;
        }

        public function getNumber()
        {
            return $this->number;
        }

        public function getStatus()
        {
            return $this->status;
        }

        public function addLine($description, $amount)
        {
            if (!is_numeric($amount)) {
                throw new InvalidArgumentException('Amount must be numeric');
            }
            $this->lines[] = array('description' => $description, 'amount' => $amount);
        }

        public function total()
        {
            return array_sum(array_map(function ($line) {
                return $line['amount'];
            }, $this->lines));
        }

        public function statusLabel()
        {
            switch ($this->status) {
                case self::STATUS_DRAFT:
                    return 'Draft';
                case self::STATUS_SENT:
                    return 'Awaiting payment';
                case self::STATUS_PAID:
                    return 'Paid';
                default:
                    throw new LogicException('Unknown status');
            }
        }

        public function recipient()
        {
            return isset($this->customerEmail) ? $this->customerEmail : 'billing@example.com';
        }

        public function isInternal()
        {
            return strpos($this->recipient(), '@example.com') !== false;
        }
    }
    ```

=== "PHP 8.5"

    ```php
    <?php

    declare(strict_types=1);

    enum InvoiceStatus: string
    {
        case Draft = 'draft';
        case Sent = 'sent';
        case Paid = 'paid';

        public function label(): string
        {
            return match ($this) {
                self::Draft => 'Draft',
                self::Sent => 'Awaiting payment',
                self::Paid => 'Paid',
            };
        }
    }

    final class Invoice
    {
        public private(set) InvoiceStatus $status = InvoiceStatus::Draft;

        /** @var list<array{description: string, amount: float}> */
        private array $lines = [];

        public function __construct(
            public readonly string $number,
            private readonly ?string $customerEmail = null,
        ) {}

        public function addLine(string $description, float $amount): void
        {
            $this->lines[] = ['description' => $description, 'amount' => $amount];
        }

        public function total(): float
        {
            return array_sum(array_column($this->lines, 'amount'));
        }

        public function recipient(): string
        {
            return $this->customerEmail ?? 'billing@example.com';
        }

        public function isInternal(): bool
        {
            return str_ends_with($this->recipient(), '@example.com');
        }
    }

    $invoice = new Invoice('INV-001', customerEmail: 'ana@example.com');
    $invoice->addLine('Consulting', 1200.0);
    echo $invoice->status->label();   // Draft
    echo $invoice->total();           // 1200
    ```

What changed:

- **Types everywhere** replace docblocks and the manual `is_numeric()` check. `strict_types` rejects a string amount at the call site.
- **An enum** replaces three string constants, and `match` inside the enum replaces the `switch`. An unknown status is now impossible rather than a runtime `LogicException`.
- **Constructor promotion** and **`readonly`** remove the property declarations, the assignments and the `getNumber()` getter.
- **Asymmetric visibility** (`public private(set)`) replaces `getStatus()`: anyone can read the status, only the class can change it.
- **`??`**, **`str_ends_with()`** and **`array_column()`** replace the `isset` ternary, the `strpos` check and the closure.

The modern version is shorter, and every invalid state the PHP 5 version guarded against at runtime is now rejected by the type system or the engine.
