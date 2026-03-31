---
title: Honor Liskov Substitution Principle
impact: HIGH
impactDescription: Ensures implementations are truly interchangeable
tags:
  - dependency-injection
  - solid
  - liskov
  - substitution
---

## Honor Liskov Substitution Principle

Any implementation of an interface or subclass of a base class must be fully substitutable for it without altering the correctness of the program. If swapping one implementation for another causes exceptions, changed return semantics, or violated preconditions, the Liskov Substitution Principle (LSP) is broken. This makes polymorphism unreliable and defeats the purpose of coding against abstractions.

**Incorrect**

```php
<?php

namespace App\Contracts;

use App\Models\User;
use Illuminate\Support\Collection;

interface UserRepositoryInterface
{
    public function find(int $id): ?User;
    public function all(): Collection;
    public function create(array $attributes): User;
    public function delete(int $id): bool;
}

namespace App\Repositories;

use App\Contracts\UserRepositoryInterface;
use App\Models\User;
use Illuminate\Support\Collection;

class EloquentUserRepository implements UserRepositoryInterface
{
    public function find(int $id): ?User
    {
        return User::find($id);
    }

    public function all(): Collection
    {
        return User::all();
    }

    public function create(array $attributes): User
    {
        return User::create($attributes);
    }

    public function delete(int $id): bool
    {
        return User::findOrFail($id)->delete();
    }
}

// Violates LSP - changes the behavior contract of the interface
class CacheUserRepository extends EloquentUserRepository
{
    public function __construct(
        private readonly CacheManager $cache,
    ) {}

    public function find(int $id): ?User
    {
        // Strengthened postcondition: throws instead of returning null
        // The interface says ?User (nullable), but this never returns null
        return $this->cache->remember("user.{$id}", 3600, function () use ($id) {
            $user = parent::find($id);
            if ($user === null) {
                // Violates contract - callers expect null, not an exception
                throw new \RuntimeException("User {$id} not found in cache layer");
            }
            return $user;
        });
    }

    public function create(array $attributes): User
    {
        // Weakened postcondition: silently skips cache invalidation
        // Callers relying on consistent state will get stale data
        return parent::create($attributes);
    }

    public function delete(int $id): bool
    {
        // Violates contract - refuses to perform the operation
        throw new \BadMethodCallException(
            'CacheUserRepository does not support deletion. Use EloquentUserRepository.'
        );
    }

    public function all(): Collection
    {
        // Changed semantics: returns only cached users, not all users
        // Subtly different behavior that breaks callers expecting complete data
        $cachedKeys = $this->cache->get('user.all.keys', []);
        return collect($cachedKeys)
            ->map(fn (int $id) => $this->cache->get("user.{$id}"))
            ->filter();
    }
}
```

**Correct**

```php
<?php

namespace App\Contracts;

use App\Models\User;
use Illuminate\Support\Collection;

interface UserRepositoryInterface
{
    /**
     * Find a user by ID.
     *
     * @return User|null Returns null when user does not exist.
     */
    public function find(int $id): ?User;

    /**
     * Retrieve all users.
     *
     * @return Collection Complete collection of all users.
     */
    public function all(): Collection;

    /**
     * Create a new user.
     *
     * @return User The persisted user instance.
     */
    public function create(array $attributes): User;

    /**
     * Delete a user by ID.
     *
     * @return bool True if deletion succeeded.
     * @throws \Illuminate\Database\Eloquent\ModelNotFoundException
     */
    public function delete(int $id): bool;
}

namespace App\Repositories;

use App\Contracts\UserRepositoryInterface;
use App\Models\User;
use Illuminate\Support\Collection;

class EloquentUserRepository implements UserRepositoryInterface
{
    public function find(int $id): ?User
    {
        return User::find($id);
    }

    public function all(): Collection
    {
        return User::all();
    }

    public function create(array $attributes): User
    {
        return User::create($attributes);
    }

    public function delete(int $id): bool
    {
        return User::findOrFail($id)->delete();
    }
}

// Honors LSP - decorates without changing behavioral contract
class CachingUserRepository implements UserRepositoryInterface
{
    public function __construct(
        private readonly UserRepositoryInterface $inner,
        private readonly CacheManager $cache,
        private readonly int $ttl = 3600,
    ) {}

    public function find(int $id): ?User
    {
        // Preserves nullable return - null is cached as a valid result
        return $this->cache->remember(
            "user.{$id}",
            $this->ttl,
            fn () => $this->inner->find($id)
        );
    }

    public function all(): Collection
    {
        // Returns the complete collection, just from cache when available
        return $this->cache->remember(
            'users.all',
            $this->ttl,
            fn () => $this->inner->all()
        );
    }

    public function create(array $attributes): User
    {
        $user = $this->inner->create($attributes);

        // Invalidate relevant caches to maintain consistency
        $this->cache->forget('users.all');
        $this->cache->put("user.{$user->id}", $user, $this->ttl);

        return $user;
    }

    public function delete(int $id): bool
    {
        // Honors the full contract - delegates then cleans up cache
        $result = $this->inner->delete($id);

        $this->cache->forget("user.{$id}");
        $this->cache->forget('users.all');

        return $result;
    }
}

// Service provider wires the decorator chain
namespace App\Providers;

use App\Contracts\UserRepositoryInterface;
use App\Repositories\CachingUserRepository;
use App\Repositories\EloquentUserRepository;
use Illuminate\Support\ServiceProvider;

class RepositoryServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind(UserRepositoryInterface::class, function ($app) {
            return new CachingUserRepository(
                inner: new EloquentUserRepository(),
                cache: $app->make('cache.store'),
            );
        });
    }
}

// Any consumer can rely on consistent behavior regardless of implementation
class UserController extends Controller
{
    public function __construct(
        private readonly UserRepositoryInterface $users,
    ) {}

    public function show(int $id): JsonResponse
    {
        // Works identically with EloquentUserRepository or CachingUserRepository
        $user = $this->users->find($id);

        if ($user === null) {
            abort(404, 'User not found');
        }

        return response()->json($user);
    }

    public function destroy(int $id): JsonResponse
    {
        // No need to check which implementation is behind the interface
        $this->users->delete($id);
        return response()->json(['message' => 'User deleted']);
    }
}
```

Reference: [Laravel Service Container](https://laravel.com/docs/container)
