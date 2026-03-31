---
title: Use API Versioning for Breaking Changes
impact: MEDIUM
impactDescription: Unversioned APIs break existing clients on updates
tags: api, versioning, routes, backwards-compatibility
---

## Use API Versioning for Breaking Changes

Without versioning, every structural change to your API risks breaking existing consumers. Mobile apps, third-party integrations, and SPAs all depend on a stable contract. API versioning lets you evolve your endpoints while giving clients time to migrate. URL prefix versioning is the most common and most explicit strategy in the Laravel ecosystem.

**Incorrect**

```php
// routes/api.php - single unversioned route file
// Any breaking change here immediately affects every client
use App\Http\Controllers\UserController;
use App\Http\Controllers\OrderController;

Route::apiResource('users', UserController::class);
Route::apiResource('orders', OrderController::class);
Route::get('users/{user}/orders', [UserController::class, 'orders']);

// When you need to rename a field, change a response shape, or remove
// an endpoint, every consumer breaks simultaneously with no migration path.
```

**Correct**

```php
// routes/api_v1.php
use App\Http\Controllers\Api\V1\UserController;
use App\Http\Controllers\Api\V1\OrderController;

Route::prefix('v1')->as('v1.')->group(function () {
    Route::apiResource('users', UserController::class);
    Route::apiResource('orders', OrderController::class);
    Route::get('users/{user}/orders', [UserController::class, 'orders']);
});

// routes/api_v2.php
use App\Http\Controllers\Api\V2\UserController;
use App\Http\Controllers\Api\V2\OrderController;

Route::prefix('v2')->as('v2.')->group(function () {
    Route::apiResource('users', UserController::class);
    Route::apiResource('orders', OrderController::class);
    // V2 introduces a dedicated nested resource instead of a custom route
    Route::apiResource('users.orders', \App\Http\Controllers\Api\V2\UserOrderController::class)
        ->shallow();
});

// bootstrap/app.php (Laravel 11+)
return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        api: __DIR__ . '/../routes/api.php',
        then: function () {
            Route::middleware('api')
                ->prefix('api')
                ->group(base_path('routes/api_v1.php'));

            Route::middleware('api')
                ->prefix('api')
                ->group(base_path('routes/api_v2.php'));
        },
    )
    ->create();

// app/Http/Controllers/Api/V1/UserController.php
namespace App\Http\Controllers\Api\V1;

use App\Http\Controllers\Controller;
use App\Http\Resources\V1\UserResource;
use App\Models\User;

class UserController extends Controller
{
    public function show(User $user)
    {
        return new UserResource($user->load('role'));
    }
}

// app/Http/Resources/V1/UserResource.php
namespace App\Http\Resources\V1;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,           // V1: single name field
            'email' => $this->email,
            'role' => $this->whenLoaded('role', fn () => $this->role->name),
            'created_at' => $this->created_at->toIso8601String(),
        ];
    }
}

// app/Http/Controllers/Api/V2/UserController.php
namespace App\Http\Controllers\Api\V2;

use App\Http\Controllers\Controller;
use App\Http\Resources\V2\UserResource;
use App\Models\User;

class UserController extends Controller
{
    public function show(User $user)
    {
        return new UserResource($user->load('role'));
    }
}

// app/Http/Resources/V2/UserResource.php
namespace App\Http\Resources\V2;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'first_name' => $this->first_name,  // V2: split name into parts
            'last_name' => $this->last_name,
            'email' => $this->email,
            'role' => new RoleResource($this->whenLoaded('role')),
            'created_at' => $this->created_at->toIso8601String(),
            'updated_at' => $this->updated_at->toIso8601String(),
        ];
    }
}

// --- Header-based versioning alternative ---
// app/Http/Middleware/ApiVersionFromHeader.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class ApiVersionFromHeader
{
    /**
     * Resolve API version from Accept header.
     * Example: Accept: application/vnd.myapp.v2+json
     */
    public function handle(Request $request, Closure $next): Response
    {
        $accept = $request->header('Accept', '');

        preg_match('/application\/vnd\.myapp\.v(\d+)\+json/', $accept, $matches);

        $version = isset($matches[1]) ? (int) $matches[1] : 1;

        $request->attributes->set('api_version', $version);

        return $next($request);
    }
}

// --- Deprecation middleware for sunsetting old versions ---
// app/Http/Middleware/DeprecateApiVersion.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class DeprecateApiVersion
{
    /**
     * Add deprecation headers to warn clients about upcoming removal.
     * Apply to old version route groups: ->middleware(DeprecateApiVersion::class.':2026-06-01')
     */
    public function handle(Request $request, Closure $next, string $sunsetDate): Response
    {
        $response = $next($request);

        $response->headers->set('Deprecation', 'true');
        $response->headers->set('Sunset', $sunsetDate);
        $response->headers->set(
            'Link',
            '<https://api.example.com/docs/migration>; rel="deprecation"; type="text/html"'
        );

        return $response;
    }
}

// Apply deprecation to V1 routes
// routes/api_v1.php
Route::prefix('v1')
    ->as('v1.')
    ->middleware(DeprecateApiVersion::class . ':2026-06-01')
    ->group(function () {
        Route::apiResource('users', V1\UserController::class);
    });
```

Reference: [Laravel Routing](https://laravel.com/docs/routing)
