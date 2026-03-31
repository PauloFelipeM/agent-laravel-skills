---
title: Prefer Constructor Injection
impact: HIGH
impactDescription: Makes dependencies explicit and testable
tags:
  - dependency-injection
  - constructor
  - facades
  - testing
---

## Prefer Constructor Injection

Prefer constructor injection over Facades or the `app()` helper. Facades and service locators hide dependencies behind static calls, making it impossible to see what a class needs at a glance. Constructor injection lists every dependency explicitly, enables straightforward mocking in tests, and lets static analysis tools verify that all requirements are satisfied. Laravel's service container automatically resolves constructor parameters for controllers, jobs, listeners, and any class resolved through the container.

**Incorrect**

```php
<?php

namespace App\Services;

use App\Models\Order;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\Notification;

class OrderFulfillmentService
{
    public function fulfill(Order $order): void
    {
        // Facade static call - hidden dependency
        DB::beginTransaction();

        try {
            $order->update(['status' => 'processing']);

            // Another Facade - impossible to see from class signature
            $inventory = app(InventoryService::class);
            $inventory->reserve($order->items);

            // Facade for shipping
            $tracking = \App::make(ShippingService::class)->ship($order);

            $order->update([
                'status' => 'shipped',
                'tracking_number' => $tracking->number,
            ]);

            // Facade for mail
            Mail::to($order->user)->send(new OrderShippedMail($order));

            // Facade for notification
            Notification::send($order->user, new OrderShippedNotification($order));

            // Facade for cache
            Cache::forget("user.{$order->user_id}.orders");

            DB::commit();

            // Facade for logging
            Log::info('Order fulfilled', ['order_id' => $order->id]);
        } catch (\Throwable $e) {
            DB::rollBack();
            Log::error('Order fulfillment failed', [
                'order_id' => $order->id,
                'error' => $e->getMessage(),
            ]);
            throw $e;
        }
    }
}
```

**Correct**

```php
<?php

namespace App\Services;

use App\Contracts\ShippingServiceInterface;
use App\Mail\OrderShippedMail;
use App\Models\Order;
use App\Notifications\OrderShippedNotification;
use Illuminate\Contracts\Cache\Repository as CacheRepository;
use Illuminate\Database\ConnectionInterface;
use Psr\Log\LoggerInterface;
use Illuminate\Contracts\Mail\Mailer;
use Illuminate\Contracts\Notifications\Dispatcher as NotificationDispatcher;

class OrderFulfillmentService
{
    public function __construct(
        private readonly ConnectionInterface $db,
        private readonly InventoryService $inventory,
        private readonly ShippingServiceInterface $shipping,
        private readonly Mailer $mailer,
        private readonly NotificationDispatcher $notifications,
        private readonly CacheRepository $cache,
        private readonly LoggerInterface $logger,
    ) {}

    public function fulfill(Order $order): void
    {
        $this->db->beginTransaction();

        try {
            $order->update(['status' => 'processing']);

            $this->inventory->reserve($order->items);

            $tracking = $this->shipping->ship($order);

            $order->update([
                'status' => 'shipped',
                'tracking_number' => $tracking->number,
            ]);

            $this->mailer->to($order->user)->send(new OrderShippedMail($order));
            $this->notifications->send($order->user, new OrderShippedNotification($order));
            $this->cache->forget("user.{$order->user_id}.orders");

            $this->db->commit();

            $this->logger->info('Order fulfilled', ['order_id' => $order->id]);
        } catch (\Throwable $e) {
            $this->db->rollBack();
            $this->logger->error('Order fulfillment failed', [
                'order_id' => $order->id,
                'error' => $e->getMessage(),
            ]);
            throw $e;
        }
    }
}

// Testing becomes straightforward - inject mocks for each dependency
namespace Tests\Unit\Services;

use App\Contracts\ShippingServiceInterface;
use App\Models\Order;
use App\Services\InventoryService;
use App\Services\OrderFulfillmentService;
use Illuminate\Contracts\Cache\Repository as CacheRepository;
use Illuminate\Contracts\Mail\Mailer;
use Illuminate\Contracts\Notifications\Dispatcher as NotificationDispatcher;
use Illuminate\Database\ConnectionInterface;
use Mockery;
use Psr\Log\LoggerInterface;
use Tests\TestCase;

class OrderFulfillmentServiceTest extends TestCase
{
    private OrderFulfillmentService $service;
    private $shipping;
    private $inventory;
    private $mailer;

    protected function setUp(): void
    {
        parent::setUp();

        $db = Mockery::mock(ConnectionInterface::class);
        $db->shouldReceive('beginTransaction', 'commit')->once();

        $this->inventory = Mockery::mock(InventoryService::class);
        $this->inventory->shouldReceive('reserve')->once();

        $this->shipping = Mockery::mock(ShippingServiceInterface::class);

        $this->mailer = Mockery::mock(Mailer::class);
        $this->mailer->shouldReceive('to->send')->once();

        $notifications = Mockery::mock(NotificationDispatcher::class);
        $notifications->shouldReceive('send')->once();

        $cache = Mockery::mock(CacheRepository::class);
        $cache->shouldReceive('forget')->once();

        $logger = Mockery::mock(LoggerInterface::class);
        $logger->shouldReceive('info')->once();

        $this->service = new OrderFulfillmentService(
            db: $db,
            inventory: $this->inventory,
            shipping: $this->shipping,
            mailer: $this->mailer,
            notifications: $notifications,
            cache: $cache,
            logger: $logger,
        );
    }

    public function test_fulfill_ships_order_and_notifies_user(): void
    {
        $tracking = new \stdClass();
        $tracking->number = 'TRACK-123';

        $this->shipping->shouldReceive('ship')->once()->andReturn($tracking);

        $order = Order::factory()->create(['status' => 'pending']);

        $this->service->fulfill($order);

        $this->assertEquals('shipped', $order->fresh()->status);
        $this->assertEquals('TRACK-123', $order->fresh()->tracking_number);
    }
}
```

Reference: [Laravel Service Container](https://laravel.com/docs/container)
