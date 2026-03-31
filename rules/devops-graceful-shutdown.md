---
title: Implement Graceful Shutdown
impact: MEDIUM
impactDescription: Ungraceful shutdown causes data loss in queue workers
tags: devops, graceful-shutdown, signals, queue-workers, deployment
---

## Implement Graceful Shutdown

Queue workers and long-running processes must be shut down gracefully to avoid losing jobs mid-execution. When a worker is killed abruptly with SIGKILL, any job currently being processed is lost -- it will not be retried because the worker never had a chance to release it back to the queue. Laravel provides built-in mechanisms for graceful restarts and zero-downtime deployments that prevent data loss during deployment cycles.

**Incorrect**

```php
// Deployment script that kills workers abruptly -- jobs in progress are lost
// deploy.sh
#!/bin/bash
git pull origin main
composer install --no-dev
php artisan migrate --force

# WRONG: SIGKILL gives workers zero time to finish current jobs
kill -9 $(pgrep -f "queue:work")

# WRONG: No maintenance mode, users see errors mid-deploy
php artisan config:cache
php artisan route:cache

# Workers restarted with no signal handling
php artisan queue:work &

// app/Console/Commands/ProcessReports.php
namespace App\Console\Commands;

use Illuminate\Console\Command;

class ProcessReports extends Command
{
    protected $signature = 'reports:process';

    public function handle()
    {
        // WRONG: No signal handling -- SIGTERM kills this mid-iteration
        // and partially processed data is left in an inconsistent state
        $reports = Report::where('status', 'pending')->get();

        foreach ($reports as $report) {
            // If killed here, report is partially processed with no way to recover
            $report->status = 'processing';
            $report->save();

            $this->generatePdf($report);
            $this->sendToClient($report);

            $report->status = 'completed';
            $report->save();
        }
    }
}

// Supervisor config with no graceful stop
// /etc/supervisor/conf.d/worker.conf
// [program:laravel-worker]
// command=php /var/www/artisan queue:work
// stopwaitsecs=1          ; Only 1 second before SIGKILL -- not enough
// stopsignal=KILL         ; WRONG: Should never use KILL as stop signal
```

**Correct**

```php
// deploy.sh -- Zero-downtime deployment with graceful shutdown
#!/bin/bash
set -e

echo "Starting zero-downtime deployment..."

# 1. Enter maintenance mode with retry header so load balancers can retry
php artisan down --retry=60 --refresh=15

# 2. Pull code and install dependencies
git pull origin main
composer install --no-dev --optimize-autoloader

# 3. Run migrations
php artisan migrate --force

# 4. Cache configuration, routes, and views
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache

# 5. Gracefully restart queue workers -- they finish current job then exit
php artisan queue:restart

# 6. If using Horizon, gracefully terminate (finishes current jobs)
php artisan horizon:terminate

# 7. Exit maintenance mode
php artisan up

echo "Deployment complete."

// app/Console/Commands/ProcessReports.php
namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\Log;

class ProcessReports extends Command
{
    protected $signature = 'reports:process';

    private bool $shouldStop = false;

    public function handle(): int
    {
        // Register signal handlers for graceful shutdown
        pcntl_async_signals(true);

        pcntl_signal(SIGTERM, function () {
            Log::info('ProcessReports received SIGTERM, finishing current report...');
            $this->shouldStop = true;
        });

        pcntl_signal(SIGINT, function () {
            Log::info('ProcessReports received SIGINT, finishing current report...');
            $this->shouldStop = true;
        });

        $reports = Report::where('status', 'pending')->get();

        foreach ($reports as $report) {
            // Check if we should stop before starting a new unit of work
            if ($this->shouldStop) {
                Log::info('ProcessReports stopping gracefully, remaining reports will be processed next run.');
                return self::SUCCESS;
            }

            // Wrap each report in a transaction for atomicity
            DB::transaction(function () use ($report) {
                $report->status = 'processing';
                $report->save();

                $this->generatePdf($report);
                $this->sendToClient($report);

                $report->status = 'completed';
                $report->save();
            });

            Log::info('Report processed successfully.', ['report_id' => $report->id]);
        }

        return self::SUCCESS;
    }
}

// app/Jobs/GenerateInvoice.php -- Queue job with proper timeout and retry
namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;

class GenerateInvoice implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    // Maximum number of attempts before marking as failed
    public int $tries = 3;

    // Timeout in seconds -- must be shorter than supervisor's stopwaitsecs
    public int $timeout = 120;

    // Retry after these many seconds on each attempt
    public function backoff(): array
    {
        return [30, 60, 120];
    }

    public function __construct(
        private readonly Invoice $invoice,
    ) {}

    public function handle(InvoiceService $service): void
    {
        Log::info('Generating invoice.', ['invoice_id' => $this->invoice->id]);

        $service->generate($this->invoice);
        $service->sendToCustomer($this->invoice);
    }

    // Called when all retry attempts are exhausted
    public function failed(\Throwable $exception): void
    {
        Log::error('Invoice generation failed permanently.', [
            'invoice_id' => $this->invoice->id,
            'error' => $exception->getMessage(),
        ]);

        $this->invoice->update(['status' => 'failed']);
    }
}

// Supervisor config with graceful shutdown settings
// /etc/supervisor/conf.d/worker.conf
//
// [program:laravel-worker]
// process_name=%(program_name)s_%(process_num)02d
// command=php /var/www/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600 --timeout=120
// autostart=true
// autorestart=true
// stopasgroup=true
// killasgroup=true
// numprocs=4
// stopsignal=TERM       ; Send SIGTERM first for graceful shutdown
// stopwaitsecs=130      ; Wait longer than job timeout before SIGKILL

// config/horizon.php -- Laravel Horizon graceful shutdown configuration
return [
    'environments' => [
        'production' => [
            'supervisor-1' => [
                'maxProcesses' => 10,
                'balanceMaxShift' => 1,
                'balanceCooldown' => 3,
                'tries' => 3,
                'timeout' => 120,
                // Horizon handles graceful shutdown automatically
                // Workers finish current job before terminating
            ],
        ],
    ],
];
```

Reference: [Laravel Queues](https://laravel.com/docs/queues), [Laravel Deployment](https://laravel.com/docs/deployment)
