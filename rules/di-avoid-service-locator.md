---
title: Avoid Service Locator Anti-Pattern
impact: CRITICAL
impactDescription: Hides dependencies and breaks testability
tags:
  - dependency-injection
  - service-locator
  - anti-pattern
  - app-helper
---

## Avoid Service Locator Anti-Pattern

Using `app()`, `resolve()`, or `App::make()` inside service methods to retrieve dependencies is known as the Service Locator anti-pattern. It hides the true dependencies of a class, making the code harder to understand, maintain, and test. When dependencies are resolved inline, there is no way to know what a class needs without reading every line of its implementation. Unit tests become brittle because you must bootstrap the entire service container or use awkward partial mocks instead of simply injecting test doubles through the constructor.

**Incorrect**

```php
<?php

namespace App\Services;

use App\Models\User;

class OrderService
{
    public function createOrder(User $user, array $items): Order
    {
        // Hidden dependency - resolved at runtime via service locator
        $cart = app(CartService::class);
        $totals = $cart->calculateTotals($items);

        // Another hidden dependency
        $tax = resolve(TaxCalculator::class);
        $taxAmount = $tax->calculate($totals, $user->address);

        // Yet another hidden dependency
        $payment = \App::make(PaymentGateway::class);
        $charge = $payment->charge($user, $totals + $taxAmount);

        // Impossible to know all dependencies without reading every line
        $notification = app(NotificationService::class);
        $notification->sendOrderConfirmation($user, $charge);

        return Order::create([
            'user_id' => $user->id,
            'total' => $totals,
            'tax' => $taxAmount,
            'charge_id' => $charge->id,
        ]);
    }
}
```

**Correct**

```php
<?php

namespace App\Services;

use App\Contracts\PaymentGatewayInterface;
use App\Models\User;

class OrderService
{
    public function __construct(
        private readonly CartService $cart,
        private readonly TaxCalculator $tax,
        private readonly PaymentGatewayInterface $payment,
        private readonly NotificationService $notification,
    ) {}

    public function createOrder(User $user, array $items): Order
    {
        $totals = $this->cart->calculateTotals($items);
        $taxAmount = $this->tax->calculate($totals, $user->address);
        $charge = $this->payment->charge($user, $totals + $taxAmount);

        $this->notification->sendOrderConfirmation($user, $charge);

        return Order::create([
            'user_id' => $user->id,
            'total' => $totals,
            'tax' => $taxAmount,
            'charge_id' => $charge->id,
        ]);
    }
}

// Testing is straightforward - inject mocks directly
class OrderServiceTest extends TestCase
{
    public function test_create_order_charges_correct_amount(): void
    {
        $cart = Mockery::mock(CartService::class);
        $cart->shouldReceive('calculateTotals')->andReturn(100.00);

        $tax = Mockery::mock(TaxCalculator::class);
        $tax->shouldReceive('calculate')->andReturn(8.50);

        $payment = Mockery::mock(PaymentGatewayInterface::class);
        $payment->shouldReceive('charge')->once()->with(
            Mockery::type(User::class),
            108.50
        )->andReturn(new Charge(['id' => 'ch_123']));

        $notification = Mockery::mock(NotificationService::class);
        $notification->shouldReceive('sendOrderConfirmation')->once();

        $service = new OrderService($cart, $tax, $payment, $notification);
        $order = $service->createOrder(User::factory()->create(), []);

        $this->assertEquals(108.50, $order->total + $order->tax);
    }
}
```

Reference: [Laravel Service Container](https://laravel.com/docs/container)
