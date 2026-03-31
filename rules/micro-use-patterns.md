---
title: Use Message and Event Patterns Correctly
impact: MEDIUM
impactDescription: Wrong patterns cause lost messages and tight coupling
tags: microservices, events, broadcasting, pub-sub, messaging
---

## Use Message and Event Patterns Correctly

Microservices should communicate asynchronously through events and message queues, not through direct synchronous HTTP calls. Direct HTTP coupling means that if one service is down, the calling service fails too, creating cascading failures. Use Laravel Broadcasting for real-time notifications, Laravel Events with queue-backed listeners for async processing, and ensure all event handlers are idempotent so that retries don't cause data corruption.

**Incorrect (direct HTTP calls between services, tight coupling):**

```php
// app/Http/Controllers/OrderController.php
// Directly calling other services via HTTP - tight coupling
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Http;

class OrderController extends Controller
{
    public function store(Request $request)
    {
        $order = Order::create($request->validated());

        // Direct HTTP call to inventory service - blocks the request
        $inventoryResponse = Http::post('http://inventory-service/api/reserve', [
            'items' => $order->items->toArray(),
        ]);

        if ($inventoryResponse->failed()) {
            $order->delete();
            return response()->json(['error' => 'Inventory service unavailable'], 500);
        }

        // Direct HTTP call to notification service - blocks again
        Http::post('http://notification-service/api/send', [
            'user_id' => $order->user_id,
            'message' => 'Your order has been placed!',
        ]);

        // Direct HTTP call to analytics service - blocks again
        Http::post('http://analytics-service/api/track', [
            'event' => 'order_created',
            'data' => $order->toArray(),
        ]);

        // If any service is down, the entire order creation fails
        // or notifications are silently lost.
        return response()->json($order, 201);
    }
}
```

**Correct (event-driven communication with broadcasting, queued listeners, and idempotent handlers):**

```php
// app/Events/OrderCreated.php
namespace App\Events;

use App\Models\Order;
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class OrderCreated implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(
        public readonly Order $order,
        public readonly string $eventId,
    ) {}

    /**
     * Broadcast on a private channel for real-time UI updates.
     */
    public function broadcastOn(): array
    {
        return [
            new Channel('orders.' . $this->order->user_id),
        ];
    }

    public function broadcastAs(): string
    {
        return 'order.created';
    }

    public function broadcastWith(): array
    {
        return [
            'order_id' => $this->order->id,
            'total' => $this->order->total,
            'status' => $this->order->status,
        ];
    }
}

// app/Http/Controllers/OrderController.php
namespace App\Http\Controllers;

use App\Events\OrderCreated;
use App\Http\Requests\StoreOrderRequest;
use Illuminate\Support\Str;

class OrderController extends Controller
{
    public function store(StoreOrderRequest $request)
    {
        $order = Order::create($request->validated());

        // Generate a unique event ID for idempotency
        $eventId = Str::uuid()->toString();

        // Dispatch the event - listeners handle the rest asynchronously
        OrderCreated::dispatch($order, $eventId);

        // The controller returns immediately, the user is not blocked
        return response()->json($order, 201);
    }
}

// app/Listeners/ReserveInventory.php
namespace App\Listeners;

use App\Events\OrderCreated;
use App\Services\InventoryService;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Log;

class ReserveInventory implements ShouldQueue
{
    use InteractsWithQueue;

    public string $queue = 'inventory';

    public int $tries = 5;

    public array $backoff = [10, 30, 60, 120, 300];

    public function __construct(
        private readonly InventoryService $inventoryService,
    ) {}

    public function handle(OrderCreated $event): void
    {
        // Idempotency check: skip if this event was already processed
        $idempotencyKey = "inventory:reserved:{$event->eventId}";

        if (Cache::has($idempotencyKey)) {
            Log::info('Inventory reservation already processed.', [
                'event_id' => $event->eventId,
                'order_id' => $event->order->id,
            ]);
            return;
        }

        $this->inventoryService->reserveItems($event->order);

        // Mark as processed with a TTL matching your business requirements
        Cache::put($idempotencyKey, true, now()->addDays(7));
    }

    /**
     * Handle a job failure. Send to dead-letter queue for manual review.
     */
    public function failed(OrderCreated $event, \Throwable $exception): void
    {
        Log::critical('Failed to reserve inventory after all retries.', [
            'order_id' => $event->order->id,
            'event_id' => $event->eventId,
            'error' => $exception->getMessage(),
        ]);

        // Dispatch to a dead-letter job for manual intervention
        dispatch(new \App\Jobs\DeadLetter\FailedInventoryReservation(
            orderId: $event->order->id,
            eventId: $event->eventId,
            error: $exception->getMessage(),
        ))->onQueue('dead-letter');
    }
}

// app/Listeners/SendOrderNotification.php
namespace App\Listeners;

use App\Events\OrderCreated;
use App\Notifications\OrderPlacedNotification;
use Illuminate\Contracts\Queue\ShouldQueue;

class SendOrderNotification implements ShouldQueue
{
    public string $queue = 'notifications';

    public int $tries = 3;

    public int $backoff = 30;

    public function handle(OrderCreated $event): void
    {
        $event->order->user->notify(
            new OrderPlacedNotification($event->order)
        );
    }

    /**
     * Determine whether the listener should be queued.
     */
    public function shouldQueue(OrderCreated $event): bool
    {
        return $event->order->user->wantsNotifications();
    }
}

// app/Listeners/TrackOrderAnalytics.php
namespace App\Listeners;

use App\Events\OrderCreated;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Support\Facades\Cache;

class TrackOrderAnalytics implements ShouldQueue
{
    public string $queue = 'analytics';

    public int $tries = 3;

    public function handle(OrderCreated $event): void
    {
        // Idempotent: use upsert or check-before-write
        $idempotencyKey = "analytics:tracked:{$event->eventId}";

        if (Cache::has($idempotencyKey)) {
            return;
        }

        // Publish to analytics topic via a dedicated service or direct write
        \App\Models\AnalyticsEvent::create([
            'event_id' => $event->eventId,
            'event_type' => 'order_created',
            'payload' => [
                'order_id' => $event->order->id,
                'total' => $event->order->total,
                'items_count' => $event->order->items->count(),
            ],
            'occurred_at' => now(),
        ]);

        Cache::put($idempotencyKey, true, now()->addDays(7));
    }
}

// app/Providers/EventServiceProvider.php
namespace App\Providers;

use App\Events\OrderCreated;
use App\Listeners\ReserveInventory;
use App\Listeners\SendOrderNotification;
use App\Listeners\TrackOrderAnalytics;
use Illuminate\Foundation\Support\Providers\EventServiceProvider as ServiceProvider;

class EventServiceProvider extends ServiceProvider
{
    protected $listen = [
        OrderCreated::class => [
            ReserveInventory::class,
            SendOrderNotification::class,
            TrackOrderAnalytics::class,
        ],
    ];
}

// config/queue.php - configure separate connections for isolation
return [
    'default' => env('QUEUE_CONNECTION', 'redis'),

    'connections' => [
        'redis' => [
            'driver' => 'redis',
            'connection' => 'queue',
            'queue' => env('REDIS_QUEUE', 'default'),
            'retry_after' => 90,
            'block_for' => null,
            'after_commit' => true, // Only dispatch after DB transaction commits
        ],

        'sqs' => [
            'driver' => 'sqs',
            'key' => env('AWS_ACCESS_KEY_ID'),
            'secret' => env('AWS_SECRET_ACCESS_KEY'),
            'prefix' => env('SQS_PREFIX', 'https://sqs.us-east-1.amazonaws.com/your-account-id'),
            'queue' => env('SQS_QUEUE', 'default'),
            'suffix' => env('SQS_SUFFIX'),
            'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
            'after_commit' => true,
        ],
    ],
];
```

Reference: [Laravel Broadcasting Documentation](https://laravel.com/docs/broadcasting)
