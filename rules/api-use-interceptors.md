---
title: Use Middleware for Cross-Cutting Concerns
impact: MEDIUM
impactDescription: Duplicated logic across controllers increases bugs
tags: api, middleware, cross-cutting, request-response
---

## Use Middleware for Cross-Cutting Concerns

Cross-cutting concerns such as logging, response formatting, locale detection, and content-type enforcement should not be scattered across individual controllers. Duplicating this logic violates DRY, makes it easy to miss an endpoint, and turns every new controller into a copy-paste exercise. Laravel Middleware provides a clean pipeline for intercepting requests and responses in a centralized, testable, and composable way.

**Incorrect**

```php
// Every controller duplicates the same boilerplate logic
class OrderController extends Controller
{
    public function index(Request $request)
    {
        // Duplicated logging in every method
        Log::info('API request', [
            'method' => $request->method(),
            'path' => $request->path(),
            'user' => $request->user()?->id,
            'ip' => $request->ip(),
        ]);

        // Duplicated locale detection
        $locale = $request->header('Accept-Language', 'en');
        app()->setLocale(substr($locale, 0, 2));

        // Duplicated JSON enforcement
        if (!$request->expectsJson()) {
            return response()->json(['error' => 'JSON required'], 406);
        }

        $orders = Order::paginate(20);

        $duration = microtime(true) - LARAVEL_START;
        Log::info('API response', ['duration_ms' => $duration * 1000]);

        // Duplicated response wrapping
        return response()->json([
            'success' => true,
            'data' => $orders->items(),
            'meta' => ['page' => $orders->currentPage()],
        ]);
    }

    // Same boilerplate repeated in store(), show(), update(), destroy()...
}
```

**Correct**

```php
// app/Http/Middleware/ForceJsonResponse.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class ForceJsonResponse
{
    /**
     * Ensure every API request accepts and returns JSON.
     */
    public function handle(Request $request, Closure $next): Response
    {
        // Force the Accept header so exception handler renders JSON
        $request->headers->set('Accept', 'application/json');

        $response = $next($request);

        // Guarantee content-type on outbound responses
        $response->headers->set('Content-Type', 'application/json');

        return $response;
    }
}

// app/Http/Middleware/ApiRequestLogger.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Symfony\Component\HttpFoundation\Response;

class ApiRequestLogger
{
    /**
     * Log incoming request and outgoing response as a terminable middleware.
     * The terminate() method runs after the response has been sent to the client,
     * so it does not add latency.
     */
    public function handle(Request $request, Closure $next): Response
    {
        $request->attributes->set('api_start_time', microtime(true));

        return $next($request);
    }

    public function terminate(Request $request, Response $response): void
    {
        $duration = microtime(true) - $request->attributes->get('api_start_time', microtime(true));

        Log::channel('api')->info('API call', [
            'method' => $request->method(),
            'path' => $request->path(),
            'status' => $response->getStatusCode(),
            'duration_ms' => round($duration * 1000, 2),
            'user_id' => $request->user()?->id,
            'ip' => $request->ip(),
        ]);
    }
}

// app/Http/Middleware/SetLocaleFromHeader.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class SetLocaleFromHeader
{
    private const SUPPORTED_LOCALES = ['en', 'es', 'fr', 'pt', 'de'];

    public function handle(Request $request, Closure $next): Response
    {
        $preferred = substr($request->header('Accept-Language', 'en'), 0, 2);

        $locale = in_array($preferred, self::SUPPORTED_LOCALES, true)
            ? $preferred
            : 'en';

        app()->setLocale($locale);

        return $next($request);
    }
}

// app/Http/Middleware/WrapApiResponse.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class WrapApiResponse
{
    /**
     * Wrap successful JSON responses in a consistent envelope.
     */
    public function handle(Request $request, Closure $next): Response
    {
        $response = $next($request);

        if (!$response instanceof JsonResponse || $response->getStatusCode() >= 400) {
            return $response;
        }

        $original = $response->getData(true);

        // Don't double-wrap if already wrapped (e.g. from a ResourceCollection)
        if (isset($original['data'])) {
            return $response;
        }

        $response->setData([
            'success' => true,
            'data' => $original,
        ]);

        return $response;
    }
}

// bootstrap/app.php (Laravel 11+)
use App\Http\Middleware\ApiRequestLogger;
use App\Http\Middleware\ForceJsonResponse;
use App\Http\Middleware\SetLocaleFromHeader;
use App\Http\Middleware\WrapApiResponse;

return Application::configure(basePath: dirname(__DIR__))
    ->withMiddleware(function (Middleware $middleware) {
        $middleware->api(prepend: [
            ForceJsonResponse::class,
            SetLocaleFromHeader::class,
        ]);

        $middleware->api(append: [
            WrapApiResponse::class,
        ]);

        // Terminable middleware - runs after response is sent
        $middleware->append(ApiRequestLogger::class);

        // Set priority so ForceJsonResponse always runs first
        $middleware->priority([
            ForceJsonResponse::class,
            \Illuminate\Auth\Middleware\Authenticate::class,
            SetLocaleFromHeader::class,
        ]);
    })
    ->create();

// Controller is now clean and focused on business logic
class OrderController extends Controller
{
    public function index()
    {
        return OrderResource::collection(
            Order::with('customer')->paginate(20)
        );
    }

    public function store(StoreOrderRequest $request)
    {
        $order = Order::create($request->validated());

        return (new OrderResource($order))
            ->response()
            ->setStatusCode(201);
    }
}
```

Reference: [Laravel Middleware](https://laravel.com/docs/middleware)
