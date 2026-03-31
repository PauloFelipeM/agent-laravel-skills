---
title: Use Structured Logging
impact: MEDIUM
impactDescription: Unstructured logs are impossible to search in production
tags: devops, logging, monolog, channels, structured-logging
---

## Use Structured Logging

Production logs must be structured and searchable. Using `error_log()`, `echo`, `dd()`, or string concatenation in log messages makes it nearly impossible to filter, aggregate, or alert on log data in tools like ELK, Datadog, or CloudWatch. Laravel's Log facade supports context arrays that produce structured key-value pairs, multiple channels for routing logs to different destinations, and contextual logging that attaches metadata to every log entry within a request lifecycle.

**Incorrect**

```php
// app/Services/OrderService.php
namespace App\Services;

class OrderService
{
    public function process(Order $order): void
    {
        // WRONG: error_log bypasses Laravel's logging entirely
        error_log("Processing order " . $order->id);

        // WRONG: echo/print pollute stdout and break JSON responses
        echo "Order total: " . $order->total;

        // WRONG: dd/dump left in code -- kills the request
        dump($order->toArray());

        try {
            $this->chargeCustomer($order);
        } catch (\Exception $e) {
            // WRONG: String concatenation -- impossible to parse or filter
            Log::error("Order failed for user " . $order->user_id . " with error: " . $e->getMessage() . " on order " . $order->id);

            // WRONG: Logging the full exception as a string loses stack trace structure
            Log::error("Exception: " . (string) $e);
        }

        // WRONG: No context, no way to correlate this log with a specific order
        Log::info("Order processed successfully");
    }
}

// config/logging.php -- Using only the default single channel
return [
    'default' => 'single',
    'channels' => [
        'single' => [
            'driver' => 'single',
            'path' => storage_path('logs/laravel.log'),
            // No rotation, no structured format, single monolithic file
        ],
    ],
];
```

**Correct**

```php
// config/logging.php -- Multiple channels with structured output
return [
    'default' => env('LOG_CHANNEL', 'stack'),

    'channels' => [
        // Stack sends to multiple channels simultaneously
        'stack' => [
            'driver' => 'stack',
            'channels' => ['daily', 'stderr'],
            'ignore_exceptions' => false,
        ],

        // Daily rotation prevents disk space issues
        'daily' => [
            'driver' => 'daily',
            'path' => storage_path('logs/laravel.log'),
            'level' => env('LOG_LEVEL', 'debug'),
            'days' => 14,
            'replace_placeholders' => true,
        ],

        // Stderr for container/cloud environments (Docker, Kubernetes)
        'stderr' => [
            'driver' => 'monolog',
            'level' => env('LOG_LEVEL', 'debug'),
            'handler' => StreamHandler::class,
            'with' => [
                'stream' => 'php://stderr',
            ],
            'formatter' => JsonFormatter::class,  // Structured JSON for log aggregators
        ],

        // Dedicated channel for payment events -- easy to monitor separately
        'payments' => [
            'driver' => 'daily',
            'path' => storage_path('logs/payments.log'),
            'level' => 'info',
            'days' => 30,
        ],

        // Slack alerts for critical errors only
        'slack' => [
            'driver' => 'slack',
            'url' => env('LOG_SLACK_WEBHOOK_URL'),
            'username' => 'Laravel Log',
            'emoji' => ':boom:',
            'level' => 'critical',
        ],

        // JSON channel for ELK/Datadog/CloudWatch ingestion
        'json' => [
            'driver' => 'monolog',
            'handler' => StreamHandler::class,
            'with' => [
                'stream' => storage_path('logs/app.json.log'),
            ],
            'formatter' => JsonFormatter::class,
            'formatter_with' => [
                'includeStacktraces' => true,
            ],
        ],
    ],
];

// app/Http/Middleware/AttachRequestContext.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Str;

class AttachRequestContext
{
    public function handle(Request $request, Closure $next)
    {
        // Attach context to every log entry for the duration of this request
        Log::withContext([
            'request_id' => $request->header('X-Request-ID', Str::uuid()->toString()),
            'user_id' => $request->user()?->id,
            'ip' => $request->ip(),
            'url' => $request->fullUrl(),
            'method' => $request->method(),
        ]);

        return $next($request);
    }
}

// app/Services/OrderService.php
namespace App\Services;

use Illuminate\Support\Facades\Log;

class OrderService
{
    public function process(Order $order): void
    {
        // CORRECT: Structured context as an array -- every field is searchable
        Log::info('Order processing started.', [
            'order_id' => $order->id,
            'user_id' => $order->user_id,
            'total' => $order->total,
            'items_count' => $order->items->count(),
        ]);

        try {
            $this->chargeCustomer($order);

            Log::info('Order payment charged successfully.', [
                'order_id' => $order->id,
                'payment_method' => $order->payment_method,
                'amount' => $order->total,
            ]);
        } catch (\Throwable $e) {
            // CORRECT: Pass exception as context -- preserves full stack trace
            Log::error('Order payment failed.', [
                'order_id' => $order->id,
                'user_id' => $order->user_id,
                'amount' => $order->total,
                'error' => $e->getMessage(),
                'exception' => $e,
            ]);

            // CORRECT: Route critical failures to a specific channel
            Log::channel('slack')->critical('Payment processing failure requires attention.', [
                'order_id' => $order->id,
                'error' => $e->getMessage(),
            ]);

            throw $e;
        }
    }
}

// app/Services/PaymentGateway.php -- Using a dedicated log channel
namespace App\Services;

use Illuminate\Support\Facades\Log;

class PaymentGateway
{
    private $log;

    public function __construct()
    {
        // Dedicated channel keeps payment logs separate and easy to audit
        $this->log = Log::channel('payments');
    }

    public function charge(string $customerId, int $amount): PaymentResult
    {
        $this->log->info('Initiating charge.', [
            'customer_id' => $customerId,
            'amount' => $amount,
            'currency' => config('services.stripe.currency'),
        ]);

        $result = $this->gateway->charge($customerId, $amount);

        $this->log->info('Charge completed.', [
            'customer_id' => $customerId,
            'transaction_id' => $result->transactionId,
            'status' => $result->status,
        ]);

        return $result;
    }
}

// app/Logging/CustomJsonFormatter.php -- Custom formatter for log aggregators
namespace App\Logging;

use Monolog\Formatter\JsonFormatter;
use Monolog\LogRecord;

class CustomJsonFormatter extends JsonFormatter
{
    public function format(LogRecord $record): string
    {
        $data = [
            'timestamp' => $record->datetime->format('c'),
            'level' => $record->level->getName(),
            'message' => $record->message,
            'context' => $record->context,
            'service' => config('app.name'),
            'environment' => config('app.env'),
            'hostname' => gethostname(),
        ];

        return json_encode($data, JSON_UNESCAPED_SLASHES) . "\n";
    }
}
```

Reference: [Laravel Logging](https://laravel.com/docs/logging)
