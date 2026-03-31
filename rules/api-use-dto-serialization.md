---
title: Use API Resources for Response Serialization
impact: MEDIUM-HIGH
impactDescription: Inconsistent API responses break client applications
tags: api, resources, serialization, dto, responses
---

## Use API Resources for Response Serialization

Returning raw Eloquent models or hand-crafted arrays from controllers leads to inconsistent response structures, accidental exposure of sensitive fields, and tightly coupled clients. Laravel API Resources provide a dedicated transformation layer between your models and the JSON responses delivered to consumers. They give you full control over which fields are included, how relationships are nested, and how pagination metadata is shaped.

**Incorrect**

```php
// Returning raw models exposes all attributes including hidden internals
class UserController extends Controller
{
    public function index()
    {
        // Leaks database columns, timestamps, pivot data, and any appended attributes
        return User::all();
    }

    public function show(User $user)
    {
        // Hand-crafted arrays are fragile and inconsistent across endpoints
        return response()->json([
            'id' => $user->id,
            'name' => $user->name,
            'email' => $user->email,
            'created' => $user->created_at->format('Y-m-d'),
            // Easy to forget fields or format them differently elsewhere
        ]);
    }

    public function store(Request $request)
    {
        $user = User::create($request->all());

        // Different endpoints return different shapes for the same entity
        return response()->json([
            'success' => true,
            'data' => $user->toArray(),
        ], 201);
    }
}
```

**Correct**

```php
// app/Http/Resources/UserResource.php
namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'avatar_url' => $this->avatar_url,
            'created_at' => $this->created_at->toIso8601String(),

            // Conditional attributes - only included when explicitly loaded
            'posts_count' => $this->when($this->posts_count !== null, $this->posts_count),
            'email_verified' => $this->when($request->user()?->isAdmin(), $this->hasVerifiedEmail()),

            // Merge a set of attributes conditionally
            $this->mergeWhen($request->user()?->is($this->resource), [
                'phone' => $this->phone,
                'two_factor_enabled' => $this->two_factor_confirmed_at !== null,
            ]),

            // Conditional relationships - only serialized when loaded on the model
            'posts' => PostResource::collection($this->whenLoaded('posts')),
            'role' => new RoleResource($this->whenLoaded('role')),

            // Links for HATEOAS-style responses
            'links' => [
                'self' => route('api.users.show', $this->id),
                'posts' => route('api.users.posts.index', $this->id),
            ],
        ];
    }
}

// app/Http/Resources/UserCollection.php
namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\ResourceCollection;

class UserCollection extends ResourceCollection
{
    public $collects = UserResource::class;

    public function toArray(Request $request): array
    {
        return [
            'data' => $this->collection,
            'meta' => [
                'total_admins' => $this->collection->where('is_admin', true)->count(),
            ],
        ];
    }
}

// app/Http/Controllers/Api/UserController.php
namespace App\Http\Controllers\Api;

use App\Http\Resources\UserCollection;
use App\Http\Resources\UserResource;
use App\Models\User;

class UserController extends Controller
{
    public function index()
    {
        $users = User::with('role')
            ->withCount('posts')
            ->paginate(20);

        // ResourceCollection wraps paginated results automatically
        return new UserCollection($users);
    }

    public function show(User $user)
    {
        $user->loadMissing(['role', 'posts' => fn ($q) => $q->latest()->limit(5)]);

        return new UserResource($user);
    }

    public function store(StoreUserRequest $request)
    {
        $user = User::create($request->validated());

        // Consistent shape with proper status code
        return (new UserResource($user))
            ->response()
            ->setStatusCode(201);
    }
}
```

Reference: [Laravel API Resources](https://laravel.com/docs/eloquent-resources)
