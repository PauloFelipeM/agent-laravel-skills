---
title: Mock External Services in Tests
impact: MEDIUM-HIGH
impactDescription: External service calls make tests slow and flaky
tags: [testing, mocking, facades, http-fake, fakes]
---

## Use Laravel's Built-in Faking Capabilities for External Services

Tests must never make real HTTP calls, send real emails, or interact with live third-party services. Real external calls make tests slow, non-deterministic, and dependent on network availability. Laravel provides first-class faking for HTTP requests, mail, notifications, queues, storage, events, and the bus -- use them to isolate your tests from the outside world while still verifying that your code interacts with those services correctly.

**Incorrect**

```php
<?php

// tests/Feature/PaymentServiceTest.php
// BAD: Making real HTTP calls and sending real emails in tests

namespace Tests\Feature;

use App\Models\Order;
use App\Models\User;
use App\Services\PaymentService;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Mail;
use Tests\TestCase;

class PaymentServiceTest extends TestCase
{
    use RefreshDatabase;

    public function test_process_payment(): void
    {
        $user = User::factory()->create();
        $order = Order::factory()->for($user)->create(['total' => 99.99]);

        // BAD: This hits the real Stripe API
        // - Requires valid API keys in test environment
        // - Costs real money or uses limited sandbox quotas
        // - Fails when the network is down
        // - Slow due to network latency
        $service = new PaymentService();
        $result = $service->charge($order);

        $this->assertTrue($result->successful);
    }

    public function test_sends_order_confirmation(): void
    {
        $user = User::factory()->create([
            'email' => 'customer@example.com',
        ]);
        $order = Order::factory()->for($user)->create();

        // BAD: This sends a real email
        // - Clutters real inboxes or mail providers
        // - Cannot assert email contents reliably
        // - Fails if SMTP server is unreachable
        $order->sendConfirmation();

        // No way to assert the email was actually sent or verify its contents
        $this->assertTrue(true);
    }
}
```

**Correct**

```php
<?php

// tests/Feature/PaymentServiceTest.php

namespace Tests\Feature;

use App\Mail\OrderConfirmation;
use App\Mail\PaymentFailed;
use App\Models\Order;
use App\Models\User;
use App\Notifications\OrderShipped;
use App\Jobs\ProcessPayment;
use App\Jobs\SyncInventory;
use App\Events\OrderPlaced;
use App\Listeners\SendOrderConfirmation;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Bus;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\Notification;
use Illuminate\Support\Facades\Queue;
use Illuminate\Support\Facades\Storage;
use Tests\TestCase;

class PaymentServiceTest extends TestCase
{
    use RefreshDatabase;

    // -------------------------------------------------------
    // Http::fake() for external API calls
    // -------------------------------------------------------

    public function test_successful_payment_via_external_api(): void
    {
        Http::fake([
            'api.stripe.com/v1/charges' => Http::response([
                'id' => 'ch_1234567890',
                'status' => 'succeeded',
                'amount' => 9999,
                'currency' => 'usd',
            ], 200),
        ]);

        $user = User::factory()->create();
        $order = Order::factory()->for($user)->create(['total' => 99.99]);

        $response = $this->actingAs($user)
            ->postJson("/api/orders/{$order->id}/pay", [
                'payment_method' => 'pm_card_visa',
            ]);

        $response->assertOk()
            ->assertJson([
                'data' => [
                    'payment_status' => 'succeeded',
                    'charge_id' => 'ch_1234567890',
                ],
            ]);

        // Verify the correct request was sent to Stripe
        Http::assertSent(function ($request) use ($order) {
            return $request->url() === 'https://api.stripe.com/v1/charges'
                && $request['amount'] === 9999
                && $request['currency'] === 'usd';
        });

        $this->assertDatabaseHas('orders', [
            'id' => $order->id,
            'payment_status' => 'paid',
        ]);
    }

    public function test_handles_payment_gateway_failure(): void
    {
        Http::fake([
            'api.stripe.com/v1/charges' => Http::response([
                'error' => [
                    'type' => 'card_error',
                    'message' => 'Your card was declined.',
                ],
            ], 402),
        ]);

        $user = User::factory()->create();
        $order = Order::factory()->for($user)->create(['total' => 99.99]);

        $response = $this->actingAs($user)
            ->postJson("/api/orders/{$order->id}/pay", [
                'payment_method' => 'pm_card_declined',
            ]);

        $response->assertStatus(402)
            ->assertJson([
                'message' => 'Payment failed: Your card was declined.',
            ]);

        $this->assertDatabaseHas('orders', [
            'id' => $order->id,
            'payment_status' => 'failed',
        ]);
    }

    // -------------------------------------------------------
    // Http::fake() with response sequences
    // -------------------------------------------------------

    public function test_retries_on_gateway_timeout(): void
    {
        Http::fake([
            'api.stripe.com/v1/charges' => Http::sequence()
                ->push(null, 504)           // First attempt: gateway timeout
                ->push(null, 504)           // Second attempt: gateway timeout
                ->push([                    // Third attempt: success
                    'id' => 'ch_retry_success',
                    'status' => 'succeeded',
                    'amount' => 4999,
                ], 200),
        ]);

        $user = User::factory()->create();
        $order = Order::factory()->for($user)->create(['total' => 49.99]);

        $response = $this->actingAs($user)
            ->postJson("/api/orders/{$order->id}/pay", [
                'payment_method' => 'pm_card_visa',
            ]);

        $response->assertOk();

        // Verify three requests were made (two retries + final success)
        Http::assertSentCount(3);
    }

    // -------------------------------------------------------
    // Http::fake() with callback for dynamic responses
    // -------------------------------------------------------

    public function test_geocoding_service_returns_coordinates(): void
    {
        Http::fake(function ($request) {
            if (str_contains($request->url(), 'maps.googleapis.com/maps/api/geocode')) {
                $address = $request['address'] ?? '';

                return Http::response([
                    'results' => [
                        [
                            'formatted_address' => $address,
                            'geometry' => [
                                'location' => [
                                    'lat' => 40.7128,
                                    'lng' => -74.0060,
                                ],
                            ],
                        ],
                    ],
                    'status' => 'OK',
                ], 200);
            }

            return Http::response('Not Found', 404);
        });

        $response = $this->postJson('/api/geocode', [
            'address' => '350 Fifth Avenue, New York, NY',
        ]);

        $response->assertOk()
            ->assertJsonPath('data.lat', 40.7128)
            ->assertJsonPath('data.lng', -74.0060);
    }

    // -------------------------------------------------------
    // Mail::fake() and Mail::assertSent()
    // -------------------------------------------------------

    public function test_order_confirmation_email_is_sent(): void
    {
        Mail::fake();

        $user = User::factory()->create();
        $order = Order::factory()->for($user)->create(['total' => 49.99]);

        $response = $this->actingAs($user)
            ->postJson("/api/orders/{$order->id}/confirm");

        $response->assertOk();

        Mail::assertSent(OrderConfirmation::class, function ($mail) use ($user, $order) {
            return $mail->hasTo($user->email)
                && $mail->order->id === $order->id;
        });
    }

    public function test_failure_notification_sent_on_payment_error(): void
    {
        Mail::fake();

        Http::fake([
            'api.stripe.com/*' => Http::response(['error' => 'declined'], 402),
        ]);

        $user = User::factory()->create();
        $order = Order::factory()->for($user)->create();

        $this->actingAs($user)
            ->postJson("/api/orders/{$order->id}/pay", [
                'payment_method' => 'pm_card_declined',
            ]);

        Mail::assertSent(PaymentFailed::class, function ($mail) use ($user) {
            return $mail->hasTo($user->email);
        });

        // Confirm no success email was sent
        Mail::assertNotSent(OrderConfirmation::class);
    }

    // -------------------------------------------------------
    // Notification::fake()
    // -------------------------------------------------------

    public function test_user_is_notified_when_order_ships(): void
    {
        Notification::fake();

        $user = User::factory()->create();
        $order = Order::factory()->for($user)->create(['status' => 'processing']);

        $response = $this->actingAs($user)
            ->postJson("/api/orders/{$order->id}/ship", [
                'tracking_number' => '1Z999AA10123456784',
                'carrier' => 'ups',
            ]);

        $response->assertOk();

        Notification::assertSentTo(
            $user,
            OrderShipped::class,
            function ($notification) use ($order) {
                return $notification->order->id === $order->id
                    && $notification->trackingNumber === '1Z999AA10123456784';
            }
        );
    }

    public function test_notification_not_sent_for_digital_orders(): void
    {
        Notification::fake();

        $user = User::factory()->create();
        $order = Order::factory()->for($user)->digital()->create();

        $this->actingAs($user)
            ->postJson("/api/orders/{$order->id}/ship");

        Notification::assertNotSentTo($user, OrderShipped::class);
    }

    // -------------------------------------------------------
    // Queue::fake() and Bus::fake()
    // -------------------------------------------------------

    public function test_payment_job_is_dispatched(): void
    {
        Bus::fake();

        $user = User::factory()->create();
        $order = Order::factory()->for($user)->create();

        $response = $this->actingAs($user)
            ->postJson("/api/orders/{$order->id}/pay-async", [
                'payment_method' => 'pm_card_visa',
            ]);

        $response->assertAccepted();

        Bus::assertDispatched(ProcessPayment::class, function ($job) use ($order) {
            return $job->order->id === $order->id;
        });
    }

    public function test_inventory_sync_dispatched_after_order(): void
    {
        Bus::fake([SyncInventory::class]);

        $user = User::factory()->create();

        $response = $this->actingAs($user)
            ->postJson('/api/orders', [
                'items' => [
                    ['product_id' => 1, 'quantity' => 2],
                    ['product_id' => 5, 'quantity' => 1],
                ],
            ]);

        $response->assertCreated();

        Bus::assertDispatched(SyncInventory::class);
        Bus::assertDispatchedTimes(SyncInventory::class, 1);
    }

    public function test_job_is_dispatched_on_correct_queue(): void
    {
        Queue::fake();

        $user = User::factory()->create();
        $order = Order::factory()->for($user)->create();

        $this->actingAs($user)
            ->postJson("/api/orders/{$order->id}/pay-async", [
                'payment_method' => 'pm_card_visa',
            ]);

        Queue::assertPushedOn('payments', ProcessPayment::class);
    }

    // -------------------------------------------------------
    // Storage::fake()
    // -------------------------------------------------------

    public function test_invoice_pdf_is_stored(): void
    {
        Storage::fake('invoices');

        $user = User::factory()->create();
        $order = Order::factory()->for($user)->create();

        $response = $this->actingAs($user)
            ->postJson("/api/orders/{$order->id}/generate-invoice");

        $response->assertOk();

        Storage::disk('invoices')->assertExists("order-{$order->id}.pdf");
    }

    public function test_old_exports_are_cleaned_up(): void
    {
        Storage::fake('exports');

        // Seed some old files
        Storage::disk('exports')->put('report-old.csv', 'old data');

        $this->artisan('exports:cleanup --days=30')
            ->assertExitCode(0);

        Storage::disk('exports')->assertMissing('report-old.csv');
    }

    // -------------------------------------------------------
    // Event::fake()
    // -------------------------------------------------------

    public function test_order_placed_event_is_dispatched(): void
    {
        Event::fake([OrderPlaced::class]);

        $user = User::factory()->create();

        $response = $this->actingAs($user)
            ->postJson('/api/orders', [
                'items' => [
                    ['product_id' => 1, 'quantity' => 1],
                ],
            ]);

        $response->assertCreated();

        Event::assertDispatched(OrderPlaced::class, function ($event) use ($user) {
            return $event->order->user_id === $user->id;
        });
    }

    public function test_event_listeners_are_not_triggered_when_faked(): void
    {
        // Faking specific events prevents their listeners from running,
        // isolating the test to only verify the event was dispatched.
        Event::fake([OrderPlaced::class]);

        $user = User::factory()->create();

        $this->actingAs($user)
            ->postJson('/api/orders', [
                'items' => [
                    ['product_id' => 1, 'quantity' => 1],
                ],
            ]);

        // The SendOrderConfirmation listener did NOT run,
        // so no email was sent. This is intentional -- test
        // the listener separately.
        Event::assertDispatched(OrderPlaced::class);
        Event::assertListening(OrderPlaced::class, SendOrderConfirmation::class);
    }

    // -------------------------------------------------------
    // Preventing stray HTTP requests
    // -------------------------------------------------------

    public function test_no_unexpected_external_calls(): void
    {
        // preventStrayRequests() causes the test to fail if any
        // HTTP request is made that was not explicitly faked.
        Http::preventStrayRequests();

        Http::fake([
            'api.stripe.com/*' => Http::response(['status' => 'ok'], 200),
        ]);

        $user = User::factory()->create();
        $order = Order::factory()->for($user)->create();

        // This will pass because the Stripe call is faked.
        // If the code made any OTHER HTTP call, the test would fail.
        $response = $this->actingAs($user)
            ->postJson("/api/orders/{$order->id}/pay", [
                'payment_method' => 'pm_card_visa',
            ]);

        $response->assertOk();
    }
}
```

Reference: [Laravel Mocking](https://laravel.com/docs/mocking)
