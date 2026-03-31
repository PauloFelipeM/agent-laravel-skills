---
title: Use Repository Pattern for Data Access
impact: HIGH
impactDescription: Enables testability and database abstraction
tags: [architecture, repository, eloquent, data-access]
---

## Use Repository Pattern for Data Access

Abstract Eloquent queries behind repository interfaces instead of scattering query logic across controllers, services, and jobs. This makes your code testable (you can mock the repository in unit tests), keeps query logic in one place, and allows you to swap the data source without touching business logic.

**Incorrect**

```php
// app/Http/Controllers/UserController.php
namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class UserController extends Controller
{
    public function index(Request $request): JsonResponse
    {
        // Eloquent queries scattered directly in the controller
        $users = User::query()
            ->when($request->search, function ($query, $search) {
                $query->where('name', 'like', "%{$search}%")
                      ->orWhere('email', 'like', "%{$search}%");
            })
            ->when($request->role, function ($query, $role) {
                $query->whereHas('roles', fn ($q) => $q->where('slug', $role));
            })
            ->when($request->status === 'active', function ($query) {
                $query->whereNotNull('email_verified_at')
                      ->where('is_active', true);
            })
            ->with(['roles', 'department'])
            ->orderBy($request->sort_by ?? 'created_at', $request->sort_dir ?? 'desc')
            ->paginate($request->per_page ?? 15);

        return response()->json($users);
    }

    public function store(Request $request): JsonResponse
    {
        // Direct model creation in the controller
        $user = User::create($request->validated());
        $user->roles()->sync($request->input('role_ids', []));

        return response()->json($user->load('roles'), 201);
    }

    public function findActiveByDepartment(int $departmentId): JsonResponse
    {
        // Same query patterns duplicated across multiple controllers/services
        $users = User::where('department_id', $departmentId)
            ->whereNotNull('email_verified_at')
            ->where('is_active', true)
            ->with('roles')
            ->get();

        return response()->json($users);
    }
}

// Problems:
// - Query logic duplicated across controllers, services, jobs
// - Cannot unit-test business logic without hitting the database
// - Changing the "active user" definition requires updating dozens of files
// - Controller is doing data-access work instead of HTTP concerns
```

**Correct**

```php
// Step 1: Define the interface

// app/Domains/User/Repositories/UserRepositoryInterface.php
namespace App\Domains\User\Repositories;

use App\Domains\User\Models\User;
use Illuminate\Contracts\Pagination\LengthAwarePaginator;
use Illuminate\Database\Eloquent\Collection;

interface UserRepositoryInterface
{
    public function findOrFail(int $id): User;

    public function search(array $filters, int $perPage = 15): LengthAwarePaginator;

    public function findActiveByDepartment(int $departmentId): Collection;

    public function create(array $attributes): User;

    public function update(User $user, array $attributes): User;

    public function syncRoles(User $user, array $roleIds): void;
}

// Step 2: Implement with Eloquent

// app/Domains/User/Repositories/EloquentUserRepository.php
namespace App\Domains\User\Repositories;

use App\Domains\User\Models\User;
use Illuminate\Contracts\Pagination\LengthAwarePaginator;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Collection;

class EloquentUserRepository implements UserRepositoryInterface
{
    public function findOrFail(int $id): User
    {
        return User::with(['roles', 'department'])->findOrFail($id);
    }

    public function search(array $filters, int $perPage = 15): LengthAwarePaginator
    {
        return User::query()
            ->when($filters['search'] ?? null, function (Builder $query, string $search) {
                $query->where(function (Builder $q) use ($search) {
                    $q->where('name', 'like', "%{$search}%")
                      ->orWhere('email', 'like', "%{$search}%");
                });
            })
            ->when($filters['role'] ?? null, function (Builder $query, string $role) {
                $query->whereHas('roles', fn (Builder $q) => $q->where('slug', $role));
            })
            ->when(($filters['status'] ?? null) === 'active', function (Builder $query) {
                $this->scopeActive($query);
            })
            ->with(['roles', 'department'])
            ->orderBy(
                $filters['sort_by'] ?? 'created_at',
                $filters['sort_dir'] ?? 'desc',
            )
            ->paginate($perPage);
    }

    public function findActiveByDepartment(int $departmentId): Collection
    {
        return User::where('department_id', $departmentId)
            ->where(fn (Builder $query) => $this->scopeActive($query))
            ->with('roles')
            ->get();
    }

    public function create(array $attributes): User
    {
        return User::create($attributes);
    }

    public function update(User $user, array $attributes): User
    {
        $user->update($attributes);

        return $user->refresh();
    }

    public function syncRoles(User $user, array $roleIds): void
    {
        $user->roles()->sync($roleIds);
    }

    /**
     * Reusable "active user" scope -- defined once, used everywhere.
     */
    private function scopeActive(Builder $query): Builder
    {
        return $query->whereNotNull('email_verified_at')
                     ->where('is_active', true);
    }
}

// Step 3: Bind in the Service Provider

// app/Domains/User/UserServiceProvider.php
namespace App\Domains\User;

use App\Domains\User\Repositories\EloquentUserRepository;
use App\Domains\User\Repositories\UserRepositoryInterface;
use Illuminate\Support\ServiceProvider;

class UserServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind(UserRepositoryInterface::class, EloquentUserRepository::class);
    }
}

// Step 4: Use the interface in services and controllers

// app/Domains/User/Services/UserService.php
namespace App\Domains\User\Services;

use App\Domains\User\Models\User;
use App\Domains\User\Repositories\UserRepositoryInterface;
use App\Domains\User\Events\UserRegistered;

class UserService
{
    public function __construct(
        private readonly UserRepositoryInterface $users,
    ) {}

    public function register(array $data, array $roleIds = []): User
    {
        $user = $this->users->create($data);

        if ($roleIds) {
            $this->users->syncRoles($user, $roleIds);
        }

        UserRegistered::dispatch($user);

        return $user;
    }
}

// app/Domains/User/Controllers/UserController.php
namespace App\Domains\User\Controllers;

use App\Domains\User\Repositories\UserRepositoryInterface;
use App\Domains\User\Requests\IndexUsersRequest;
use Illuminate\Http\JsonResponse;
use Illuminate\Routing\Controller;

class UserController extends Controller
{
    public function __construct(
        private readonly UserRepositoryInterface $users,
    ) {}

    public function index(IndexUsersRequest $request): JsonResponse
    {
        $result = $this->users->search(
            filters: $request->validated(),
            perPage: $request->integer('per_page', 15),
        );

        return response()->json($result);
    }
}

// In tests, mock the repository easily:
//
// $mock = $this->mock(UserRepositoryInterface::class);
// $mock->shouldReceive('search')
//      ->with(['search' => 'jane'], 15)
//      ->andReturn(new LengthAwarePaginator([$fakeUser], 1, 15));
```

Reference: [Laravel Service Container](https://laravel.com/docs/container)
