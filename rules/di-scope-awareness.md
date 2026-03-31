---
title: Understand Provider Scopes
impact: MEDIUM-HIGH
impactDescription: Wrong scope causes subtle bugs in long-running processes
tags:
  - dependency-injection
  - singleton
  - bind
  - scoped
  - service-container
---

## Understand Provider Scopes

Laravel's service container offers three binding scopes: `bind()` creates a new instance every time the service is resolved, `singleton()` creates one shared instance for the entire application lifecycle, and `scoped()` creates one instance per request lifecycle (reset between requests). Choosing the wrong scope leads to subtle bugs that are especially dangerous in long-running processes like Laravel Octane, queue workers, and Reverb. A singleton that caches request-specific data will leak state between requests under Octane. A bind where a singleton is needed may cause unnecessary object creation and inconsistent state.

**Incorrect**

```php
<?php

namespace App\Providers;

use App\Services\AuthContext;
use App\Services\ReportGenerator;
use App\Services\ShoppingCart;
use App\Services\DatabaseConnectionPool;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // WRONG: Singleton for request-specific data
        // Under Octane, the authenticated user from request #1
        // will leak into request #2
        $this->app->singleton(AuthContext::class, function ($app) {
            return new AuthContext(
                user: $app->make('auth')->user(),  // Captured once, stale forever
                ipAddress: $app->make('request')->ip(),
            );
        });

        // WRONG: Singleton for stateful per-request object
        // Shopping cart items from one user bleed into the next request
        $this->app->singleton(ShoppingCart::class, function ($app) {
            return new ShoppingCart();
        });

        // WRONG: bind() for expensive shared resource
        // Creates a new connection pool on every resolution - wasteful and dangerous
        $this->app->bind(DatabaseConnectionPool::class, function ($app) {
            return new DatabaseConnectionPool(
                host: config('database.connections.mysql.host'),
                maxConnections: 10,
            );
        });

        // WRONG: bind() when consistency within a request matters
        // Different parts of the same request get different instances
        // with potentially different state
        $this->app->bind(ReportGenerator::class, function ($app) {
            return new ReportGenerator(
                startedAt: now(),  // Different timestamp each resolution
                requestId: Str::uuid(),  // Different ID each resolution
            );
        });
    }
}
```

**Correct**

```php
<?php

namespace App\Providers;

use App\Contracts\ConnectionPoolInterface;
use App\Services\AuthContext;
use App\Services\DatabaseConnectionPool;
use App\Services\ReportGenerator;
use App\Services\ShoppingCart;
use App\Services\TemporaryFileManager;
use Illuminate\Support\ServiceProvider;
use Illuminate\Support\Str;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // scoped() - Reset between requests, safe for Octane
        // Perfect for request-specific data that should be consistent
        // within a single request but fresh for each new request
        $this->app->scoped(AuthContext::class, function ($app) {
            return new AuthContext(
                user: $app->make('auth')->user(),
                ipAddress: $app->make('request')->ip(),
            );
        });

        // scoped() - Per-request stateful object
        // Each request gets its own cart, automatically cleared between requests
        $this->app->scoped(ShoppingCart::class, function ($app) {
            return new ShoppingCart();
        });

        // singleton() - Shared expensive resource that is stateless
        // Connection pool is thread-safe and should be reused across requests
        $this->app->singleton(ConnectionPoolInterface::class, function ($app) {
            return new DatabaseConnectionPool(
                host: config('database.connections.mysql.host'),
                maxConnections: 10,
            );
        });

        // scoped() - Consistent identity within a request
        // Same instance returned throughout the request lifecycle,
        // so requestId and startedAt remain consistent
        $this->app->scoped(ReportGenerator::class, function ($app) {
            return new ReportGenerator(
                startedAt: now(),
                requestId: Str::uuid()->toString(),
            );
        });

        // bind() - Truly stateless, independent instances needed each time
        // Each resolution should be a fresh, isolated object
        $this->app->bind(TemporaryFileManager::class, function ($app) {
            return new TemporaryFileManager(
                directory: storage_path('app/temp/' . Str::random(16)),
            );
        });
    }
}

// Quick reference for choosing the right scope:
//
// singleton() - Use when:
//   - The object is expensive to create (connection pools, HTTP clients)
//   - The object holds NO request-specific state
//   - The object is thread-safe / stateless between requests
//   - Examples: config readers, HTTP client pools, third-party SDK clients
//
// scoped() - Use when:
//   - The object holds request-specific state (auth, cart, request ID)
//   - You need the same instance within a request but a fresh one per request
//   - Running under Octane, Reverb, or any long-lived process
//   - Examples: auth context, shopping cart, request loggers, tenant context
//
// bind() - Use when:
//   - Every resolution should produce an independent instance
//   - The object is lightweight and short-lived
//   - No shared state is needed between resolutions
//   - Examples: DTOs, value objects, temp file handlers, one-off builders

// Octane-specific considerations:
// In config/octane.php, list singletons that need flushing between requests:
//
// 'flush' => [
//     AuthContext::class,
//     ShoppingCart::class,
// ],
//
// Or better yet, use scoped() so Laravel handles it automatically.
```

Reference: [Laravel Service Container](https://laravel.com/docs/container)
