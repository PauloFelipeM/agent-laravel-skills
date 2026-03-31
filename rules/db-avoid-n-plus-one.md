---
title: Avoid N+1 Query Problems
impact: HIGH
impactDescription: N+1 queries are one of the most common performance killers
tags: [database, n-plus-one, queries, eager-loading, with]
---

## Avoid N+1 Query Problems

N+1 queries occur when you load a collection of models and then access a relationship on each model inside a loop. This results in 1 query to fetch the collection plus N additional queries (one per model) to fetch the related data. On a page displaying 100 posts with their authors, that means 101 queries instead of 2.

Laravel provides eager loading via `with()` to solve this. You should also enable strict mode in development to catch lazy loading violations early.

**Incorrect**

```php
// Controller - triggers N+1: 1 query for posts + 1 query per post for author
$posts = Post::all();

// Blade template
@foreach ($posts as $post)
    <p>{{ $post->title }} by {{ $post->author->name }}</p>  {{-- Lazy loads author each iteration --}}
@endforeach

// Another common N+1 - accessing nested relationships
$orders = Order::all();
foreach ($orders as $order) {
    echo $order->customer->address->city; // 2 extra queries per order
}

// N+1 inside an accessor or computed property
class Post extends Model
{
    public function getCommentCountLabelAttribute(): string
    {
        // This triggers a query every time the attribute is accessed
        return $this->comments->count() . ' comments';
    }
}
```

**Correct**

```php
// Eager load relationships with with()
$posts = Post::with('author')->get();

// Eager load nested relationships using dot notation
$orders = Order::with('customer.address')->get();

// Eager load multiple relationships at once
$posts = Post::with(['author', 'comments', 'tags'])->get();

// Constrain eager loads to fetch only what you need
$posts = Post::with(['comments' => function (Builder $query) {
    $query->where('approved', true)->latest()->limit(5);
}])->get();

// Use withCount() when you only need the count, not the full relation
$posts = Post::withCount('comments')->get();

foreach ($posts as $post) {
    echo "{$post->title} has {$post->comments_count} comments";
}

// Use loadCount() on an existing collection
$posts = Post::all();
$posts->loadCount('comments');

// Lazy eager loading - when you already have a collection and need to load relations
$posts = Post::all();
$posts->load('author'); // Single query to load all authors

// Lazy eager loading on a single model
$post = Post::find(1);
$post->load(['comments', 'tags']);

// Use withAggregate for sums, averages, min, max
$customers = Customer::withSum('orders', 'total')
    ->withAvg('orders', 'total')
    ->get();

foreach ($customers as $customer) {
    echo "Total spent: {$customer->orders_sum_total}";
    echo "Average order: {$customer->orders_avg_total}";
}

// Prevent lazy loading in non-production to catch N+1 problems early
// app/Providers/AppServiceProvider.php
use Illuminate\Database\Eloquent\Model;

public function boot(): void
{
    Model::preventLazyLoading(! $this->app->isProduction());

    // Optionally log instead of throwing exceptions in production
    Model::handleLazyLoadingViolationUsing(function (Model $model, string $relation) {
        logger()->warning("Lazy loading [{$relation}] on model [{$model::class}].");
    });
}

// Use WithoutRelations when dispatching queued jobs to avoid serializing loaded relations
use Illuminate\Queue\SerializesModels;

class ProcessPodcast implements ShouldQueue
{
    use SerializesModels;

    public function __construct(
        public Podcast $podcast,
    ) {
        // Strip loaded relations so only the model ID is serialized
        $this->podcast = $podcast->withoutRelations();
    }

    public function handle(): void
    {
        // Re-load only the relations this job actually needs
        $this->podcast->load('episodes');
    }
}
```

**Reference:** [Laravel Eloquent Relationships - Eager Loading](https://laravel.com/docs/eloquent-relationships#eager-loading)
