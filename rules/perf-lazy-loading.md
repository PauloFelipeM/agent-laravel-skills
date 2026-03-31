---
title: Use Lazy Loading and Route Caching
impact: MEDIUM-HIGH
impactDescription: "Reduces memory usage and startup time"
tags: [performance, lazy-loading, lazy-collections, route-cache]
---

## Use Lazy Loading and Route Caching

Standard Eloquent collections load all results into memory at once, which becomes a problem with large datasets. Laravel's `LazyCollection` streams results one at a time using PHP generators, keeping memory usage constant regardless of dataset size. Combined with production caching commands (`route:cache`, `config:cache`, `event:cache`), this significantly reduces both memory footprint and request latency.

In development, use `Model::preventLazyLoading()` to catch accidental N+1 queries early -- it throws an exception whenever a relationship is lazy-loaded instead of eager-loaded.

**Incorrect**

```php
// app/Http/Controllers/ExportController.php
namespace App\Http\Controllers;

use App\Models\Transaction;
use App\Models\User;

class ExportController extends Controller
{
    public function exportTransactions()
    {
        // BAD: Loading 100k+ records into memory at once
        // This will consume hundreds of MB of RAM and may trigger OOM errors
        $transactions = Transaction::all();

        $csv = fopen('php://output', 'w');
        header('Content-Type: text/csv');

        foreach ($transactions as $transaction) {
            // BAD: Lazy loading user on each iteration (N+1 problem)
            fputcsv($csv, [
                $transaction->id,
                $transaction->user->name,  // Fires a query per row
                $transaction->amount,
                $transaction->created_at,
            ]);
        }

        fclose($csv);
    }

    public function generateReport()
    {
        // BAD: get() loads ALL matching users into a Collection
        $users = User::where('is_active', true)->get();

        // BAD: Chaining collection methods on 50k+ records in memory
        $highValueUsers = $users
            ->filter(fn ($user) => $user->orders->sum('total') > 1000)
            ->sortByDesc(fn ($user) => $user->orders->sum('total'))
            ->values();

        return view('reports.high-value', compact('highValueUsers'));
    }
}

// app/Console/Commands/ProcessUsers.php
namespace App\Console\Commands;

use App\Models\User;
use Illuminate\Console\Command;

class ProcessUsers extends Command
{
    protected $signature = 'users:process';

    public function handle(): void
    {
        // BAD: Loading all users into memory to process them
        $users = User::all();

        foreach ($users as $user) {
            $this->processUser($user);
        }
    }
}
```

**Correct**

```php
// app/Http/Controllers/ExportController.php
namespace App\Http\Controllers;

use App\Models\Transaction;
use App\Models\User;
use Illuminate\Support\LazyCollection;
use Symfony\Component\HttpFoundation\StreamedResponse;

class ExportController extends Controller
{
    public function exportTransactions(): StreamedResponse
    {
        return response()->streamDownload(function () {
            $csv = fopen('php://output', 'w');
            fputcsv($csv, ['ID', 'User', 'Amount', 'Date']);

            // GOOD: cursor() returns a LazyCollection -- one row in memory at a time
            // Eager load 'user' to prevent N+1 queries
            Transaction::query()
                ->select(['id', 'user_id', 'amount', 'created_at'])
                ->with('user:id,name')
                ->orderBy('id')
                ->cursor()
                ->each(function (Transaction $transaction) use ($csv) {
                    fputcsv($csv, [
                        $transaction->id,
                        $transaction->user->name,
                        $transaction->amount,
                        $transaction->created_at->toDateString(),
                    ]);
                });

            fclose($csv);
        }, 'transactions.csv', ['Content-Type' => 'text/csv']);
    }

    public function generateReport()
    {
        // GOOD: Use database aggregation instead of loading everything into PHP
        $highValueUsers = User::query()
            ->select(['users.id', 'users.name', 'users.email'])
            ->where('is_active', true)
            ->withSum('orders', 'total')
            ->having('orders_sum_total', '>', 1000)
            ->orderByDesc('orders_sum_total')
            ->paginate(25);

        return view('reports.high-value', compact('highValueUsers'));
    }
}

// app/Console/Commands/ProcessUsers.php
namespace App\Console\Commands;

use App\Models\User;
use Illuminate\Console\Command;

class ProcessUsers extends Command
{
    protected $signature = 'users:process';

    public function handle(): void
    {
        // GOOD: chunk() processes 500 records at a time, freeing memory between chunks
        User::query()
            ->select(['id', 'name', 'email'])
            ->orderBy('id')
            ->chunk(500, function ($users) {
                foreach ($users as $user) {
                    $this->processUser($user);
                }
            });

        // ALTERNATIVE: chunkById() is safer if records are modified during processing
        // It uses WHERE id > ? instead of OFFSET, avoiding skipped/duplicated rows
        User::query()
            ->where('needs_processing', true)
            ->chunkById(500, function ($users) {
                foreach ($users as $user) {
                    $this->processUser($user);
                    $user->update(['needs_processing' => false]);
                }
            });

        // ALTERNATIVE: cursor() for minimal memory when no batching is needed
        User::query()
            ->select(['id', 'name', 'email'])
            ->cursor()
            ->each(fn (User $user) => $this->processUser($user));
    }
}

// app/Providers/AppServiceProvider.php
namespace App\Providers;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // GOOD: Prevent lazy loading in non-production environments
        // Throws a LazyLoadingViolationException when a relationship is accessed
        // without being eager-loaded, catching N+1 issues during development
        Model::preventLazyLoading(! $this->app->isProduction());

        // GOOD: Prevent silently discarding attributes not in $fillable
        Model::preventSilentlyDiscardingAttributes(! $this->app->isProduction());

        // GOOD: Prevent accessing missing attributes (returns null by default)
        Model::preventAccessingMissingAttributes(! $this->app->isProduction());
    }
}

// Production deployment: run these caching commands after every deploy
//
// php artisan route:cache   -- Caches all route registrations into a single file.
//                              Dramatically speeds up route resolution, especially
//                              with hundreds of routes. Must re-run after any route change.
//
// php artisan config:cache  -- Merges all config files into one cached file.
//                              Eliminates filesystem reads for config on each request.
//                              WARNING: config() works but env() returns null after caching.
//                              Always use config() in application code, never env().
//
// php artisan event:cache   -- Caches the event-to-listener mapping so Laravel
//                              doesn't scan provider boot() methods on each request.
//
// php artisan view:cache    -- Pre-compiles all Blade templates so they don't need
//                              to be compiled on first render in production.
//
// Example deploy script:
// php artisan config:cache && php artisan route:cache && php artisan event:cache && php artisan view:cache
```

Reference: [Laravel Collections](https://laravel.com/docs/collections)
