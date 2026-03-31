---
title: Use Event-Driven Architecture for Decoupling
impact: MEDIUM-HIGH
impactDescription: Reduces module coupling significantly
tags: [architecture, events, listeners, decoupling]
---

## Use Event-Driven Architecture for Decoupling

When an action in one module needs to trigger side-effects in other modules, use Laravel's Event/Listener system instead of calling those modules directly. This way the originating module does not need to know about or depend on the downstream modules. New side-effects can be added by registering new listeners without modifying the original code.

**Incorrect**

```php
// app/Domains/Order/Services/OrderService.php
namespace App\Domains\Order\Services;

use App\Domains\Inventory\Services\InventoryService;
use App\Domains\Notification\Services\NotificationService;
use App\Domains\Analytics\Services\AnalyticsService;
use App\Domains\Loyalty\Services\LoyaltyPointsService;
use App\Domains\Order\Models\Order;

class OrderService
{
    public function __construct(
        private readonly InventoryService $inventoryService,
        private readonly NotificationService $notificationService,
        private readonly AnalyticsService $analyticsService,
        private readonly LoyaltyPointsService $loyaltyService,
    ) {}

    public function placeOrder(int $userId, array $items): Order
    {
        $order = Order::create(['user_id' => $userId, 'status' => 'placed']);

        foreach ($items as $item) {
            $order->items()->create($item);
        }

        // OrderService directly calls into 4 other modules.
        // Adding a 5th side-effect means modifying this class.
        $this->inventoryService->decrementStock($order);
        $this->notificationService->sendOrderConfirmation($order);
        $this->analyticsService->trackPurchase($order);
        $this->loyaltyService->awardPoints($order->user, $order->total);

        return $order;
    }
}

// Problems:
// - OrderService depends on 4 unrelated modules
// - Every new side-effect requires editing OrderService
// - A failure in AnalyticsService can break the order flow
// - Testing placeOrder() requires mocking 4 services
// - Violates Open/Closed Principle
```

**Correct**

```php
// Step 1: Define the event

// app/Domains/Order/Events/OrderPlaced.php
namespace App\Domains\Order\Events;

use App\Domains\Order\Models\Order;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class OrderPlaced
{
    use Dispatchable;
    use SerializesModels;

    public function __construct(
        public readonly Order $order,
    ) {}
}

// Step 2: The OrderService only dispatches the event -- no cross-module calls

// app/Domains/Order/Services/OrderService.php
namespace App\Domains\Order\Services;

use App\Domains\Order\Events\OrderPlaced;
use App\Domains\Order\Models\Order;

class OrderService
{
    public function placeOrder(int $userId, array $items): Order
    {
        $order = Order::create(['user_id' => $userId, 'status' => 'placed']);

        foreach ($items as $item) {
            $order->items()->create($item);
        }

        // Single dispatch -- OrderService has zero knowledge of downstream modules
        OrderPlaced::dispatch($order);

        return $order;
    }
}

// Step 3: Each module registers its own listener for the event

// app/Domains/Inventory/Listeners/DecrementStockOnOrderPlaced.php
namespace App\Domains\Inventory\Listeners;

use App\Domains\Inventory\Services\InventoryService;
use App\Domains\Order\Events\OrderPlaced;
use Illuminate\Contracts\Queue\ShouldQueue;

class DecrementStockOnOrderPlaced implements ShouldQueue
{
    public function __construct(
        private readonly InventoryService $inventoryService,
    ) {}

    public function handle(OrderPlaced $event): void
    {
        foreach ($event->order->items as $item) {
            $this->inventoryService->decrementStock(
                productId: $item->product_id,
                quantity: $item->quantity,
            );
        }
    }

    public function failed(OrderPlaced $event, \Throwable $exception): void
    {
        // Handle failure -- e.g., flag the order for manual review
        $event->order->update(['requires_stock_review' => true]);
    }
}

// app/Domains/Notification/Listeners/SendOrderConfirmationEmail.php
namespace App\Domains\Notification\Listeners;

use App\Domains\Notification\Mail\OrderConfirmationMail;
use App\Domains\Order\Events\OrderPlaced;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Support\Facades\Mail;

class SendOrderConfirmationEmail implements ShouldQueue
{
    public string $queue = 'notifications';

    public function handle(OrderPlaced $event): void
    {
        Mail::to($event->order->user->email)->send(
            new OrderConfirmationMail($event->order),
        );
    }
}

// app/Domains/Loyalty/Listeners/AwardLoyaltyPointsOnOrderPlaced.php
namespace App\Domains\Loyalty\Listeners;

use App\Domains\Loyalty\Services\LoyaltyPointsService;
use App\Domains\Order\Events\OrderPlaced;
use Illuminate\Contracts\Queue\ShouldQueue;

class AwardLoyaltyPointsOnOrderPlaced implements ShouldQueue
{
    public string $queue = 'loyalty';

    public function __construct(
        private readonly LoyaltyPointsService $loyaltyService,
    ) {}

    public function handle(OrderPlaced $event): void
    {
        $this->loyaltyService->awardPoints(
            user: $event->order->user,
            points: (int) floor($event->order->total),
        );
    }
}

// Step 4: Register events and listeners in EventServiceProvider

// app/Providers/EventServiceProvider.php
namespace App\Providers;

use App\Domains\Inventory\Listeners\DecrementStockOnOrderPlaced;
use App\Domains\Loyalty\Listeners\AwardLoyaltyPointsOnOrderPlaced;
use App\Domains\Notification\Listeners\SendOrderConfirmationEmail;
use App\Domains\Order\Events\OrderPlaced;
use Illuminate\Foundation\Support\Providers\EventServiceProvider as ServiceProvider;

class EventServiceProvider extends ServiceProvider
{
    protected $listen = [
        OrderPlaced::class => [
            DecrementStockOnOrderPlaced::class,
            SendOrderConfirmationEmail::class,
            AwardLoyaltyPointsOnOrderPlaced::class,
            // Adding a new side-effect is a one-line change here.
            // No modification to OrderService required.
        ],
    ];
}

// Benefits:
// - OrderService has zero dependencies on other modules
// - Each listener runs on its own queue and handles its own failures
// - Adding a new side-effect = create a listener + one line in EventServiceProvider
// - Testing OrderService only requires asserting the event was dispatched:
//
//   Event::fake();
//   $service->placeOrder($userId, $items);
//   Event::assertDispatched(OrderPlaced::class);
```

Reference: [Laravel Events](https://laravel.com/docs/events)
