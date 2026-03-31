---
title: Use Exception Handler for Error Handling
impact: CRITICAL
impactDescription: Centralized error handling prevents information leaks
tags: [error-handling, exception-handler, logging, responses]
---

## Use Exception Handler for Error Handling

Scattering try/catch blocks across every controller method leads to duplicated error-formatting logic, inconsistent response structures, and the inevitable forgotten catch that leaks a stack trace to the client in production. Laravel's Exception Handler (`bootstrap/app.php` in Laravel 11+, or `app/Exceptions/Handler.php` in earlier versions) gives you a single place to map exception types to HTTP responses, configure logging channels, suppress sensitive details, and report to external services.

Move all cross-cutting error handling into the Handler's `register()` method using `reportable()` and `renderable()` callbacks. Controllers should stay clean -- let exceptions propagate naturally.

**Incorrect**

```php
// app/Http/Controllers/OrderController.php
namespace App\Http\Controllers;

use App\Models\Order;
use App\Services\OrderService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Illuminate\Validation\ValidationException;
use Throwable;

class OrderController extends Controller
{
    public function store(Request $request, OrderService $service): JsonResponse
    {
        // Duplicated try/catch in every single controller method
        try {
            $validated = $request->validate([
                'product_id' => 'required|exists:products,id',
                'quantity' => 'required|integer|min:1',
            ]);

            $order = $service->createOrder($request->user(), $validated);

            return response()->json($order, 201);
        } catch (ValidationException $e) {
            return response()->json(['errors' => $e->errors()], 422);
        } catch (Throwable $e) {
            // Logging is done ad-hoc with different formats in every controller
            Log::error('Order creation failed: ' . $e->getMessage());

            // In production this leaks the exception message to the client.
            // Different controllers return different error shapes.
            return response()->json([
                'error' => $e->getMessage(),
                'trace' => $e->getTraceAsString(), // SECURITY: stack trace exposed!
            ], 500);
        }
    }

    public function show(int $id, Request $request): JsonResponse
    {
        // Same pattern repeated -- 20 controllers x 5 methods = 100 try/catch blocks
        try {
            $order = Order::findOrFail($id);

            if ($order->user_id !== $request->user()->id) {
                return response()->json(['error' => 'Forbidden'], 403);
            }

            return response()->json($order);
        } catch (Throwable $e) {
            Log::error($e->getMessage());
            return response()->json(['error' => 'Something went wrong'], 500);
        }
    }
}
```

**Correct**

```php
// bootstrap/app.php (Laravel 11+)
// All exception handling is configured in one place.
use App\Exceptions\ExternalServiceException;
use App\Exceptions\InsufficientInventoryException;
use Illuminate\Auth\Access\AuthorizationException;
use Illuminate\Auth\AuthenticationException;
use Illuminate\Database\Eloquent\ModelNotFoundException;
use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;
use Illuminate\Http\Request;
use Illuminate\Validation\ValidationException;
use Symfony\Component\HttpKernel\Exception\HttpException;
use Throwable;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__ . '/../routes/web.php',
        api: __DIR__ . '/../routes/api.php',
    )
    ->withMiddleware(function (Middleware $middleware) {
        //
    })
    ->withExceptions(function (Exceptions $exceptions) {

        // -------------------------------------------------------
        // Reporting: control what gets logged and where
        // -------------------------------------------------------

        // Don't log exceptions that are normal user errors
        $exceptions->dontReport([
            AuthorizationException::class,
            HttpException::class,
            ModelNotFoundException::class,
            ValidationException::class,
        ]);

        // Report external service failures to Sentry/Bugsnag AND the default log
        $exceptions->reportable(function (ExternalServiceException $e) {
            app('sentry')->captureException($e);

            // Return false to ALSO log via the default channel.
            // Return true (or nothing) to stop propagation.
            return false;
        });

        // Report all unexpected errors to an external monitoring service
        $exceptions->reportable(function (Throwable $e) {
            if (app()->bound('sentry') && $e->getCode() >= 500) {
                app('sentry')->captureException($e);
            }
        });

        // -------------------------------------------------------
        // Rendering: map exception types to JSON responses
        // -------------------------------------------------------

        // Consistent 404 shape for missing models
        $exceptions->renderable(function (ModelNotFoundException $e, Request $request) {
            if ($request->expectsJson()) {
                $modelName = class_basename($e->getModel());

                return response()->json([
                    'message' => "{$modelName} not found.",
                    'type' => 'resource_not_found',
                ], 404);
            }
        });

        // Domain exception with business context
        $exceptions->renderable(function (InsufficientInventoryException $e, Request $request) {
            if ($request->expectsJson()) {
                return response()->json([
                    'message' => $e->getMessage(),
                    'type' => 'insufficient_inventory',
                    'product_id' => $e->productId,
                    'requested' => $e->requested,
                    'available' => $e->available,
                ], 422);
            }
        });

        // Catch-all for unexpected errors -- hide internals in production
        $exceptions->renderable(function (Throwable $e, Request $request) {
            if ($request->expectsJson() && ! $e instanceof HttpException) {
                $status = method_exists($e, 'getStatusCode')
                    ? $e->getStatusCode()
                    : 500;

                $body = [
                    'message' => $status >= 500
                        ? 'An internal error occurred.'
                        : $e->getMessage(),
                    'type' => 'server_error',
                ];

                // Only include debug info in non-production environments
                if (config('app.debug')) {
                    $body['debug'] = [
                        'exception' => get_class($e),
                        'message' => $e->getMessage(),
                        'file' => $e->getFile(),
                        'line' => $e->getLine(),
                    ];
                }

                return response()->json($body, $status);
            }
        });

    })->create();

// app/Exceptions/InsufficientInventoryException.php
namespace App\Exceptions;

use RuntimeException;

class InsufficientInventoryException extends RuntimeException
{
    public function __construct(
        public readonly int $productId,
        public readonly int $requested,
        public readonly int $available,
    ) {
        parent::__construct(
            "Product #{$productId}: requested {$requested}, only {$available} available."
        );
    }
}

// app/Exceptions/ExternalServiceException.php
namespace App\Exceptions;

use RuntimeException;

class ExternalServiceException extends RuntimeException
{
    public function __construct(
        public readonly string $service,
        string $message,
        int $code = 0,
        ?\Throwable $previous = null,
    ) {
        parent::__construct("[{$service}] {$message}", $code, $previous);
    }
}

// app/Http/Controllers/OrderController.php
// Controllers are clean -- no try/catch, no error formatting.
namespace App\Http\Controllers;

use App\Http\Requests\StoreOrderRequest;
use App\Http\Resources\OrderResource;
use App\Models\Order;
use App\Services\OrderService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class OrderController extends Controller
{
    public function store(StoreOrderRequest $request, OrderService $service): JsonResponse
    {
        // Validation is handled by StoreOrderRequest (throws ValidationException)
        // Inventory check throws InsufficientInventoryException
        // All mapped to correct responses by the Handler
        $order = $service->createOrder(
            $request->user(),
            $request->validated(),
        );

        return OrderResource::make($order)
            ->response()
            ->setStatusCode(201);
    }

    public function show(int $id, Request $request): JsonResponse
    {
        // findOrFail -> ModelNotFoundException -> 404 via Handler
        $order = Order::findOrFail($id);

        // authorize() -> AuthorizationException -> 403 via Handler
        $this->authorize('view', $order);

        return OrderResource::make($order)->response();
    }
}
```

Reference: [Laravel Error Handling](https://laravel.com/docs/errors)
