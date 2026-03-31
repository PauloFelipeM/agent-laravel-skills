---
title: Single Responsibility for Services
impact: HIGH
impactDescription: "God classes are the #1 maintainability killer"
tags: [architecture, services, solid, single-responsibility]
---

## Single Responsibility for Services

Each service class should handle exactly one domain concern. When a single service accumulates methods for user management, email sending, payment processing, and report generation, it becomes a "god class" that is impossible to test in isolation, painful to modify, and a merge-conflict magnet. Split these into focused, single-purpose services that can be composed together.

**Incorrect**

```php
// app/Services/UserService.php
namespace App\Services;

use App\Models\User;
use App\Models\Invoice;
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\DB;
use Stripe\StripeClient;

class UserService
{
    private StripeClient $stripe;

    public function __construct()
    {
        $this->stripe = new StripeClient(config('services.stripe.secret'));
    }

    // --- User CRUD ---
    public function createUser(array $data): User
    {
        return User::create($data);
    }

    public function updateUser(User $user, array $data): User
    {
        $user->update($data);
        return $user->refresh();
    }

    public function deleteUser(User $user): void
    {
        $user->delete();
    }

    // --- Email notifications ---
    public function sendWelcomeEmail(User $user): void
    {
        Mail::to($user->email)->send(new \App\Mail\WelcomeEmail($user));
    }

    public function sendPasswordResetEmail(User $user, string $token): void
    {
        Mail::to($user->email)->send(new \App\Mail\PasswordReset($user, $token));
    }

    public function sendInvoiceEmail(User $user, Invoice $invoice): void
    {
        Mail::to($user->email)->send(new \App\Mail\InvoiceMail($user, $invoice));
    }

    // --- Payment processing ---
    public function chargeUser(User $user, float $amount, string $description): object
    {
        return $this->stripe->charges->create([
            'amount'      => (int) ($amount * 100),
            'currency'    => 'usd',
            'customer'    => $user->stripe_customer_id,
            'description' => $description,
        ]);
    }

    public function createSubscription(User $user, string $priceId): object
    {
        return $this->stripe->subscriptions->create([
            'customer' => $user->stripe_customer_id,
            'items'    => [['price' => $priceId]],
        ]);
    }

    public function refundCharge(string $chargeId): object
    {
        return $this->stripe->refunds->create(['charge' => $chargeId]);
    }

    // --- Reporting ---
    public function getMonthlyActiveUsers(): int
    {
        return User::where('last_login_at', '>=', now()->subMonth())->count();
    }

    public function getRevenueReport(string $startDate, string $endDate): array
    {
        return DB::table('payments')
            ->whereBetween('created_at', [$startDate, $endDate])
            ->selectRaw('DATE(created_at) as date, SUM(amount) as total')
            ->groupBy('date')
            ->get()
            ->toArray();
    }

    public function getUserGrowthReport(): array
    {
        return DB::table('users')
            ->selectRaw('DATE_FORMAT(created_at, "%Y-%m") as month, COUNT(*) as signups')
            ->groupBy('month')
            ->orderBy('month')
            ->get()
            ->toArray();
    }
}

// This class has 12+ methods spanning 4 completely unrelated concerns.
// Testing chargeUser() requires setting up email mocks. Changing reporting
// logic risks breaking payment code. Every developer touches this file.
```

**Correct**

```php
// app/Domains/User/Services/UserService.php
namespace App\Domains\User\Services;

use App\Domains\User\Models\User;
use App\Domains\User\Repositories\UserRepositoryInterface;
use App\Domains\User\Events\UserRegistered;

class UserService
{
    public function __construct(
        private readonly UserRepositoryInterface $userRepository,
    ) {}

    public function register(array $data): User
    {
        $user = $this->userRepository->create($data);

        UserRegistered::dispatch($user);

        return $user;
    }

    public function update(User $user, array $data): User
    {
        return $this->userRepository->update($user, $data);
    }

    public function deactivate(User $user): void
    {
        $this->userRepository->update($user, ['is_active' => false]);
    }
}

// app/Domains/Notification/Services/EmailNotificationService.php
namespace App\Domains\Notification\Services;

use App\Domains\User\Models\User;
use App\Domains\Billing\Models\Invoice;
use Illuminate\Support\Facades\Mail;

class EmailNotificationService
{
    public function sendWelcomeEmail(User $user): void
    {
        Mail::to($user->email)->send(new \App\Mail\WelcomeEmail($user));
    }

    public function sendPasswordResetEmail(User $user, string $token): void
    {
        Mail::to($user->email)->send(new \App\Mail\PasswordReset($user, $token));
    }

    public function sendInvoiceEmail(User $user, Invoice $invoice): void
    {
        Mail::to($user->email)->send(new \App\Mail\InvoiceMail($user, $invoice));
    }
}

// app/Domains/Billing/Services/PaymentService.php
namespace App\Domains\Billing\Services;

use App\Domains\User\Models\User;
use App\Domains\Billing\Contracts\PaymentGatewayInterface;

class PaymentService
{
    public function __construct(
        private readonly PaymentGatewayInterface $gateway,
    ) {}

    public function charge(User $user, float $amount, string $description): object
    {
        return $this->gateway->charge(
            customerId: $user->stripe_customer_id,
            amount: $amount,
            currency: 'usd',
            description: $description,
        );
    }

    public function subscribe(User $user, string $priceId): object
    {
        return $this->gateway->createSubscription(
            customerId: $user->stripe_customer_id,
            priceId: $priceId,
        );
    }

    public function refund(string $chargeId): object
    {
        return $this->gateway->refund($chargeId);
    }
}

// app/Domains/Reporting/Services/UserReportService.php
namespace App\Domains\Reporting\Services;

use Illuminate\Support\Facades\DB;

class UserReportService
{
    public function monthlyActiveUsers(): int
    {
        return DB::table('users')
            ->where('last_login_at', '>=', now()->subMonth())
            ->count();
    }

    public function userGrowth(): array
    {
        return DB::table('users')
            ->selectRaw('DATE_FORMAT(created_at, "%Y-%m") as month, COUNT(*) as signups')
            ->groupBy('month')
            ->orderBy('month')
            ->get()
            ->toArray();
    }
}

// app/Domains/Reporting/Services/RevenueReportService.php
namespace App\Domains\Reporting\Services;

use Illuminate\Support\Facades\DB;

class RevenueReportService
{
    public function revenueByDate(string $startDate, string $endDate): array
    {
        return DB::table('payments')
            ->whereBetween('created_at', [$startDate, $endDate])
            ->selectRaw('DATE(created_at) as date, SUM(amount) as total')
            ->groupBy('date')
            ->get()
            ->toArray();
    }
}

// Each service is focused, testable in isolation, and owned by one domain.
// PaymentService can be tested without email concerns.
// ReportService changes never risk breaking user registration.
```

Reference: [Laravel Application Structure](https://laravel.com/docs/structure)
