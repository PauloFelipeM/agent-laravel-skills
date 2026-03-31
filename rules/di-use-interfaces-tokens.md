---
title: Use Injection Tokens for Interfaces
impact: HIGH
impactDescription: Essential for swappable implementations
tags:
  - dependency-injection
  - interfaces
  - binding
  - service-provider
---

## Use Injection Tokens for Interfaces

Bind interfaces to implementations in ServiceProviders rather than type-hinting concrete classes directly. This decouples your application code from specific implementations, making it straightforward to swap implementations for different environments (e.g., a real payment gateway in production vs. a fake one in tests), apply the decorator pattern, or switch vendors without touching consumer code. The interface acts as an injection token that the container resolves to whatever concrete class is bound.

**Incorrect**

```php
<?php

namespace App\Services;

// Hard-coded dependency on a specific payment provider
use App\Services\Payment\StripePaymentProcessor;
use App\Services\Notification\TwilioSmsNotifier;
use App\Services\Storage\AwsS3FileStorage;

class CheckoutService
{
    // Tightly coupled to concrete implementations
    // Changing payment provider requires editing this class
    public function __construct(
        private readonly StripePaymentProcessor $payment,
        private readonly TwilioSmsNotifier $sms,
        private readonly AwsS3FileStorage $storage,
    ) {}

    public function process(Order $order): Receipt
    {
        $charge = $this->payment->charge(
            $order->total,
            $order->user->stripeCustomerId,  // Stripe-specific detail leaking in
        );

        $this->sms->sendViaTwilio(  // Twilio-specific method name
            $order->user->phone,
            "Order {$order->id} confirmed!",
        );

        $receiptPath = $this->storage->uploadToS3(  // S3-specific method name
            "receipts/{$order->id}.pdf",
            $charge->receiptPdf,
        );

        return new Receipt($charge->id, $receiptPath);
    }
}

// Testing requires the real Stripe SDK, Twilio credentials, and AWS keys
// or elaborate mocking of concrete classes with vendor-specific methods
```

**Correct**

```php
<?php

// Step 1: Define focused interfaces with vendor-agnostic method names

namespace App\Contracts;

interface PaymentProcessorInterface
{
    public function charge(float $amount, string $customerToken): PaymentResult;
    public function refund(string $chargeId, float $amount): RefundResult;
}

interface SmsNotifierInterface
{
    public function send(string $phoneNumber, string $message): void;
}

interface FileStorageInterface
{
    public function upload(string $path, string $contents): string;
    public function download(string $path): string;
    public function delete(string $path): bool;
}

// Step 2: Create concrete implementations

namespace App\Services\Payment;

use App\Contracts\PaymentProcessorInterface;
use App\Contracts\PaymentResult;
use App\Contracts\RefundResult;
use Stripe\StripeClient;

class StripePaymentProcessor implements PaymentProcessorInterface
{
    public function __construct(
        private readonly StripeClient $stripe,
    ) {}

    public function charge(float $amount, string $customerToken): PaymentResult
    {
        $charge = $this->stripe->charges->create([
            'amount' => (int) ($amount * 100),
            'currency' => 'usd',
            'customer' => $customerToken,
        ]);

        return new PaymentResult(
            chargeId: $charge->id,
            amount: $amount,
            receiptPdf: $charge->receipt_url,
        );
    }

    public function refund(string $chargeId, float $amount): RefundResult
    {
        $refund = $this->stripe->refunds->create([
            'charge' => $chargeId,
            'amount' => (int) ($amount * 100),
        ]);

        return new RefundResult($refund->id, $amount);
    }
}

namespace App\Services\Payment;

use App\Contracts\PaymentProcessorInterface;
use App\Contracts\PaymentResult;
use App\Contracts\RefundResult;

class FakePaymentProcessor implements PaymentProcessorInterface
{
    /** @var array<PaymentResult> */
    public array $charges = [];

    public function charge(float $amount, string $customerToken): PaymentResult
    {
        $result = new PaymentResult(
            chargeId: 'fake_ch_' . uniqid(),
            amount: $amount,
            receiptPdf: 'https://example.com/receipt.pdf',
        );

        $this->charges[] = $result;

        return $result;
    }

    public function refund(string $chargeId, float $amount): RefundResult
    {
        return new RefundResult('fake_re_' . uniqid(), $amount);
    }

    public function assertChargedTotal(float $expected): void
    {
        $actual = array_sum(array_column($this->charges, 'amount'));
        assert($actual === $expected, "Expected total charge of {$expected}, got {$actual}");
    }
}

// Step 3: Bind interfaces to implementations in a ServiceProvider

namespace App\Providers;

use App\Contracts\FileStorageInterface;
use App\Contracts\PaymentProcessorInterface;
use App\Contracts\SmsNotifierInterface;
use App\Services\Notification\TwilioSmsNotifier;
use App\Services\Notification\LogSmsNotifier;
use App\Services\Payment\FakePaymentProcessor;
use App\Services\Payment\StripePaymentProcessor;
use App\Services\Storage\AwsS3FileStorage;
use App\Services\Storage\LocalFileStorage;
use Illuminate\Support\ServiceProvider;

class PaymentServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Environment-based binding: real gateway in production, fake in testing
        $this->app->bind(
            PaymentProcessorInterface::class,
            $this->app->environment('testing')
                ? FakePaymentProcessor::class
                : StripePaymentProcessor::class,
        );

        // Log SMS in local/staging, send real SMS in production
        $this->app->bind(
            SmsNotifierInterface::class,
            $this->app->environment('production')
                ? TwilioSmsNotifier::class
                : LogSmsNotifier::class,
        );

        // Local disk in development, S3 in production
        $this->app->bind(
            FileStorageInterface::class,
            $this->app->environment('local')
                ? LocalFileStorage::class
                : AwsS3FileStorage::class,
        );
    }
}

// Step 4: Consume only interfaces - no vendor lock-in

namespace App\Services;

use App\Contracts\FileStorageInterface;
use App\Contracts\PaymentProcessorInterface;
use App\Contracts\SmsNotifierInterface;
use App\Models\Order;

class CheckoutService
{
    public function __construct(
        private readonly PaymentProcessorInterface $payment,
        private readonly SmsNotifierInterface $sms,
        private readonly FileStorageInterface $storage,
    ) {}

    public function process(Order $order): Receipt
    {
        $result = $this->payment->charge(
            $order->total,
            $order->user->payment_token,
        );

        $this->sms->send(
            $order->user->phone,
            "Order {$order->id} confirmed!",
        );

        $receiptPath = $this->storage->upload(
            "receipts/{$order->id}.pdf",
            $result->receiptPdf,
        );

        return new Receipt($result->chargeId, $receiptPath);
    }
}

// Step 5: Tests use the fake implementation automatically, or inject mocks

namespace Tests\Feature;

use App\Contracts\PaymentProcessorInterface;
use App\Services\Payment\FakePaymentProcessor;
use Tests\TestCase;

class CheckoutTest extends TestCase
{
    public function test_checkout_charges_correct_amount(): void
    {
        // The FakePaymentProcessor is already bound in testing environment
        $order = Order::factory()->create(['total' => 99.99]);

        $response = $this->actingAs($order->user)
            ->postJson("/api/orders/{$order->id}/checkout");

        $response->assertOk();

        // Access the fake to make assertions
        $fake = $this->app->make(PaymentProcessorInterface::class);
        $fake->assertChargedTotal(99.99);
    }
}
```

Reference: [Laravel Service Container](https://laravel.com/docs/container)
