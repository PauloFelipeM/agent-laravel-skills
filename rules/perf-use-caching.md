---
title: Use Caching Strategically
impact: HIGH
impactDescription: "Proper caching can reduce response times by 90%+"
tags: [performance, caching, redis, cache-tags, remember]
---

## Use Caching Strategically

Caching is one of the most impactful performance optimizations available. However, caching everything indiscriminately leads to stale data, high memory usage, and hard-to-debug inconsistencies. The key is to cache expensive or frequently-accessed data with clear invalidation strategies. Laravel provides `Cache::remember()` for transparent caching, cache tags for grouped invalidation, and artisan commands for config/route/view caching in production.

Always pair caching with explicit invalidation -- typically via model observers or events -- so users never see stale data.

**Incorrect**

```php
// app/Http/Controllers/ProductController.php
namespace App\Http\Controllers;

use App\Models\Product;
use App\Models\Category;

class ProductController extends Controller
{
    // BAD: No caching on an expensive query that runs on every request
    public function index()
    {
        // This query with aggregations runs on every page load
        $products = Product::with(['category', 'reviews', 'images'])
            ->withAvg('reviews', 'rating')
            ->withCount('orders')
            ->where('is_active', true)
            ->orderByDesc('orders_count')
            ->paginate(20);

        return view('products.index', compact('products'));
    }

    // BAD: Caching everything with no invalidation strategy
    public function show(Product $product)
    {
        // Cached forever -- stale data when product is updated
        $product = Cache::rememberForever("product_{$product->id}", function () use ($product) {
            return Product::with(['category', 'reviews', 'images', 'variants'])
                ->find($product->id);
        });

        // BAD: Caching trivially cheap operations wastes memory
        $appName = Cache::remember('app_name', 3600, function () {
            return config('app.name'); // config() is already fast
        });

        return view('products.show', compact('product'));
    }

    // BAD: No way to invalidate related caches when data changes
    public function update(Request $request, Product $product)
    {
        $product->update($request->validated());

        // Forgot to clear the cache -- users see stale data
        return redirect()->route('products.show', $product);
    }
}
```

**Correct**

```php
// app/Http/Controllers/ProductController.php
namespace App\Http\Controllers;

use App\Models\Product;
use Illuminate\Support\Facades\Cache;

class ProductController extends Controller
{
    public function index()
    {
        // Cache the expensive query for 15 minutes
        // Use cache tags so we can invalidate all product listing caches at once
        $products = Cache::tags(['products', 'product-listings'])
            ->remember('products:index:page:' . request('page', 1), now()->addMinutes(15), function () {
                return Product::with(['category', 'reviews', 'images'])
                    ->withAvg('reviews', 'rating')
                    ->withCount('orders')
                    ->where('is_active', true)
                    ->orderByDesc('orders_count')
                    ->paginate(20);
            });

        return view('products.index', compact('products'));
    }

    public function show(Product $product)
    {
        // Cache individual product with a reasonable TTL
        $product = Cache::tags(['products', "product:{$product->id}"])
            ->remember("products:{$product->id}:detail", now()->addHours(1), function () use ($product) {
                return Product::with(['category', 'reviews', 'images', 'variants'])
                    ->find($product->id);
            });

        // Cache expensive aggregation separately (changes less often)
        $relatedProducts = Cache::tags(['products', "product:{$product->id}"])
            ->remember("products:{$product->id}:related", now()->addHours(6), function () use ($product) {
                return Product::where('category_id', $product->category_id)
                    ->where('id', '!=', $product->id)
                    ->withAvg('reviews', 'rating')
                    ->orderByDesc('reviews_avg_rating')
                    ->limit(4)
                    ->get();
            });

        return view('products.show', compact('product', 'relatedProducts'));
    }
}

// app/Observers/ProductObserver.php
namespace App\Observers;

use App\Models\Product;
use Illuminate\Support\Facades\Cache;

class ProductObserver
{
    /**
     * Invalidate caches when a product is created, updated, or deleted.
     * Cache tags allow flushing all related keys in a single call.
     */
    public function saved(Product $product): void
    {
        // Flush all caches tagged with this specific product
        Cache::tags(["product:{$product->id}"])->flush();

        // Flush listing caches since ordering/content may have changed
        Cache::tags(['product-listings'])->flush();
    }

    public function deleted(Product $product): void
    {
        Cache::tags(["product:{$product->id}"])->flush();
        Cache::tags(['product-listings'])->flush();
    }
}

// app/Providers/AppServiceProvider.php
namespace App\Providers;

use App\Models\Product;
use App\Observers\ProductObserver;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        Product::observe(ProductObserver::class);
    }
}

// config/cache.php -- Redis configuration for tag support
// Cache tags require a tag-aware driver: redis, memcached, or array
return [
    'default' => env('CACHE_STORE', 'redis'),

    'stores' => [
        'redis' => [
            'driver' => 'redis',
            'connection' => env('CACHE_REDIS_CONNECTION', 'cache'),
            'lock_connection' => env('CACHE_REDIS_LOCK_CONNECTION', 'default'),
        ],
    ],

    'prefix' => env('CACHE_PREFIX', 'myapp_cache_'),
];

// Production deployment script -- always run these after deploy:
// php artisan config:cache   -- merges config files into a single cached file
// php artisan route:cache    -- compiles route registrations into a cached file
// php artisan view:cache     -- pre-compiles all Blade templates
// php artisan event:cache    -- caches event/listener mappings
```

Reference: [Laravel Cache](https://laravel.com/docs/cache)
