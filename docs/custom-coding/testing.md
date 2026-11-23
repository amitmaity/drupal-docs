# Testing

Drupal's test suite runs on PHPUnit with three primary suites: **Unit** (isolated), **Kernel** (bootstrapped Drupal, no browser), and **Functional** (full HTTP/browser).

## File structure

```
example_module/
├── phpunit.xml.dist          # Optional module-level config
└── tests/
    └── src/
        ├── Unit/
        │   └── GreetingGeneratorTest.php
        ├── Kernel/
        │   └── GreetingGeneratorKernelTest.php
        └── Functional/
            └── GreetingPageTest.php
```

Run tests from the Drupal root (where `phpunit.xml` lives):

```bash
ddev exec vendor/bin/phpunit -c web/core/phpunit.xml.dist \
  web/modules/custom/example_module/tests/src/Unit/GreetingGeneratorTest.php
```

---

## Unit test

Unit tests mock dependencies and do not bootstrap Drupal.

### `tests/src/Unit/GreetingGeneratorTest.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\Tests\example_module\Unit;

use Drupal\Core\Config\ConfigFactoryInterface;
use Drupal\Core\Config\ImmutableConfig;
use Drupal\Core\StringTranslation\TranslationInterface;
use Drupal\example_module\Service\GreetingGenerator;
use Drupal\Tests\UnitTestCase;
use PHPUnit\Framework\Attributes\CoversClass;
use PHPUnit\Framework\Attributes\Group;

/**
 * Tests the greeting generator.
 */
#[CoversClass(GreetingGenerator::class)]
#[Group('example_module')]
final class GreetingGeneratorTest extends UnitTestCase {

  /**
   * Tests that generate() uses the configured greeting.
   */
  public function testGenerateUsesConfigGreeting(): void {
    $config = $this->createMock(ImmutableConfig::class);
    $config->method('get')->with('default_greeting')->willReturn('Howdy');

    $configFactory = $this->createMock(ConfigFactoryInterface::class);
    $configFactory->method('get')->with('example_module.settings')->willReturn($config);

    $translation = $this->createMock(TranslationInterface::class);
    $translation->method('translate')->willReturnCallback(
      static fn(string $string, array $args = []) => strtr($string, $args),
    );

    $generator = new GreetingGenerator($configFactory, $translation);
    $result = $generator->generate('Amit');

    $this->assertSame('Howdy, Amit!', $result);
  }

}
```

---

## Kernel test

Kernel tests boot a minimal Drupal environment against the test database configured in `SIMPLETEST_DB` (SQLite, MySQL or PostgreSQL), without installing a full site.

### `tests/src/Kernel/GreetingGeneratorKernelTest.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\Tests\example_module\Kernel;

use Drupal\example_module\Service\GreetingGenerator;
use Drupal\KernelTests\KernelTestBase;
use PHPUnit\Framework\Attributes\CoversClass;
use PHPUnit\Framework\Attributes\Group;
use PHPUnit\Framework\Attributes\RunTestsInSeparateProcesses;

/**
 * Tests the greeting generator with installed config.
 */
#[CoversClass(GreetingGenerator::class)]
#[Group('example_module')]
#[RunTestsInSeparateProcesses]
final class GreetingGeneratorKernelTest extends KernelTestBase {

  /**
   * {@inheritdoc}
   */
  protected static $modules = [
    'system',
    'user',
    'example_module',
  ];

  /**
   * {@inheritdoc}
   */
  protected function setUp(): void {
    parent::setUp();

    $this->installConfig(['example_module']);
  }

  /**
   * Tests generate() with the module's default config.
   */
  public function testGenerateWithInstalledConfig(): void {
    /** @var \Drupal\example_module\Service\GreetingGenerator $generator */
    $generator = $this->container->get('example_module.greeting_generator');

    $result = $generator->generate('Tester');
    $this->assertStringContainsString('Tester', $result);
    $this->assertStringContainsString('Hello', $result);
  }

}
```

---

## Kernel test for entity

### `tests/src/Kernel/TaskEntityTest.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\Tests\example_module\Kernel;

use Drupal\example_module\Entity\Task;
use Drupal\KernelTests\Core\Entity\EntityKernelTestBase;
use PHPUnit\Framework\Attributes\Group;
use PHPUnit\Framework\Attributes\RunTestsInSeparateProcesses;

/**
 * Tests the Task entity.
 */
#[Group('example_module')]
#[RunTestsInSeparateProcesses]
final class TaskEntityTest extends EntityKernelTestBase {

  /**
   * {@inheritdoc}
   */
  protected static $modules = ['example_module'];

  /**
   * {@inheritdoc}
   */
  protected function setUp(): void {
    // EntityKernelTestBase already enables user, system, field, text and
    // filter, and installs the user entity schema.
    parent::setUp();
    $this->installEntitySchema('example_task');
  }

  public function testCreateTask(): void {
    $task = Task::create([
      'title' => 'Write tests',
      'status' => FALSE,
    ]);
    $task->save();

    $this->assertNotNull($task->id());
    $this->assertFalse($task->isCompleted());

    $task->setCompleted(TRUE)->save();
    $this->assertTrue($task->isCompleted());
  }

}
```

---

## Functional test

Functional tests make HTTP requests through Drupal's browser harness.

### `tests/src/Functional/GreetingPageTest.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\Tests\example_module\Functional;

use Drupal\Tests\BrowserTestBase;
use PHPUnit\Framework\Attributes\Group;
use PHPUnit\Framework\Attributes\RunTestsInSeparateProcesses;

/**
 * Tests the greeting page.
 */
#[Group('example_module')]
#[RunTestsInSeparateProcesses]
final class GreetingPageTest extends BrowserTestBase {

  /**
   * {@inheritdoc}
   */
  protected static $modules = ['example_module'];

  /**
   * {@inheritdoc}
   */
  protected $defaultTheme = 'stark';

  public function testAnonymousUserDenied(): void {
    $this->drupalGet('/example/greeting');
    $this->assertSession()->statusCodeEquals(403);
  }

  public function testAuthenticatedUserCanView(): void {
    $account = $this->drupalCreateUser(['view example module greeting']);
    $this->drupalLogin($account);

    $this->drupalGet('/example/greeting');
    $this->assertSession()->statusCodeEquals(200);
    $this->assertSession()->pageTextContains('Hello');
  }

}
```

---

## FunctionalJavascript test (optional)

For AJAX interactions, extend `WebDriverTestBase`:

```php
<?php

declare(strict_types=1);

namespace Drupal\Tests\example_module\FunctionalJavascript;

use Drupal\FunctionalJavascriptTests\WebDriverTestBase;
use PHPUnit\Framework\Attributes\Group;
use PHPUnit\Framework\Attributes\RunTestsInSeparateProcesses;

/**
 * Tests the AJAX demo form.
 */
#[Group('example_module')]
#[RunTestsInSeparateProcesses]
final class AjaxDemoFormTest extends WebDriverTestBase {

  protected static $modules = ['example_module'];

  protected $defaultTheme = 'stark';

  public function testCountryAjaxUpdatesCity(): void {
    $this->drupalCreateUser(['access content']);
    $this->drupalLogin($this->rootUser);

    $this->drupalGet('/example/ajax-demo');
    $this->getSession()->getPage()->selectFieldOption('Country', 'us');
    $this->assertSession()->assertWaitOnAjaxRequest();

    $this->assertSession()->optionExists('City', 'nyc');
  }

}
```

Requires ChromeDriver or Selenium in CI.

---

## Run test suites

```bash
# All example_module tests
ddev exec vendor/bin/phpunit -c web/core/phpunit.xml.dist \
  --group example_module

# Unit only
ddev exec vendor/bin/phpunit -c web/core/phpunit.xml.dist \
  web/modules/custom/example_module/tests/src/Unit

# Kernel only
ddev exec vendor/bin/phpunit -c web/core/phpunit.xml.dist \
  web/modules/custom/example_module/tests/src/Kernel

# Functional (slower — installs profile)
ddev exec vendor/bin/phpunit -c web/core/phpunit.xml.dist \
  web/modules/custom/example_module/tests/src/Functional
```

---

## Test base class comparison

| Suite | Base class | Boots Drupal | Database | HTTP |
|-------|-----------|--------------|----------|------|
| Unit | `UnitTestCase` | No | No | No |
| Kernel | `KernelTestBase` | Partial | `SIMPLETEST_DB` | No |
| Functional | `BrowserTestBase` | Full | `SIMPLETEST_DB` | Yes |
| JS Functional | `WebDriverTestBase` | Full | `SIMPLETEST_DB` | Real browser |

!!! tip "Use the smallest suite"
    Prefer Unit tests for pure logic, Kernel for services and entities, Functional only for routes and UI flows.

!!! note "PHPUnit attributes"
    Use PHPUnit attributes (`#[Group('example_module')]`, `#[CoversClass(...)]`) instead of `@group` / `@covers` docblock annotations, which are deprecated in PHPUnit 11 and removed in PHPUnit 12. `#[Group('example_module')]` lets you run all module tests with `--group example_module`.

!!! warning "Process isolation (Drupal 11.3+)"
    Since Drupal 11.3, every concrete Kernel, Functional and FunctionalJavascript test class must declare `#[RunTestsInSeparateProcesses]`; omitting it is deprecated and will throw in Drupal 12.

!!! warning "Functional test isolation"
    Functional tests reinstall Drupal for **each test method**. Group related assertions into fewer methods and keep `$modules` minimal to speed up runs.
