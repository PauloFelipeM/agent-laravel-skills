---
title: Optimize Database Queries
impact: HIGH
impactDescription: "Database is the most common performance bottleneck"
tags: [performance, database, queries, indexing, eloquent]
---

## Optimize Database Queries

The database is the most common bottleneck in Laravel applications. Unoptimized Eloquent queries -- selecting all columns, missing indexes, loading entire tables into memory -- compound quickly as traffic grows. Use `select()` to fetch only the columns you need, add database indexes for columns used in WHERE/ORDER BY/JOIN clauses, use `chunk()` or `cursor()` for large datasets, and consider `toBase()` for read-only queries that do not need Eloquent model hydration.

Use Laravel Debugbar or Telescope in development to identify slow queries, N+1 problems, and duplicate queries.

**Incorrect**

```php
// app/Http/Controllers/OrderController.php
namespace App\Http\Controllers;

use App\Models\Order;
use App\Models\Product;

class OrderController extends Controller
{
    public function index()
    {
        // BAD: SELECT * fetches all columns including large text/blob fields
        $orders = Order::with('customer', 'items.product')
            ->where('status', 'completed')
            ->orderBy('created_at', 'desc')
            ->get();

        return view('orders.index', compact('orders'));
    }

    public function report()
    {
        // BAD: Loading 100k+ rows into memory at once
        $orders = Order::all();

        $totalRevenue = 0;
        foreach ($orders as $order) {
            // BAD: N+1 query -- each iteration fires a new query
            $totalRevenue += $order->items->sum('price');
        }

        // BAD: Using Eloquent models for a simple aggregation
        $topProducts = Product::all()->sortByDesc(function ($product) {
            return $product->orders()->count(); // N+1 again
        })->take(10);

        return view('orders.report', compact('totalRevenue', 'topProducts'));
    }
}

// BAD: Migration without indexes on frequently queried columns
// database/migrations/2024_01_01_create_orders_table.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('orders', function (Blueprint $table) {
            $table->id();
            $table->foreignId('customer_id');
            $table->string('status');         // No index -- slow WHERE filters
            $table->decimal('total', 10, 2);
            $table->text('notes');
            $table->timestamps();             // No index on created_at -- slow ORDER BY
        });
    }
};
```

**Correct**

```php
// app/Http/Controllers/OrderController.php
namespace App\Http\Controllers;

use App\Models\Order;
use App\Models\Product;
use Illuminate\Support\Facades\DB;

class OrderController extends Controller
{
    public function index()
    {
        // GOOD: Select only needed columns, paginate instead of get()
        $orders = Order::query()
            ->select(['id', 'customer_id', 'status', 'total', 'created_at'])
            ->with([
                'customer:id,name,email',
                'items:id,order_id,product_id,quantity,price',
                'items.product:id,name,slug',
            ])
            ->where('status', 'completed')
            ->orderByDesc('created_at')
            ->paginate(25);

        return view('orders.index', compact('orders'));
    }

    public function report()
    {
        // GOOD: Use aggregate queries instead of loading all rows
        $totalRevenue = Order::where('status', 'completed')
            ->sum('total');

        // GOOD: Use a subquery for complex aggregation at the database level
        $topProducts = Product::query()
            ->select(['products.id', 'products.name', 'products.slug'])
            ->selectSub(
                DB::table('order_items')
                    ->selectRaw('COUNT(DISTINCT order_items.order_id)')
                    ->whereColumn('order_items.product_id', 'products.id'),
                'order_count'
            )
            ->orderByDesc('order_count')
            ->limit(10)
            ->get();

        // GOOD: Use toBase() for read-only data that doesn't need Eloquent models
        $monthlySales = Order::query()
            ->selectRaw('DATE_FORMAT(created_at, "%Y-%m") as month')
            ->selectRaw('SUM(total) as revenue')
            ->selectRaw('COUNT(*) as order_count')
            ->where('status', 'completed')
            ->groupByRaw('DATE_FORMAT(created_at, "%Y-%m")')
            ->orderByDesc('month')
            ->limit(12)
            ->toBase()
            ->get();

        return view('orders.report', compact('totalRevenue', 'topProducts', 'monthlySales'));
    }

    // GOOD: Use chunk() for batch processing large datasets
    public function exportCompleted()
    {
        Order::where('status', 'completed')
            ->select(['id', 'customer_id', 'total', 'created_at'])
            ->with('customer:id,name,email')
            ->orderBy('id')
            ->chunk(500, function ($orders) {
                foreach ($orders as $order) {
                    // Process each chunk -- only 500 models in memory at a time
                    $this->csvWriter->addRow([
                        $order->id,
                        $order->customer->name,
                        $order->total,
                        $order->created_at->toDateString(),
                    ]);
                }
            });
    }
}

// GOOD: Migration with proper indexes
// database/migrations/2024_01_01_create_orders_table.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('orders', function (Blueprint $table) {
            $table->id();
            $table->foreignId('customer_id')->constrained()->index();
            $table->string('status');
            $table->decimal('total', 10, 2);
            $table->text('notes')->nullable();
            $table->timestamps();

            // Composite index for the most common query pattern
            $table->index(['status', 'created_at']);

            // Index for reporting queries
            $table->index(['status', 'total']);
        });
    }
};

// Debugging: Install Laravel Debugbar for development query analysis
// composer require barryvdh/laravel-debugbar --dev
//
// Debugbar shows:
// - Number of queries per request
// - Duplicate/N+1 queries highlighted in red
// - Query execution time and EXPLAIN output
// - Memory usage per request
```

Reference: [Laravel Eloquent](https://laravel.com/docs/eloquent)
