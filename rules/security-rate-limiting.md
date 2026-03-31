---
title: Implement Rate Limiting
impact: HIGH
impactDescription: Prevents brute force and DoS attacks
tags: [security, rate-limiting, throttle, middleware]
---

## Implement Rate Limiting

Without rate limiting, your application is vulnerable to brute force password attacks, credential stuffing, API abuse, and denial-of-service attacks. Laravel provides a built-in rate limiter that integrates with the `throttle` middleware. Configure rate limits per endpoint type: stricter limits for authentication endpoints, moderate limits for write operations, and relaxed limits for read-heavy API endpoints.

Always define rate limiters in your `AppServiceProvider` (or `RouteServiceProvider` in older Laravel versions) and apply them via middleware on route groups or individual routes.

**Incorrect**

```php
// routes/api.php

// No rate limiting at all - endpoints are wide open to abuse
Route::post('/login', [AuthController::class, 'login']);
Route::post('/register', [AuthController::class, 'register']);
Route::post('/forgot-password', [PasswordResetController::class, 'send']);

// API routes with no throttling - a single client can make unlimited requests
Route::middleware('auth:sanctum')->group(function () {
    Route::get('/users', [UserController::class, 'index']);
    Route::post('/posts', [PostController::class, 'store']);
    Route::post('/comments', [CommentController::class, 'store']);
    Route::post('/upload', [UploadController::class, 'store']);
});

// Webhook endpoint with no protection - can be flooded
Route::post('/webhooks/stripe', [StripeWebhookController::class, 'handle']);
```

**Correct**

```php
// app/Providers/AppServiceProvider.php
namespace App\Providers;

use App\Models\User;
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        $this->configureRateLimiting();
    }

    protected function configureRateLimiting(): void
    {
        // Strict limit for authentication endpoints (brute force protection)
        RateLimiter::for('auth', function (Request $request) {
            return Limit::perMinute(5)
                ->by($request->ip() . '|' . $request->input('email', ''))
                ->response(function (Request $request, array $headers) {
                    return response()->json([
                        'message' => 'Too many login attempts. Please try again later.',
                        'retry_after' => $headers['Retry-After'] ?? 60,
                    ], 429, $headers);
                });
        });

        // Password reset: very strict to prevent email bombing
        RateLimiter::for('password-reset', function (Request $request) {
            return Limit::perMinute(3)
                ->by($request->ip())
                ->response(function () {
                    return response()->json([
                        'message' => 'Too many password reset attempts. Please wait before trying again.',
                    ], 429);
                });
        });

        // Standard API rate limit: per authenticated user
        RateLimiter::for('api', function (Request $request) {
            return $request->user()
                ? Limit::perMinute(60)->by($request->user()->id)
                : Limit::perMinute(20)->by($request->ip());
        });

        // Higher limit for read-heavy endpoints
        RateLimiter::for('api-reads', function (Request $request) {
            return $request->user()
                ? Limit::perMinute(120)->by($request->user()->id)
                : Limit::perMinute(30)->by($request->ip());
        });

        // Strict limit for write/mutation endpoints
        RateLimiter::for('api-writes', function (Request $request) {
            return $request->user()
                ? Limit::perMinute(30)->by($request->user()->id)
                : Limit::none(); // Unauthenticated users cannot write (handled by auth middleware)
        });

        // Uploads: very strict due to resource cost
        RateLimiter::for('uploads', function (Request $request) {
            return Limit::perMinute(10)->by($request->user()?->id ?: $request->ip());
        });

        // Tiered rate limits based on user plan
        RateLimiter::for('api-tiered', function (Request $request) {
            $user = $request->user();

            if (! $user) {
                return Limit::perMinute(10)->by($request->ip());
            }

            return match ($user->plan) {
                'enterprise' => Limit::perMinute(500)->by($user->id),
                'pro'        => Limit::perMinute(120)->by($user->id),
                'basic'      => Limit::perMinute(60)->by($user->id),
                default      => Limit::perMinute(30)->by($user->id),
            };
        });

        // Webhook endpoints: limit by source IP
        RateLimiter::for('webhooks', function (Request $request) {
            return Limit::perMinute(100)->by($request->ip());
        });

        // Multiple limits: per-second burst + per-minute sustained
        RateLimiter::for('api-strict', function (Request $request) {
            $key = $request->user()?->id ?: $request->ip();

            return [
                Limit::perSecond(3)->by($key),   // Prevent burst abuse
                Limit::perMinute(60)->by($key),   // Sustained limit
                Limit::perHour(500)->by($key),    // Hourly cap
            ];
        });
    }
}

// routes/api.php
use App\Http\Controllers\Auth\AuthController;
use App\Http\Controllers\Auth\PasswordResetController;
use App\Http\Controllers\CommentController;
use App\Http\Controllers\PostController;
use App\Http\Controllers\UploadController;
use App\Http\Controllers\UserController;
use App\Http\Controllers\Webhook\StripeWebhookController;

// Authentication routes with strict throttling
Route::middleware('throttle:auth')->group(function () {
    Route::post('/login', [AuthController::class, 'login']);
    Route::post('/register', [AuthController::class, 'register']);
});

Route::post('/forgot-password', [PasswordResetController::class, 'send'])
    ->middleware('throttle:password-reset');

// Authenticated API routes with tiered rate limiting
Route::middleware(['auth:sanctum'])->group(function () {
    // Read endpoints: higher limit
    Route::middleware('throttle:api-reads')->group(function () {
        Route::get('/users', [UserController::class, 'index']);
        Route::get('/posts', [PostController::class, 'index']);
        Route::get('/posts/{post}', [PostController::class, 'show']);
        Route::get('/posts/{post}/comments', [CommentController::class, 'index']);
    });

    // Write endpoints: lower limit
    Route::middleware('throttle:api-writes')->group(function () {
        Route::post('/posts', [PostController::class, 'store']);
        Route::put('/posts/{post}', [PostController::class, 'update']);
        Route::delete('/posts/{post}', [PostController::class, 'destroy']);
        Route::post('/comments', [CommentController::class, 'store']);
    });

    // Upload endpoints: strictest limit
    Route::post('/upload', [UploadController::class, 'store'])
        ->middleware('throttle:uploads');
});

// Webhook with IP-based throttling
Route::post('/webhooks/stripe', [StripeWebhookController::class, 'handle'])
    ->middleware('throttle:webhooks');

// Custom rate limit response with detailed headers

// app/Http/Controllers/Auth/AuthController.php
namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

class AuthController extends Controller
{
    public function login(Request $request): JsonResponse
    {
        // Manual rate limiting for additional control within a controller
        $throttleKey = 'login:' . $request->ip() . '|' . $request->input('email');

        if (RateLimiter::tooManyAttempts($throttleKey, 5)) {
            $seconds = RateLimiter::availableIn($throttleKey);

            return response()->json([
                'message' => "Too many login attempts. Please try again in {$seconds} seconds.",
                'retry_after' => $seconds,
            ], 429);
        }

        RateLimiter::increment($throttleKey, 60); // Decay after 60 seconds

        // ... perform authentication logic ...

        // On successful login, clear the rate limiter
        RateLimiter::clear($throttleKey);

        return response()->json(['token' => $token]);
    }
}
```

Reference: [Laravel Rate Limiting](https://laravel.com/docs/routing#rate-limiting)
