---
title: Use Message Queues for Background Jobs
impact: MEDIUM-HIGH
impactDescription: Synchronous heavy processing blocks user requests
tags: microservices, queues, jobs, horizon, async-processing
---

## Use Message Queues for Background Jobs

Heavy or time-consuming operations such as sending emails, generating reports, processing images, or calling external APIs should never run synchronously inside a controller. This blocks the HTTP response, increases server resource usage, and degrades user experience. Use Laravel Queues with dedicated Job classes to offload work to background workers. Combine this with Laravel Horizon for monitoring, job chaining for sequential workflows, batching for parallel tasks, and job middleware for deduplication and throttling.

**Incorrect (synchronous heavy processing in the controller):**

```php
// app/Http/Controllers/ReportController.php
namespace App\Http\Controllers;

use App\Mail\ReportReady;
use App\Models\Report;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Mail;
use Barryvdh\DomPDF\Facade\Pdf;

class ReportController extends Controller
{
    public function generate(Request $request)
    {
        // Heavy processing directly in the controller - blocks the request
        $data = Report::query()
            ->with('entries', 'entries.transactions')
            ->whereBetween('created_at', [
                $request->input('start_date'),
                $request->input('end_date'),
            ])
            ->get();

        // CPU-intensive PDF generation - can take 30+ seconds
        $pdf = Pdf::loadView('reports.monthly', ['data' => $data]);
        $path = storage_path("app/reports/report-{$request->user()->id}.pdf");
        $pdf->save($path);

        // Sending email synchronously - waits for SMTP response
        Mail::to($request->user())->send(new ReportReady($path));

        // User has been waiting 30-60 seconds for this response
        return response()->json(['message' => 'Report generated and emailed.']);
    }
}
```

**Correct (queued jobs with chaining, batching, rate limiting, and Horizon monitoring):**

```php
// app/Jobs/GenerateReport.php
namespace App\Jobs;

use App\Models\Report;
use App\Models\User;
use Barryvdh\DomPDF\Facade\Pdf;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\Middleware\WithoutOverlapping;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Storage;

class GenerateReport implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;

    public int $timeout = 300; // 5 minutes max

    public int $backoff = 60;

    public function __construct(
        public readonly User $user,
        public readonly string $startDate,
        public readonly string $endDate,
        public readonly string $reportId,
    ) {}

    /**
     * Job middleware: prevent duplicate report generation for the same user.
     */
    public function middleware(): array
    {
        return [
            new WithoutOverlapping("report:{$this->user->id}"),
        ];
    }

    public function handle(): void
    {
        $data = Report::query()
            ->with('entries', 'entries.transactions')
            ->whereBetween('created_at', [$this->startDate, $this->endDate])
            ->get();

        $pdf = Pdf::loadView('reports.monthly', ['data' => $data]);
        $filename = "reports/{$this->reportId}.pdf";

        Storage::disk('s3')->put($filename, $pdf->output());

        // Update the report record with the file location
        Report::where('id', $this->reportId)->update([
            'status' => 'generated',
            'file_path' => $filename,
            'generated_at' => now(),
        ]);
    }

    public function failed(\Throwable $exception): void
    {
        Report::where('id', $this->reportId)->update([
            'status' => 'failed',
            'error' => $exception->getMessage(),
        ]);
    }
}

// app/Jobs/EmailReport.php
namespace App\Jobs;

use App\Mail\ReportReady;
use App\Models\Report;
use App\Models\User;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\Middleware\RateLimited;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Mail;

class EmailReport implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;

    public array $backoff = [30, 60, 120];

    public function __construct(
        public readonly User $user,
        public readonly string $reportId,
    ) {}

    /**
     * Throttle email sending to avoid hitting SMTP rate limits.
     */
    public function middleware(): array
    {
        return [
            new RateLimited('emails'),
        ];
    }

    public function handle(): void
    {
        $report = Report::findOrFail($this->reportId);

        Mail::to($this->user)->send(new ReportReady($report));

        $report->update(['emailed_at' => now()]);
    }
}

// app/Jobs/ProcessBulkReports.php - Batching for parallel work
namespace App\Jobs;

use App\Models\User;
use Illuminate\Bus\Batch;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Bus;

class ProcessBulkReports implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function handle(): void
    {
        $users = User::where('wants_monthly_report', true)->get();

        $jobs = $users->map(function (User $user) {
            $reportId = \Illuminate\Support\Str::uuid()->toString();

            return [
                new GenerateReport(
                    user: $user,
                    startDate: now()->subMonth()->startOfMonth()->toDateString(),
                    endDate: now()->subMonth()->endOfMonth()->toDateString(),
                    reportId: $reportId,
                ),
                new EmailReport(user: $user, reportId: $reportId),
            ];
        });

        // Bus::chain runs jobs sequentially per user
        // Bus::batch runs all user chains in parallel
        $chains = $jobs->map(fn (array $chain) => Bus::chain($chain));

        Bus::batch($chains->toArray())
            ->name('Monthly Reports - ' . now()->subMonth()->format('F Y'))
            ->onQueue('reports')
            ->allowFailures()
            ->then(function (Batch $batch) {
                // All jobs completed successfully
                \Illuminate\Support\Facades\Log::info('Bulk report batch completed.', [
                    'batch_id' => $batch->id,
                    'total' => $batch->totalJobs,
                ]);
            })
            ->catch(function (Batch $batch, \Throwable $e) {
                // First failure in the batch
                \Illuminate\Support\Facades\Log::warning('Bulk report batch had failures.', [
                    'batch_id' => $batch->id,
                    'failed' => $batch->failedJobs,
                ]);
            })
            ->dispatch();
    }
}

// app/Http/Controllers/ReportController.php
namespace App\Http\Controllers;

use App\Jobs\GenerateReport;
use App\Jobs\EmailReport;
use App\Models\Report;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Bus;
use Illuminate\Support\Str;

class ReportController extends Controller
{
    public function generate(Request $request)
    {
        $validated = $request->validate([
            'start_date' => ['required', 'date'],
            'end_date' => ['required', 'date', 'after:start_date'],
        ]);

        $reportId = Str::uuid()->toString();

        // Create a pending report record
        $report = Report::create([
            'id' => $reportId,
            'user_id' => $request->user()->id,
            'status' => 'pending',
            'start_date' => $validated['start_date'],
            'end_date' => $validated['end_date'],
        ]);

        // Chain jobs: generate first, then email
        Bus::chain([
            new GenerateReport(
                user: $request->user(),
                startDate: $validated['start_date'],
                endDate: $validated['end_date'],
                reportId: $reportId,
            ),
            new EmailReport(
                user: $request->user(),
                reportId: $reportId,
            ),
        ])->onQueue('reports')->dispatch();

        // Return immediately - the user can poll for status
        return response()->json([
            'message' => 'Report generation started.',
            'report_id' => $reportId,
            'status_url' => route('reports.status', $reportId),
        ], 202);
    }

    public function status(string $reportId)
    {
        $report = Report::where('id', $reportId)
            ->where('user_id', auth()->id())
            ->firstOrFail();

        return response()->json([
            'id' => $report->id,
            'status' => $report->status,
            'generated_at' => $report->generated_at,
            'emailed_at' => $report->emailed_at,
        ]);
    }
}

// app/Providers/AppServiceProvider.php - Rate limiter configuration
namespace App\Providers;

use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Rate limit for email sending jobs
        RateLimiter::for('emails', function (object $job) {
            return Limit::perMinute(30);
        });
    }
}

// config/horizon.php - Horizon configuration for monitoring and queue workers
return [
    'domain' => null,
    'path' => 'horizon',
    'use' => 'default',
    'prefix' => env('HORIZON_PREFIX', 'horizon:' . env('APP_NAME', 'laravel') . ':'),
    'middleware' => ['web', 'auth:admin'],

    'waits' => [
        'redis:default' => 60,
        'redis:reports' => 120,
    ],

    'trim' => [
        'recent' => 60,        // minutes
        'pending' => 60,
        'completed' => 60,
        'recent_failed' => 10080,  // 7 days
        'failed' => 10080,
        'monitored' => 10080,
    ],

    'environments' => [
        'production' => [
            'default-worker' => [
                'connection' => 'redis',
                'queue' => ['default'],
                'balance' => 'auto',
                'autoScalingStrategy' => 'time',
                'minProcesses' => 1,
                'maxProcesses' => 10,
                'balanceMaxShift' => 1,
                'balanceCooldown' => 3,
                'tries' => 3,
                'timeout' => 60,
            ],
            'reports-worker' => [
                'connection' => 'redis',
                'queue' => ['reports'],
                'balance' => 'auto',
                'autoScalingStrategy' => 'size',
                'minProcesses' => 1,
                'maxProcesses' => 5,
                'balanceMaxShift' => 1,
                'balanceCooldown' => 3,
                'tries' => 3,
                'timeout' => 600,  // Reports can take longer
            ],
            'notifications-worker' => [
                'connection' => 'redis',
                'queue' => ['notifications'],
                'balance' => 'simple',
                'processes' => 3,
                'tries' => 3,
                'timeout' => 30,
            ],
            'dead-letter-worker' => [
                'connection' => 'redis',
                'queue' => ['dead-letter'],
                'balance' => 'simple',
                'processes' => 1,
                'tries' => 1,
                'timeout' => 60,
            ],
        ],
    ],
];
```

Reference: [Laravel Queues Documentation](https://laravel.com/docs/queues)
