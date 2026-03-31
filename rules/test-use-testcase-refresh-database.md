---
title: Use Laravel TestCase and RefreshDatabase
impact: HIGH
impactDescription: Enables proper isolated testing with fresh database state
tags: [testing, phpunit, testcase, refresh-database, pest]
---

## Use Laravel's TestCase Base Class and RefreshDatabase Trait

Every test in a Laravel application should extend `Tests\TestCase` and use the `RefreshDatabase` trait to guarantee proper test isolation. Laravel's TestCase bootstraps the entire application, giving tests access to the service container, helpers, and testing utilities. The `RefreshDatabase` trait ensures each test starts with a clean, migrated database by wrapping test execution in a transaction that is rolled back afterward (or by re-migrating when necessary).

Without these, tests lack access to Laravel's testing infrastructure, cannot use factories or assertions like `assertDatabaseHas()`, and risk leaking state between tests.

For test suites where migration speed is a concern, `DatabaseTransactions` can be used instead of `RefreshDatabase` when the database schema is already up to date (e.g., in CI environments where migrations run once before the suite).

**Incorrect**

```php
<?php

// tests/Unit/UserServiceTest.php
// BAD: Using raw PHPUnit without Laravel bootstrapping

use PHPUnit\Framework\TestCase;

class UserServiceTest extends TestCase
{
    protected function setUp(): void
    {
        parent::setUp();

        // Manual database connection - fragile and error-prone
        $this->pdo = new PDO('mysql:host=localhost;dbname=test_db', 'root', '');
        $this->pdo->beginTransaction();
    }

    protected function tearDown(): void
    {
        // Manual cleanup - easy to forget, leaks state on failure
        $this->pdo->rollBack();
        parent::tearDown();
    }

    public function test_user_can_be_created(): void
    {
        // Manual SQL insertion - no factory support, no model events
        $this->pdo->exec(
            "INSERT INTO users (name, email, password) VALUES ('John', 'john@example.com', 'hashed')"
        );

        $stmt = $this->pdo->query("SELECT * FROM users WHERE email = 'john@example.com'");
        $user = $stmt->fetch(PDO::FETCH_ASSOC);

        $this->assertEquals('John', $user['name']);
    }

    public function test_user_requires_authentication(): void
    {
        // No way to simulate authentication without Laravel's auth system
        // Manual session/token handling is brittle
        $_SESSION['user_id'] = 1;
        $controller = new UserController();
        $result = $controller->profile();

        $this->assertNotNull($result);
    }
}
```

**Correct**

```php
<?php

// tests/Feature/UserServiceTest.php (PHPUnit style)

namespace Tests\Feature;

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class UserServiceTest extends TestCase
{
    use RefreshDatabase;

    public function test_user_can_be_created(): void
    {
        $user = User::factory()->create([
            'name' => 'John Doe',
            'email' => 'john@example.com',
        ]);

        $this->assertDatabaseHas('users', [
            'email' => 'john@example.com',
            'name' => 'John Doe',
        ]);

        $this->assertInstanceOf(User::class, $user);
        $this->assertEquals('John Doe', $user->name);
    }

    public function test_user_can_access_profile_when_authenticated(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)
            ->getJson('/api/user/profile');

        $response->assertOk()
            ->assertJson([
                'id' => $user->id,
                'name' => $user->name,
                'email' => $user->email,
            ]);
    }

    public function test_guest_cannot_access_profile(): void
    {
        $response = $this->getJson('/api/user/profile');

        $response->assertUnauthorized();
    }
}
```

```php
<?php

// tests/Feature/UserServiceTest.php (Pest style)

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

it('can create a user', function () {
    $user = User::factory()->create([
        'name' => 'John Doe',
        'email' => 'john@example.com',
    ]);

    $this->assertDatabaseHas('users', [
        'email' => 'john@example.com',
        'name' => 'John Doe',
    ]);

    expect($user)
        ->toBeInstanceOf(User::class)
        ->name->toBe('John Doe');
});

it('allows authenticated users to access their profile', function () {
    $user = User::factory()->create();

    $response = $this->actingAs($user)
        ->getJson('/api/user/profile');

    $response->assertOk()
        ->assertJson([
            'id' => $user->id,
            'name' => $user->name,
        ]);
});

it('rejects guest access to profile', function () {
    $this->getJson('/api/user/profile')
        ->assertUnauthorized();
});
```

```php
<?php

// tests/Feature/ReportGenerationTest.php
// Using DatabaseTransactions for faster tests when schema is already migrated

namespace Tests\Feature;

use App\Models\User;
use App\Models\Report;
use Illuminate\Foundation\Testing\DatabaseTransactions;
use Tests\TestCase;

class ReportGenerationTest extends TestCase
{
    // DatabaseTransactions wraps each test in a transaction and rolls back.
    // Faster than RefreshDatabase because it skips migration checks,
    // but requires the database schema to already be up to date.
    use DatabaseTransactions;

    public function test_user_can_generate_report(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)
            ->postJson('/api/reports', [
                'title' => 'Monthly Sales',
                'type' => 'sales',
                'date_range' => ['2026-01-01', '2026-01-31'],
            ]);

        $response->assertCreated();

        $this->assertDatabaseHas('reports', [
            'user_id' => $user->id,
            'title' => 'Monthly Sales',
        ]);
    }
}
```

Reference: [Laravel Testing - Getting Started](https://laravel.com/docs/testing)
