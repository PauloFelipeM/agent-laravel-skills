---
title: Use Guards for Authentication and Authorization
impact: HIGH
impactDescription: Missing authorization = unauthorized data access
tags: [security, guards, gates, policies, authorization]
---

## Use Guards for Authentication and Authorization

Never perform manual authentication or authorization checks scattered throughout your controllers. Laravel provides Guards for verifying identity (who is the user?) and Gates/Policies for verifying permissions (what can the user do?). Policies are the preferred approach for model-based authorization because they co-locate all permission logic for a model in one class, support auto-discovery, and integrate cleanly with controllers and Blade templates.

Missing or inconsistent authorization checks are the leading cause of Insecure Direct Object Reference (IDOR) vulnerabilities, where users can access or modify resources belonging to other users.

**Incorrect**

```php
// app/Http/Controllers/PostController.php
namespace App\Http\Controllers;

use App\Models\Post;
use Illuminate\Http\Request;

class PostController extends Controller
{
    public function update(Request $request, int $id): JsonResponse
    {
        $post = Post::findOrFail($id);

        // Manual auth check - easy to forget, inconsistent, hard to test
        if ($request->user()->id !== $post->user_id) {
            abort(403, 'Unauthorized');
        }

        // What about admins? Editors? This logic gets duplicated everywhere.
        $post->update($request->all());

        return response()->json($post);
    }

    public function destroy(int $id): JsonResponse
    {
        $post = Post::findOrFail($id);

        // Forgot the auth check entirely - any authenticated user can delete any post
        $post->delete();

        return response()->json(null, 204);
    }

    public function publish(int $id): JsonResponse
    {
        $post = Post::findOrFail($id);

        // Checking role with string comparison - fragile and scattered
        if (auth()->user()->role !== 'admin' && auth()->user()->role !== 'editor') {
            return response()->json(['error' => 'Not allowed'], 403);
        }

        $post->update(['published_at' => now()]);

        return response()->json($post);
    }
}
```

**Correct**

```php
// Step 1: Define Gates for simple, non-model actions

// app/Providers/AppServiceProvider.php
namespace App\Providers;

use App\Models\User;
use Illuminate\Support\Facades\Gate;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Gates for non-model actions
        Gate::define('access-admin-dashboard', function (User $user): bool {
            return $user->isAdmin();
        });

        Gate::define('view-analytics', function (User $user): bool {
            return $user->hasAnyRole(['admin', 'analyst']);
        });

        // Super-admin bypass: runs before all Gates and Policies
        Gate::before(function (User $user, string $ability): ?bool {
            if ($user->isSuperAdmin()) {
                return true; // Bypass all checks
            }

            return null; // Fall through to normal checks
        });
    }
}

// Step 2: Create a Policy for model authorization
// php artisan make:policy PostPolicy --model=Post

// app/Policies/PostPolicy.php
namespace App\Policies;

use App\Models\Post;
use App\Models\User;

class PostPolicy
{
    /**
     * Determine whether the user can view any posts.
     */
    public function viewAny(User $user): bool
    {
        return true; // All authenticated users can list posts
    }

    /**
     * Determine whether the user can view the post.
     */
    public function view(User $user, Post $post): bool
    {
        // Published posts are visible to everyone; drafts only to the author
        if ($post->isPublished()) {
            return true;
        }

        return $user->id === $post->user_id;
    }

    /**
     * Determine whether the user can create posts.
     */
    public function create(User $user): bool
    {
        return $user->hasAnyRole(['admin', 'editor', 'author']);
    }

    /**
     * Determine whether the user can update the post.
     */
    public function update(User $user, Post $post): bool
    {
        // Authors can edit their own posts; editors can edit any post
        return $user->id === $post->user_id
            || $user->hasRole('editor');
    }

    /**
     * Determine whether the user can delete the post.
     */
    public function delete(User $user, Post $post): bool
    {
        return $user->id === $post->user_id
            || $user->hasRole('admin');
    }

    /**
     * Determine whether the user can publish the post.
     */
    public function publish(User $user, Post $post): bool
    {
        return $user->hasAnyRole(['admin', 'editor']);
    }

    /**
     * Determine whether the user can restore a soft-deleted post.
     */
    public function restore(User $user, Post $post): bool
    {
        return $user->hasRole('admin');
    }

    /**
     * Determine whether the user can permanently delete the post.
     */
    public function forceDelete(User $user, Post $post): bool
    {
        return $user->hasRole('admin');
    }
}

// Step 3: Policy auto-discovery works when naming follows convention:
//   App\Models\Post  ->  App\Policies\PostPolicy
// No manual registration needed. For non-standard names, register in AppServiceProvider:
//   Gate::policy(Post::class, PostPolicy::class);

// Step 4: Use authorization in controllers

// app/Http/Controllers/PostController.php
namespace App\Http\Controllers;

use App\Http\Requests\StorePostRequest;
use App\Http\Requests\UpdatePostRequest;
use App\Models\Post;
use Illuminate\Http\JsonResponse;

class PostController extends Controller
{
    public function index(): JsonResponse
    {
        $this->authorize('viewAny', Post::class);

        $posts = Post::published()->paginate();

        return response()->json($posts);
    }

    public function store(StorePostRequest $request): JsonResponse
    {
        $this->authorize('create', Post::class);

        $post = $request->user()->posts()->create($request->validated());

        return response()->json($post, 201);
    }

    public function show(Post $post): JsonResponse
    {
        $this->authorize('view', $post);

        return response()->json($post->load('author', 'comments'));
    }

    public function update(UpdatePostRequest $request, Post $post): JsonResponse
    {
        $this->authorize('update', $post);

        $post->update($request->validated());

        return response()->json($post);
    }

    public function destroy(Post $post): JsonResponse
    {
        $this->authorize('delete', $post);

        $post->delete();

        return response()->json(null, 204);
    }

    public function publish(Post $post): JsonResponse
    {
        $this->authorize('publish', $post);

        $post->update(['published_at' => now()]);

        return response()->json($post);
    }
}

// Step 5: Apply authorization via route middleware as an alternative

// routes/api.php
use App\Http\Controllers\PostController;

Route::middleware('auth:sanctum')->group(function () {
    Route::apiResource('posts', PostController::class);

    // Middleware-based authorization for specific routes
    Route::patch('/posts/{post}/publish', [PostController::class, 'publish'])
        ->middleware('can:publish,post');

    // Gate-based middleware for non-model actions
    Route::get('/admin/dashboard', [AdminController::class, 'index'])
        ->middleware('can:access-admin-dashboard');
});

// Step 6: Use policies in Blade templates

// resources/views/posts/show.blade.php
// @can('update', $post)
//     <a href="{{ route('posts.edit', $post) }}">Edit Post</a>
// @endcan
//
// @can('delete', $post)
//     <form method="POST" action="{{ route('posts.destroy', $post) }}">
//         @csrf @method('DELETE')
//         <button type="submit">Delete</button>
//     </form>
// @endcan
//
// @cannot('publish', $post)
//     <p>You do not have permission to publish this post.</p>
// @endcannot
```

Reference: [Laravel Authorization](https://laravel.com/docs/authorization)
