# Breaking Changes

This page lists what you must change when upgrading, grouped by upgrade step. For **how** to work through the upgrade (tools, order, testing), see [Upgrade Strategy](upgrading.md).

!!! tip "Deprecated is not removed"
    A deprecation in version *X.Y* emits an `E_DEPRECATED` message but keeps working. Removal normally happens at the next major version. Run your test suite with `error_reporting=E_ALL` so deprecations show up long before they become fatal.

## PHP 5.6 → 7.0

### Removed extensions and functions

| Removed | Replacement |
|---------|-------------|
| `ext/mysql` (`mysql_connect()`, `mysql_query()`, …) | PDO or `mysqli` with prepared statements |
| `ext/ereg` (`ereg()`, `eregi()`, `split()`) | `preg_match()`, `preg_split()` |
| `preg_replace()` with the `/e` modifier | `preg_replace_callback()` |
| `ext/mssql`, `ext/sybase_ct` | `pdo_sqlsrv` / `pdo_dblib` |
| `$HTTP_RAW_POST_DATA` | `file_get_contents('php://input')` |
| `call_user_method()`, `call_user_method_array()` | `call_user_func([$obj, 'method'])` |
| `set_magic_quotes_runtime()` | Delete it; magic quotes were removed in 5.4 |
| ASP tags `<% %>` and `<script language="php">` | `<?php ?>` |
| Assigning `new` by reference (`$o =& new Foo`) | `$o = new Foo` |

The `mysql_*` removal is the change that breaks the most legacy code. Moving to PDO is also the moment to switch to prepared statements:

=== "PHP 5 (ext/mysql)"

    ```php
    $link = mysql_connect('localhost', 'app', $password);
    mysql_select_db('shop', $link);

    // SQL injection: $email is pasted straight into the query.
    $result = mysql_query("SELECT id, name FROM users WHERE email = '$email'", $link);
    while ($row = mysql_fetch_assoc($result)) {
        echo $row['name'];
    }
    ```

=== "PHP 7+ (PDO)"

    ```php
    $pdo = new PDO('mysql:host=localhost;dbname=shop;charset=utf8mb4', 'app', $password, [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,   // the default since PHP 8.0
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
    ]);

    $stmt = $pdo->prepare('SELECT id, name FROM users WHERE email = :email');
    $stmt->execute(['email' => $email]);
    foreach ($stmt as $row) {
        echo $row['name'];
    }
    ```

### Fatal errors become `Error` exceptions

Most fatal errors now throw `Error`, which does **not** extend `Exception`. This breaks two common patterns:

```php
// 1. A catch-all that no longer catches everything.
try {
    $handler->run();
} catch (Exception $e) {        // misses TypeError, DivisionByZeroError, …
    $logger->error($e->getMessage());
}

// 2. An exception handler with an Exception type hint.
set_exception_handler(function (Exception $e) { /* ... */ });  // TypeError when given an Error
```

Catch or type-hint `Throwable` instead.

### Uniform variable syntax

Indirect variable, property and method access is now evaluated strictly left to right:

| Expression | PHP 5 meaning | PHP 7 meaning |
|------------|---------------|---------------|
| `$$foo['bar']['baz']` | `${$foo['bar']['baz']}` | `($$foo)['bar']['baz']` |
| `$foo->$bar['baz']` | `$foo->{$bar['baz']}` | `($foo->$bar)['baz']` |
| `$foo->$bar['baz']()` | `$foo->{$bar['baz']}()` | `($foo->$bar)['baz']()` |
| `Foo::$bar['baz']()` | `Foo::{$bar['baz']}()` | `(Foo::$bar)['baz']()` |

Add explicit braces (`$foo->{$bar['baz']}`) wherever the old meaning was intended.

### Other 7.0 behaviour changes

- `list()` assigns left to right, cannot be empty, and can no longer unpack strings.
- `foreach` no longer moves the array's internal pointer, and by-value `foreach` iterates over a copy.
- Hexadecimal strings are no longer numeric: `"0x10" == 16` is `false`.
- Invalid octal literals (`0789`) are a `ParseError`; shifting by a negative number throws `ArithmeticError`; `%` by zero throws `DivisionByZeroError`.
- `int`, `float`, `bool`, `string`, `null`, `true` and `false` can no longer be class, interface or trait names. `object` (7.2) and `mixed` (8.0) were reserved later. Legacy classes named `String` or `Object` must be renamed.
- PHP 4-style constructors (a method named after its class) are deprecated, and calling a non-static method statically is deprecated. Both became errors in 8.0.

## PHP 7.0 → 7.4

Each 7.x release had a short list of breaks and deprecations. Fix the deprecations now; they are all removals in 8.0.

| Version | Change | What to do |
|---------|--------|------------|
| 7.1 | Too few arguments to a user function throws `ArgumentCountError` (was a warning) | Pass every required argument |
| 7.1 | Arithmetic on non-numeric strings emits "A non-numeric value encountered" | Validate or cast input |
| 7.1 | `mcrypt` deprecated (removed in 7.2) | `sodium_*` or `openssl_*` |
| 7.2 | `count()` on a non-countable value warns (`TypeError` in 8.0) | `is_countable()` first |
| 7.2 | `create_function()`, `each()`, `__autoload()`, `(unset)` cast deprecated | Closures, `foreach`, `spl_autoload_register()` |
| 7.3 | Case-insensitive constants deprecated | Use exact-case constant names |
| 7.3 | `continue` targeting a `switch` warns | `continue 2` |
| 7.4 | Nested ternary without parentheses, `$str{0}`, `(real)`, `array_key_exists()` on objects deprecated | Parenthesize, `$str[0]`, `(float)`, `property_exists()` |

## PHP 7.4 → 8.0

This is the hardest step for most codebases: it removes everything deprecated during 7.x and tightens many behaviours at once.

### Removed functions and syntax

| Removed in 8.0 | Replacement |
|----------------|-------------|
| `create_function()` | Closure or arrow function |
| `each()` | `foreach` |
| `__autoload()` | `spl_autoload_register()` or Composer autoloading |
| `$str{0}` | `$str[0]` |
| `(real)` cast, `is_real()` | `(float)`, `is_float()` |
| `(unset)` cast | `null` |
| `$php_errormsg` / `track_errors` | `error_get_last()` |
| `money_format()` | `NumberFormatter::formatCurrency()` (intl) |
| `get_magic_quotes_gpc()` / `get_magic_quotes_runtime()` | Remove; they always returned `false` |
| `fgetss()` | `strip_tags(fgets($fh))` |
| `hebrevc()`, `convert_cyr_string()`, `restore_include_path()`, `ezmlm_hash()` | `nl2br(hebrev())`, `mb_convert_encoding()`, `ini_restore('include_path')` |
| `define()` with case-insensitive flag | Exact-case constants |
| Unparenthesized nested ternary `a ? b : c ? d : e` | Compile error; add parentheses |
| PHP 4 constructors | A method named after its class is now an ordinary method; rename it to `__construct()` |
| Static call to a non-static method | Throws `Error` |

=== "PHP 5 idiom"

    ```php
    // create_function() was eval() in disguise.
    $double = create_function('$x', 'return $x * 2;');

    // each() advanced the internal pointer.
    while (list($key, $value) = each($config)) {
        echo "$key=$value\n";
    }
    ```

=== "PHP 8"

    ```php
    $double = fn($x) => $x * 2;

    foreach ($config as $key => $value) {
        echo "$key=$value\n";
    }
    ```

### Comparisons and numeric strings

`0 == "foo"` is now `false`, `in_array("abc", [0])` is now `false`, and arithmetic on non-numeric strings throws `TypeError`. The full table is in [PHP 8.0](php-8.md#saner-comparisons-and-numeric-strings).

Audit loose comparisons (`==`, `in_array()` and `array_search()` without `strict: true`, `switch`) that compare user input against integers, especially in authentication, permission and validation code.

### Internal functions throw instead of returning `null`

Passing the wrong type or an invalid value to a built-in function used to emit a warning and return `null` or `false`. Now it throws:

```php
strlen([]);                    // TypeError: strlen(): Argument #1 ($string) must be of type string, array given
count(5);                      // TypeError: count(): Argument #1 ($value) must be of type Countable|array, int given
array_combine([1], [1, 2]);    // ValueError: array_combine(): Argument #1 ($keys) and argument #2 ($values) must have the same number of elements
```

### Errors and warnings reclassified

| Situation | PHP 7.4 | PHP 8.0 |
|-----------|---------|---------|
| Undefined variable | Notice | Warning |
| Undefined array key | Notice | Warning |
| Reading a property of `null` | Notice | Warning |
| Writing a property of `null` | Warning, `null` becomes `stdClass` | `Error` |
| `1 / 0` | Warning, returns `INF` | `DivisionByZeroError` |
| Arithmetic on arrays, resources or objects (`$obj + 1`) | Notice, warning or fatal error, depending on the operand | `TypeError` |
| Default `error_reporting` | `E_ALL & ~E_NOTICE & ~E_STRICT & ~E_DEPRECATED` | `E_ALL` |

!!! warning "`@` no longer hides fatal errors, and custom error handlers must change"
    Inside an `@`-suppressed call, `error_reporting()` used to return `0`. In PHP 8 it returns `4437` (`E_ERROR | E_CORE_ERROR | E_COMPILE_ERROR | E_USER_ERROR | E_RECOVERABLE_ERROR | E_PARSE`). Error handlers that test `if (error_reporting() === 0) return;` stop working. Use:

    ```php
    set_error_handler(function (int $errno, string $message): bool {
        if (!(error_reporting() & $errno)) {
            return false;   // suppressed with @ or excluded by error_reporting
        }
        throw new ErrorException($message, 0, $errno);
    });
    ```

### Resources became objects

Many extensions now return objects instead of resources: `curl_init()` returns `CurlHandle`, `imagecreate()` returns `GdImage`, `xml_parser_create()` returns `XMLParser`, `socket_create()` returns `Socket`, and OpenSSL keys are `OpenSSLAsymmetricKey`. Code that checks `is_resource()` breaks:

```php
$ch = curl_init($url);
if (!is_resource($ch)) {   // always true in PHP 8: $ch is a CurlHandle object
    throw new RuntimeException('cURL init failed');
}

// Works on every version:
if ($ch === false) {
    throw new RuntimeException('cURL init failed');
}
```

### Other 8.0 changes

- **PDO throws by default.** The default error mode changed from silent to `PDO::ERRMODE_EXCEPTION`. Code that checked `$stmt === false` now gets a `PDOException` instead.
- **Concatenation precedence:** `"Total: " . $a + $b` now means `"Total: " . ($a + $b)`.
- `substr()`, `iconv_substr()` and similar return `""` instead of `false` when the offset is out of range.
- `match` and `mixed` are reserved keywords.
- Sorting is stable: elements that compare equal keep their original order.
- Magic method signatures (`__get(string $name)`, …) and abstract trait method signatures are now validated.

## PHP 8.0 → 8.1

| Change | Example | Fix |
|--------|---------|-----|
| Passing `null` to non-nullable internal parameters is deprecated | `strlen(null)`, `trim(null)`, `htmlspecialchars(null)` | `?? ''`, or fix the source of the `null` |
| Implicit float-to-int conversion losing precision is deprecated | `$list[1.5]` | Cast explicitly: `$list[(int) $index]` |
| Internal methods have tentative return types | Overriding `ArrayAccess::offsetGet()` without a return type emits a deprecation | Add the return type, or `#[\ReturnTypeWillChange]` while you still support 7.x |
| `mysqli` throws by default | Failed queries throw `mysqli_sql_exception` | Catch it, or set `mysqli_report()` explicitly |
| `htmlspecialchars()` escapes single quotes by default | `'` becomes `&#039;` | Usually desirable; update test fixtures |
| `Serializable` without `__serialize()` is deprecated | `class X implements Serializable` | Add `__serialize()` / `__unserialize()` |
| `strftime()`, `gmstrftime()`, `FILTER_SANITIZE_STRING` deprecated | `strftime('%B %Y')` | `IntlDateFormatter`, `DateTimeInterface::format()`, `htmlspecialchars()` |
| `$GLOBALS` can no longer be written as a whole | `$GLOBALS = []` | Write individual keys |

The `null`-to-internal-function deprecation is by far the most common in older code, because PHP 5 and 7 silently treated `null` as `""` or `0`.

## PHP 8.1 → 8.3

| Version | Change | Fix |
|---------|--------|-----|
| 8.2 | Dynamic properties deprecated | Declare the property, use `stdClass` or `__get()`/`__set()`, or add `#[\AllowDynamicProperties]` temporarily |
| 8.2 | `"${var}"` and `"${expr}"` interpolation deprecated | `"{$var}"` |
| 8.2 | `utf8_encode()` / `utf8_decode()` deprecated | `mb_convert_encoding($s, 'UTF-8', 'ISO-8859-1')` |
| 8.2 | Partially supported callables deprecated | `"self::method"`, `"parent::method"`, `[$this, 'parent::method']` become `self::method(...)` |
| 8.2 | `strtolower()` / `strtoupper()` ignore the locale | Use `mb_strtolower()` for locale-aware or non-ASCII text |
| 8.3 | `get_class()` / `get_parent_class()` without arguments deprecated | `static::class`, `self::class`, `parent::class` |
| 8.3 | `++` / `--` on empty strings, and `--` on `null`, warn or are deprecated | Initialize counters as integers; use `str_increment()` for strings |

## PHP 8.3 → 8.6

| Version | Change | Fix |
|---------|--------|-----|
| 8.4 | Implicitly nullable parameters deprecated: `string $s = null` | `?string $s = null` |
| 8.4 | `trigger_error()` with `E_USER_ERROR` deprecated | Throw an exception |
| 8.4 | `E_STRICT` constant deprecated | Remove it from `error_reporting()` expressions |
| 8.4 | IMAP, OCI8, PDO_OCI, Pspell moved to PECL | Install from PECL, or replace (e.g. a Composer IMAP library) |
| 8.5 | Backtick operator deprecated | `shell_exec()` or `proc_open()` |
| 8.5 | `(integer)`, `(boolean)`, `(double)`, `(binary)` deprecated | `(int)`, `(bool)`, `(float)`, `(string)` |
| 8.5 | `++` on non-numeric strings deprecated | `str_increment()` |
| 8.5 | `null` as an array offset or `array_key_exists()` key deprecated | Use `''` explicitly |
| 8.5 | `__sleep()` / `__wakeup()` soft-deprecated | `__serialize()` / `__unserialize()` |
| 8.5 | `curl_close()`, `finfo_close()`, `imagedestroy()`, `xml_parser_free()` deprecated | Delete the calls; objects are freed automatically |
| 8.6 | `return` in `finally`, returning a value from `__construct()` / `__destruct()` deprecated | Restructure the control flow |
| 8.6 | `is_double()`, `is_long()`, `is_integer()`, `doubleval()`, `spl_object_hash()` deprecated | `is_float()`, `is_int()`, `floatval()`, `spl_object_id()` |
| 8.6 | Session defaults: strict mode on, `HttpOnly` on, `SameSite=Lax` | Opt out explicitly in `php.ini` if your app depended on the old defaults |

Most of these can be fixed automatically. See [Upgrade Strategy](upgrading.md#5-automate-the-mechanical-changes-with-rector).
