---
title: Apply Interface Segregation Principle
impact: HIGH
impactDescription: Prevents bloated interfaces that force unnecessary implementations
tags:
  - dependency-injection
  - solid
  - interface-segregation
  - contracts
---

## Apply Interface Segregation Principle

No client should be forced to depend on methods it does not use. When an interface grows too large, implementing classes are burdened with methods they do not need, leading to empty stubs, thrown exceptions for unsupported operations, or tightly coupled code. Split large interfaces into smaller, focused ones so that each consumer depends only on the behavior it actually requires.

**Incorrect**

```php
<?php

namespace App\Contracts;

use App\Models\User;
use Illuminate\Support\Collection;

// Fat interface - forces every implementation to handle all concerns
interface UserRepositoryInterface
{
    public function find(int $id): ?User;
    public function findByEmail(string $email): ?User;
    public function findByUsername(string $username): ?User;
    public function all(): Collection;
    public function paginate(int $perPage = 15): LengthAwarePaginator;
    public function create(array $attributes): User;
    public function update(int $id, array $attributes): User;
    public function delete(int $id): bool;
    public function forceDelete(int $id): bool;
    public function restore(int $id): bool;
    public function search(string $query): Collection;
    public function searchByFilters(array $filters): Collection;
    public function reindex(): void;
    public function export(string $format): string;
    public function import(string $data, string $format): int;
}

// A read-only API consumer is forced to implement write methods it never uses
class ApiUserRepository implements UserRepositoryInterface
{
    public function find(int $id): ?User { /* ... */ }
    public function findByEmail(string $email): ?User { /* ... */ }
    public function findByUsername(string $username): ?User { /* ... */ }
    public function all(): Collection { /* ... */ }
    public function paginate(int $perPage = 15): LengthAwarePaginator { /* ... */ }

    // Forced to stub out methods that make no sense for this implementation
    public function create(array $attributes): User
    {
        throw new \BadMethodCallException('Not supported');
    }

    public function update(int $id, array $attributes): User
    {
        throw new \BadMethodCallException('Not supported');
    }

    public function delete(int $id): bool
    {
        throw new \BadMethodCallException('Not supported');
    }

    public function forceDelete(int $id): bool
    {
        throw new \BadMethodCallException('Not supported');
    }

    public function restore(int $id): bool
    {
        throw new \BadMethodCallException('Not supported');
    }

    public function search(string $query): Collection
    {
        throw new \BadMethodCallException('Not supported');
    }

    public function searchByFilters(array $filters): Collection
    {
        throw new \BadMethodCallException('Not supported');
    }

    public function reindex(): void
    {
        throw new \BadMethodCallException('Not supported');
    }

    public function export(string $format): string
    {
        throw new \BadMethodCallException('Not supported');
    }

    public function import(string $data, string $format): int
    {
        throw new \BadMethodCallException('Not supported');
    }
}
```

**Correct**

```php
<?php

namespace App\Contracts;

use App\Models\User;
use Illuminate\Support\Collection;
use Illuminate\Pagination\LengthAwarePaginator;

// Focused interface for reading users
interface UserReadableInterface
{
    public function find(int $id): ?User;
    public function findByEmail(string $email): ?User;
    public function findByUsername(string $username): ?User;
    public function all(): Collection;
    public function paginate(int $perPage = 15): LengthAwarePaginator;
}

// Focused interface for writing users
interface UserWritableInterface
{
    public function create(array $attributes): User;
    public function update(int $id, array $attributes): User;
    public function delete(int $id): bool;
    public function forceDelete(int $id): bool;
    public function restore(int $id): bool;
}

// Focused interface for searching users
interface UserSearchableInterface
{
    public function search(string $query): Collection;
    public function searchByFilters(array $filters): Collection;
    public function reindex(): void;
}

// Focused interface for import/export
interface UserTransferableInterface
{
    public function export(string $format): string;
    public function import(string $data, string $format): int;
}

// Full Eloquent implementation composes all interfaces it supports
class EloquentUserRepository implements
    UserReadableInterface,
    UserWritableInterface,
    UserSearchableInterface,
    UserTransferableInterface
{
    public function find(int $id): ?User
    {
        return User::find($id);
    }

    public function findByEmail(string $email): ?User
    {
        return User::where('email', $email)->first();
    }

    public function findByUsername(string $username): ?User
    {
        return User::where('username', $username)->first();
    }

    public function all(): Collection
    {
        return User::all();
    }

    public function paginate(int $perPage = 15): LengthAwarePaginator
    {
        return User::paginate($perPage);
    }

    public function create(array $attributes): User
    {
        return User::create($attributes);
    }

    public function update(int $id, array $attributes): User
    {
        $user = User::findOrFail($id);
        $user->update($attributes);
        return $user->fresh();
    }

    public function delete(int $id): bool
    {
        return User::findOrFail($id)->delete();
    }

    public function forceDelete(int $id): bool
    {
        return User::withTrashed()->findOrFail($id)->forceDelete();
    }

    public function restore(int $id): bool
    {
        return User::withTrashed()->findOrFail($id)->restore();
    }

    public function search(string $query): Collection
    {
        return User::where('name', 'like', "%{$query}%")
            ->orWhere('email', 'like', "%{$query}%")
            ->get();
    }

    public function searchByFilters(array $filters): Collection
    {
        $query = User::query();
        foreach ($filters as $field => $value) {
            $query->where($field, $value);
        }
        return $query->get();
    }

    public function reindex(): void
    {
        User::all()->searchable();
    }

    public function export(string $format): string { /* ... */ }
    public function import(string $data, string $format): int { /* ... */ }
}

// Read-only consumer only implements what it needs - no stubs
class ApiUserRepository implements UserReadableInterface
{
    public function __construct(
        private readonly HttpClient $client,
    ) {}

    public function find(int $id): ?User { /* ... */ }
    public function findByEmail(string $email): ?User { /* ... */ }
    public function findByUsername(string $username): ?User { /* ... */ }
    public function all(): Collection { /* ... */ }
    public function paginate(int $perPage = 15): LengthAwarePaginator { /* ... */ }
}

// Controllers type-hint only the interface they need
class UserProfileController extends Controller
{
    public function __construct(
        private readonly UserReadableInterface $users,
    ) {}

    public function show(int $id): JsonResponse
    {
        $user = $this->users->find($id);
        return response()->json($user);
    }
}

class UserAdminController extends Controller
{
    public function __construct(
        private readonly UserReadableInterface $reader,
        private readonly UserWritableInterface $writer,
    ) {}

    public function update(Request $request, int $id): JsonResponse
    {
        $user = $this->writer->update($id, $request->validated());
        return response()->json($user);
    }
}
```

Reference: [Laravel Contracts](https://laravel.com/docs/contracts)
