---
title: Implement Secure JWT Authentication
impact: CRITICAL
impactDescription: Essential for secure APIs
tags: [security, jwt, authentication, sanctum, passport]
---

## Implement Secure JWT Authentication

Never roll your own JWT authentication. Manual token generation with hardcoded secrets, missing expiration, and no revocation mechanism creates severe security vulnerabilities. Laravel provides two battle-tested packages: **Sanctum** for SPA and mobile token authentication, and **Passport** for full OAuth2 server implementations. Both handle token signing, expiration, revocation, and scope management out of the box.

Sanctum is the recommended choice for most applications (SPAs, mobile apps, simple API tokens). Use Passport only when you need full OAuth2 compliance (authorization codes, client credentials, third-party API access).

**Incorrect**

```php
// app/Http/Controllers/AuthController.php
namespace App\Http\Controllers;

use App\Models\User;
use Firebase\JWT\JWT;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;

class AuthController extends Controller
{
    // Hardcoded secret in source code - will leak via version control
    private const JWT_SECRET = 'my-super-secret-key-123';

    public function login(Request $request): JsonResponse
    {
        $user = User::where('email', $request->email)->first();

        if (! $user || ! Hash::check($request->password, $user->password)) {
            return response()->json(['error' => 'Invalid credentials'], 401);
        }

        // No expiration set - token is valid forever
        // No token ID - cannot be revoked individually
        // No audience/issuer claims - vulnerable to token confusion attacks
        $token = JWT::encode(
            ['sub' => $user->id, 'email' => $user->email],
            self::JWT_SECRET,
            'HS256'
        );

        // Token returned but never stored - no way to revoke it
        return response()->json(['token' => $token]);
    }

    public function logout(Request $request): JsonResponse
    {
        // No actual token invalidation - the token remains valid
        return response()->json(['message' => 'Logged out']);
    }
}

// routes/api.php
// No middleware protection, no rate limiting on auth endpoints
Route::post('/login', [AuthController::class, 'login']);
Route::get('/user', function (Request $request) {
    // Manual token parsing in every route - error-prone and inconsistent
    $token = $request->bearerToken();
    $payload = JWT::decode($token, new Key(AuthController::JWT_SECRET, 'HS256'));
    return User::find($payload->sub);
});
```

**Correct**

```php
// Using Laravel Sanctum for SPA/Mobile Token Authentication

// Install: composer require laravel/sanctum
// Publish: php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"

// config/sanctum.php - configure token expiration
return [
    'expiration' => 60 * 24, // Tokens expire after 24 hours (in minutes)
    'token_prefix' => '',
    'middleware' => [
        'authenticate_session' => Laravel\Sanctum\Http\Middleware\AuthenticateSession::class,
        'encrypt_cookies' => Illuminate\Cookie\Middleware\EncryptCookies::class,
        'validate_csrf_token' => Illuminate\Foundation\Http\Middleware\ValidateCsrfToken::class,
    ],
];

// app/Http/Controllers/Auth/TokenAuthController.php
namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use App\Http\Requests\Auth\LoginRequest;
use App\Models\User;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\ValidationException;

class TokenAuthController extends Controller
{
    public function login(LoginRequest $request): JsonResponse
    {
        $user = User::where('email', $request->validated('email'))->first();

        if (! $user || ! Hash::check($request->validated('password'), $user->password)) {
            throw ValidationException::withMessages([
                'email' => ['The provided credentials are incorrect.'],
            ]);
        }

        // Define token abilities (scopes) based on user role
        $abilities = $this->getAbilitiesForUser($user);

        // Create token with specific abilities and optional expiration override
        $token = $user->createToken(
            name: $request->validated('device_name', 'api-token'),
            abilities: $abilities,
            expiresAt: now()->addHours(24),
        );

        return response()->json([
            'token' => $token->plainTextToken,
            'expires_at' => $token->accessToken->expires_at,
            'abilities' => $abilities,
        ]);
    }

    public function logout(Request $request): JsonResponse
    {
        // Revoke the specific token used for this request
        $request->user()->currentAccessToken()->delete();

        return response()->json(['message' => 'Token revoked successfully.']);
    }

    public function logoutAll(Request $request): JsonResponse
    {
        // Revoke all tokens for this user (e.g., password change, security breach)
        $request->user()->tokens()->delete();

        return response()->json(['message' => 'All tokens revoked successfully.']);
    }

    private function getAbilitiesForUser(User $user): array
    {
        return match ($user->role) {
            'admin' => ['*'],
            'editor' => ['posts:read', 'posts:write', 'comments:read', 'comments:write'],
            'viewer' => ['posts:read', 'comments:read'],
            default => ['posts:read'],
        };
    }
}

// routes/api.php
use App\Http\Controllers\Auth\TokenAuthController;
use App\Http\Controllers\PostController;

Route::post('/login', [TokenAuthController::class, 'login'])
    ->middleware('throttle:5,1'); // 5 attempts per minute

Route::middleware('auth:sanctum')->group(function () {
    Route::post('/logout', [TokenAuthController::class, 'logout']);
    Route::post('/logout-all', [TokenAuthController::class, 'logoutAll']);

    // Protected routes with token ability checks
    Route::get('/posts', [PostController::class, 'index'])
        ->middleware('ability:posts:read');
    Route::post('/posts', [PostController::class, 'store'])
        ->middleware('ability:posts:write');
});

// app/Http/Controllers/PostController.php
namespace App\Http\Controllers;

use Illuminate\Http\Request;

class PostController extends Controller
{
    public function index(Request $request): JsonResponse
    {
        // Token abilities are already verified by middleware
        // Access the authenticated user safely
        $user = $request->user();

        // Check for a specific ability programmatically when needed
        if ($user->tokenCan('posts:write')) {
            // Include draft posts for users who can write
            return response()->json($user->posts()->withDrafts()->paginate());
        }

        return response()->json($user->posts()->published()->paginate());
    }
}

// Using Laravel Passport for Full OAuth2 (when OAuth2 compliance is required)

// Install: composer require laravel/passport
// Run: php artisan passport:install

// app/Models/User.php
namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens;
}

// app/Providers/AppServiceProvider.php
namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Laravel\Passport\Passport;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Define OAuth2 scopes
        Passport::tokensCan([
            'read-posts' => 'Read posts',
            'write-posts' => 'Create and update posts',
            'admin' => 'Full administrative access',
        ]);

        Passport::tokensExpireIn(now()->addDays(15));
        Passport::refreshTokensExpireIn(now()->addDays(30));
        Passport::personalAccessTokensExpireIn(now()->addMonths(6));
    }
}

// routes/api.php (Passport)
Route::middleware('auth:api')->group(function () {
    Route::get('/user', function (Request $request) {
        return $request->user();
    })->middleware('scope:read-posts');
});
```

Reference: [Laravel Sanctum](https://laravel.com/docs/sanctum) | [Laravel Passport](https://laravel.com/docs/passport)
