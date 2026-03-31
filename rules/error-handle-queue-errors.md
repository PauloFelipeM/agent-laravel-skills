---
title: Handle Queue and Job Errors Properly
impact: HIGH
impactDescription: Unhandled job failures cause silent data loss
tags: [error-handling, queues, jobs, failed-jobs]
---

## Handle Queue and Job Errors Properly

Queued jobs run outside the HTTP request lifecycle, so unhandled exceptions disappear silently unless you explicitly configure retry logic and failure handling. Without the `failed()` method, retry properties, and proper exception handling, a job that throws an exception will retry indefinitely with no backoff (wasting resources) or fail permanently with no notification. The `failed_jobs` table only captures the final failure -- if you never inspect it or set up alerts, data loss goes unnoticed for days.

Always define `$tries`, `$backoff`, and `$maxExceptions` on your jobs, implement the `failed()` method for cleanup and alerting, and catch specific exceptions to decide whether to retry or permanently fail.

**Incorrect**

```php
// app/Jobs/ProcessPayment.php
namespace App\Jobs;

use App\Models\Order;
use App\Services\PaymentGateway;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class ProcessPayment implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(
        public readonly Order $order,
    ) {}

    public function handle(PaymentGateway $gateway): void
    {
        // No try/catch -- any exception causes silent retry with no backoff.
        // No $tries property -- job retries indefinitely, hammering the payment API.
        // No failed() method -- when the job finally dies, nobody is notified.
        // No distinction between transient and permanent errors.
        $gateway->charge(
            $this->order->user->payment_method_id,
            $this->order->total_cents,
        );

        $this->order->update(['status' => 'paid']);
    }
}

// Dispatching with no awareness of failure handling
// app/Http/Controllers/CheckoutController.php
class CheckoutController extends Controller
{
    public function store(Request $request): JsonResponse
    {
        $order = Order::create($request->validated());

        // Fire and forget -- caller has no way to know if this ever succeeds
        ProcessPayment::dispatch($order);

        return response()->json(['message' => 'Order placed']);
    }
}
```

**Correct**

```php
// app/Jobs/ProcessPayment.php
namespace App\Jobs;

use App\Events\PaymentFailed;
use App\Exceptions\PaymentDeclinedException;
use App\Models\Order;
use App\Services\PaymentGateway;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\MaxAttemptsExceededException;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;
use Throwable;

class ProcessPayment implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    /**
     * Maximum number of attempts before the job is sent to failed_jobs.
     */
    public int $tries = 5;

    /**
     * Exponential backoff intervals (in seconds) between retries.
     */
    public array $backoff = [10, 30, 60, 120, 300];

    /**
     * Stop retrying after this many unhandled exceptions (prevents
     * burning through all $tries on a permanently broken payload).
     */
    public int $maxExceptions = 3;

    /**
     * Timeout in seconds for a single attempt.
     */
    public int $timeout = 30;

    public function __construct(
        public readonly Order $order,
    ) {}

    public function handle(PaymentGateway $gateway): void
    {
        try {
            $result = $gateway->charge(
                $this->order->user->payment_method_id,
                $this->order->total_cents,
            );

            $this->order->update([
                'status' => 'paid',
                'transaction_id' => $result->transactionId,
                'paid_at' => now(),
            ]);

            Log::info('Payment processed successfully', [
                'order_id' => $this->order->id,
                'transaction_id' => $result->transactionId,
            ]);
        } catch (PaymentDeclinedException $e) {
            // Permanent failure -- card declined, do NOT retry
            Log::warning('Payment declined permanently', [
                'order_id' => $this->order->id,
                'reason' => $e->declineReason(),
            ]);

            $this->order->update(['status' => 'payment_declined']);
            $this->fail($e); // Marks as failed immediately, skips remaining retries
        }
        // Transient exceptions (network timeouts, 5xx from gateway) are NOT caught
        // here, so they bubble up and trigger automatic retry with backoff.
    }

    /**
     * Called when the job has exhausted all retries or was manually failed.
     * Use this for cleanup, alerting, and compensating actions.
     */
    public function failed(?Throwable $exception): void
    {
        Log::error('Payment job failed permanently', [
            'order_id' => $this->order->id,
            'exception' => $exception?->getMessage(),
            'attempts' => $this->attempts(),
        ]);

        $this->order->update(['status' => 'payment_failed']);

        // Notify stakeholders so the failure does not go unnoticed
        PaymentFailed::dispatch($this->order, $exception);
    }

    /**
     * Determine the middleware the job should pass through.
     * Rate-limit to avoid hammering the payment gateway.
     */
    public function middleware(): array
    {
        return [
            new \Illuminate\Queue\Middleware\RateLimited('payments'),
        ];
    }
}

// config/queue.php -- ensure failed_jobs table is configured
// Run: php artisan queue:failed-table && php artisan migrate
//
// 'failed' => [
//     'driver' => env('QUEUE_FAILED_DRIVER', 'database-uuids'),
//     'database' => env('DB_CONNECTION', 'mysql'),
//     'table' => 'failed_jobs',
// ],

// app/Providers/AppServiceProvider.php -- global failure monitoring
namespace App\Providers;

use Illuminate\Queue\Events\JobFailed;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Queue;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Global listener that fires for ANY failed job across the application
        Queue::failing(function (JobFailed $event) {
            Log::channel('slack')->critical('Queue job failed', [
                'connection' => $event->connectionName,
                'queue' => $event->job->getQueue(),
                'job' => $event->job->resolveName(),
                'exception' => $event->exception->getMessage(),
            ]);
        });
    }
}
```

Reference: [Laravel Queues](https://laravel.com/docs/queues)
