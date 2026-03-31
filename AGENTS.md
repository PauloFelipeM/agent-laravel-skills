# Laravel Best Practices

**Version 1.0.0**
Laravel Best Practices
2026-03-30

> **Note:**
> This document is mainly for agents and LLMs to follow when maintaining,
> generating, or refactoring Laravel codebases. Humans may also find it
> useful, but guidance here is optimized for automation and consistency
> by AI-assisted workflows.

---

## Abstract

This document provides a comprehensive set of best practices for Laravel applications, covering architecture, dependency injection, error handling, security, performance, testing, database patterns, API design, microservices, and DevOps. Each rule includes impact assessment, incorrect/correct code examples, and references to official Laravel documentation. Rules are prioritized by impact level to guide automated refactoring and AI-assisted code generation.

---

## Table of Contents

1. [Architecture](#1-architecture) — **CRITICAL**
   - 1.1 [Avoid Circular Dependencies](#11-avoid-circular-dependencies)
   - 1.2 [Organize by Feature Modules](#12-organize-by-feature-modules)
   - 1.3 [Single Responsibility for Services](#13-single-responsibility-for-services)
   - 1.4 [Use Event-Driven Architecture for Decoupling](#14-use-event-driven-architecture-for-decoupling)
   - 1.5 [Use Repository Pattern for Data Access](#15-use-repository-pattern-for-data-access)
2. [Dependency Injection](#2-dependency-injection) — **CRITICAL**
   - 2.1 [Avoid Service Locator Anti-Pattern](#21-avoid-service-locator-anti-pattern)
   - 2.2 [Apply Interface Segregation Principle](#22-apply-interface-segregation-principle)
   - 2.3 [Honor Liskov Substitution Principle](#23-honor-liskov-substitution-principle)
   - 2.4 [Prefer Constructor Injection](#24-prefer-constructor-injection)
   - 2.5 [Understand Provider Scopes](#25-understand-provider-scopes)
   - 2.6 [Use Injection Tokens for Interfaces](#26-use-injection-tokens-for-interfaces)
3. [Error Handling](#3-error-handling) — **HIGH**
   - 3.1 [Handle Queue and Job Errors Properly](#31-handle-queue-and-job-errors-properly)
   - 3.2 [Throw HTTP Exceptions from Services](#32-throw-http-exceptions-from-services)
   - 3.3 [Use Exception Handler for Error Handling](#33-use-exception-handler-for-error-handling)
4. [Security](#4-security) — **HIGH**
   - 4.1 [Implement Secure JWT Authentication](#41-implement-secure-jwt-authentication)
   - 4.2 [Implement Rate Limiting](#42-implement-rate-limiting)
   - 4.3 [Sanitize Output to Prevent XSS](#43-sanitize-output-to-prevent-xss)
   - 4.4 [Use Guards for Authentication and Authorization](#44-use-guards-for-authentication-and-authorization)
   - 4.5 [Validate All Input with Form Requests](#45-validate-all-input-with-form-requests)
5. [Performance](#5-performance) — **HIGH**
   - 5.1 [Use Lazy Loading and Route Caching](#51-use-lazy-loading-and-route-caching)
   - 5.2 [Optimize Database Queries](#52-optimize-database-queries)
   - 5.3 [Use Service Provider Lifecycle Correctly](#53-use-service-provider-lifecycle-correctly)
   - 5.4 [Use Caching Strategically](#54-use-caching-strategically)
6. [Testing](#6-testing) — **MEDIUM-HIGH**
   - 6.1 [Use HTTP Tests for Feature Testing](#61-use-http-tests-for-feature-testing)
   - 6.2 [Mock External Services in Tests](#62-mock-external-services-in-tests)
   - 6.3 [Use Laravel TestCase and RefreshDatabase](#63-use-laravel-testcase-and-refreshdatabase)
7. [Database & ORM](#7-database-orm) — **MEDIUM-HIGH**
   - 7.1 [Avoid N+1 Query Problems](#71-avoid-n-1-query-problems)
   - 7.2 [Use Database Migrations](#72-use-database-migrations)
   - 7.3 [Use Transactions for Multi-Step Operations](#73-use-transactions-for-multi-step-operations)
8. [API Design](#8-api-design) — **MEDIUM**
   - 8.1 [Use API Resources for Response Serialization](#81-use-api-resources-for-response-serialization)
   - 8.2 [Use Middleware for Cross-Cutting Concerns](#82-use-middleware-for-cross-cutting-concerns)
   - 8.3 [Use Form Requests for Input Transformation](#83-use-form-requests-for-input-transformation)
   - 8.4 [Use API Versioning for Breaking Changes](#84-use-api-versioning-for-breaking-changes)
9. [Microservices](#9-microservices) — **MEDIUM**
   - 9.1 [Implement Health Checks for Microservices](#91-implement-health-checks-for-microservices)
   - 9.2 [Use Message and Event Patterns Correctly](#92-use-message-and-event-patterns-correctly)
   - 9.3 [Use Message Queues for Background Jobs](#93-use-message-queues-for-background-jobs)
10. [DevOps & Deployment](#10-devops-deployment) — **LOW-MEDIUM**
   - 10.1 [Implement Graceful Shutdown](#101-implement-graceful-shutdown)
   - 10.2 [Use Config for Environment Configuration](#102-use-config-for-environment-configuration)
   - 10.3 [Use Structured Logging](#103-use-structured-logging)

---

## 1. Architecture

**Section Impact: CRITICAL**

### 1.1 Avoid Circular Dependencies

**Impact: CRITICAL** — "#1 cause of runtime crashes"

Circular dependencies occur when two or more Service Providers depend on each other, creating an unresolvable dependency loop. Laravel's service container will throw a `BindingResolutionException` or cause infinite recursion at runtime. This is especially dangerous because it may only surface under specific request paths or during queue processing.

The solution is to extract shared logic into a dedicated third provider, or use Laravel's Event system to decouple the communication between providers entirely.

**Incorrect**

```php
// app/Providers/BillingServiceProvider.php
namespace App\Providers;

use App\Services\Billing\BillingService;
use App\Services\User\UserService;
use Illuminate\Support\ServiceProvider;

class BillingServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->singleton(BillingService::class, function ($app) {
            // BillingService depends on UserService
            return new BillingService($app->make(UserService::class));
        });
    }
}

// app/Providers/UserServiceProvider.php
namespace App\Providers;

use App\Services\Billing\BillingService;
use App\Services\User\UserService;
use Illuminate\Support\ServiceProvider;

class UserServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->singleton(UserService::class, function ($app) {
            // UserService depends on BillingService -> circular!
            return new BillingService($app->make(BillingService::class));
        });
    }
}

// At runtime, resolving either service triggers resolution of the other,
// resulting in an infinite loop or BindingResolutionException.
```

**Correct**

```php
// Option A: Extract shared logic into a third provider

// app/Services/Shared/AccountBalanceResolver.php
namespace App\Services\Shared;

use App\Models\User;

class AccountBalanceResolver
{
    public function getBalance(User $user): float
    {
        return $user->transactions()->sum('amount');
    }

    public function hasSufficientFunds(User $user, float $amount): bool
    {
        return $this->getBalance($user) >= $amount;
    }
}

// app/Providers/SharedServiceProvider.php
namespace App\Providers;

use App\Services\Shared\AccountBalanceResolver;
use Illuminate\Support\ServiceProvider;

class SharedServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->singleton(AccountBalanceResolver::class);
    }
}

// app/Services/Billing/BillingService.php
namespace App\Services\Billing;

use App\Models\User;
use App\Services\Shared\AccountBalanceResolver;

class BillingService
{
    public function __construct(
        private readonly AccountBalanceResolver $balanceResolver,
    ) {}

    public function charge(User $user, float $amount): bool
    {
        if (! $this->balanceResolver->hasSufficientFunds($user, $amount)) {
            throw new InsufficientFundsException($user, $amount);
        }

        return $user->transactions()->create([
            'amount' => -$amount,
            'type'   => 'charge',
        ]);
    }
}

// app/Services/User/UserService.php
namespace App\Services\User;

use App\Models\User;
use App\Services\Shared\AccountBalanceResolver;

class UserService
{
    public function __construct(
        private readonly AccountBalanceResolver $balanceResolver,
    ) {}

    public function getDashboardSummary(User $user): array
    {
        return [
            'balance'  => $this->balanceResolver->getBalance($user),
            'name'     => $user->name,
            'is_active' => $user->is_active,
        ];
    }
}

// Option B: Use Events for decoupled cross-module communication

// app/Events/UserAccountCharged.php
namespace App\Events;

use App\Models\User;
use Illuminate\Foundation\Events\Dispatchable;

class UserAccountCharged
{
    use Dispatchable;

    public function __construct(
        public readonly User $user,
        public readonly float $amount,
    ) {}
}

// app/Listeners/UpdateUserBalanceCache.php
namespace App\Listeners;

use App\Events\UserAccountCharged;
use Illuminate\Support\Facades\Cache;

class UpdateUserBalanceCache
{
    public function handle(UserAccountCharged $event): void
    {
        Cache::forget("user:{$event->user->id}:balance");
    }
}

// BillingService fires the event; UserService never needs to know about it.
// The listener handles the cross-module side-effect.
```

Reference: [Laravel Service Providers](https://laravel.com/docs/providers)

---

### 1.2 Organize by Feature Modules

**Impact: HIGH** — Improves maintainability and team scalability

Keep your project organized with clear separation of concerns. The standard Laravel structure works well for most projects — just ensure you include all necessary layers (Services, Repositories, Resources). For larger projects with multiple teams, consider a domain-driven structure where each feature is self-contained.

**Incorrect (missing layers in standard structure):**

```php
// Standard structure missing Services, Repositories, and Resources
// app/
//   Http/
//     Controllers/
//       UserController.php
//       OrderController.php
//   Models/
//     User.php
//     Order.php

// Controller doing too much — no service layer, no resources
// app/Http/Controllers/OrderController.php
namespace App\Http\Controllers;

use App\Models\Order;
use Illuminate\Http\Request;

class OrderController extends Controller
{
    public function store(Request $request)
    {
        // No validation via Form Request
        // No service layer — business logic in controller
        // No API Resource — exposing model directly
        $order = Order::create($request->all());
        return response()->json($order);
    }

    public function index()
    {
        // Returning raw models exposes all columns including sensitive data
        return Order::all();
    }
}
```

**Correct (standard Laravel structure with all layers):**

```php
// Standard structure with proper layers — works great for most projects
// app/
//   Http/
//     Controllers/
//       UserController.php
//       OrderController.php
//       ProductController.php
//       InvoiceController.php
//     Requests/
//       StoreUserRequest.php
//       UpdateUserRequest.php
//       StoreOrderRequest.php
//       StoreProductRequest.php
//     Resources/
//       UserResource.php
//       OrderResource.php
//       OrderCollection.php
//       ProductResource.php
//   Models/
//     User.php
//     Order.php
//     Product.php
//     Invoice.php
//   Services/
//     UserService.php
//     OrderService.php
//     ProductService.php
//     InvoiceService.php
//   Repositories/
//     UserRepository.php
//     OrderRepository.php
//     ProductRepository.php

// app/Http/Controllers/OrderController.php
namespace App\Http\Controllers;

use App\Http\Requests\StoreOrderRequest;
use App\Http\Resources\OrderResource;
use App\Services\OrderService;
use Illuminate\Http\JsonResponse;

class OrderController extends Controller
{
    public function __construct(
        private readonly OrderService $orderService,
    ) {}

    public function store(StoreOrderRequest $request): JsonResponse
    {
        $order = $this->orderService->placeOrder(
            userId: $request->user()->id,
            items: $request->validated('items'),
        );

        return OrderResource::make($order)
            ->response()
            ->setStatusCode(201);
    }

    public function index(): OrderCollection
    {
        $orders = $this->orderService->listForUser(auth()->id());

        return new OrderCollection($orders);
    }
}
```

**For larger projects, consider domain-driven structure:**

```php
// Domain-driven structure — each feature is self-contained
// app/
//   Domains/
//     User/
//       Models/User.php
//       Services/UserService.php
//       Controllers/UserController.php
//       Requests/StoreUserRequest.php
//       Resources/UserResource.php
//       Repositories/UserRepositoryInterface.php
//       Events/UserRegistered.php
//       Policies/UserPolicy.php
//       Routes/api.php
//       UserServiceProvider.php
//     Order/
//       Models/Order.php, OrderItem.php
//       Services/OrderService.php
//       Controllers/OrderController.php
//       Requests/StoreOrderRequest.php
//       Resources/OrderResource.php
//       Repositories/OrderRepositoryInterface.php
//       Events/OrderPlaced.php
//       Policies/OrderPolicy.php
//       Routes/api.php
//       OrderServiceProvider.php

// app/Domains/Order/OrderServiceProvider.php
namespace App\Domains\Order;

use App\Domains\Order\Repositories\EloquentOrderRepository;
use App\Domains\Order\Repositories\OrderRepositoryInterface;
use Illuminate\Support\Facades\Route;
use Illuminate\Support\ServiceProvider;

class OrderServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind(OrderRepositoryInterface::class, EloquentOrderRepository::class);
    }

    public function boot(): void
    {
        Route::middleware('api')
            ->prefix('api/orders')
            ->group(__DIR__ . '/Routes/api.php');
    }
}

// Register domain providers in config/app.php or bootstrap/providers.php
// App\Domains\User\UserServiceProvider::class,
// App\Domains\Order\OrderServiceProvider::class,
```

Reference: [Laravel Application Structure](https://laravel.com/docs/structure)

---

### 1.3 Single Responsibility for Services

**Impact: HIGH** — "God classes are the #1 maintainability killer"

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

---

### 1.4 Use Event-Driven Architecture for Decoupling

**Impact: MEDIUM-HIGH** — Reduces module coupling significantly

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

---

### 1.5 Use Repository Pattern for Data Access

**Impact: HIGH** — Enables testability and database abstraction

Abstract Eloquent queries behind repository interfaces instead of scattering query logic across controllers, services, and jobs. This makes your code testable (you can mock the repository in unit tests), keeps query logic in one place, and allows you to swap the data source without touching business logic.

**Incorrect**

```php
// app/Http/Controllers/UserController.php
namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class UserController extends Controller
{
    public function index(Request $request): JsonResponse
    {
        // Eloquent queries scattered directly in the controller
        $users = User::query()
            ->when($request->search, function ($query, $search) {
                $query->where('name', 'like', "%{$search}%")
                      ->orWhere('email', 'like', "%{$search}%");
            })
            ->when($request->role, function ($query, $role) {
                $query->whereHas('roles', fn ($q) => $q->where('slug', $role));
            })
            ->when($request->status === 'active', function ($query) {
                $query->whereNotNull('email_verified_at')
                      ->where('is_active', true);
            })
            ->with(['roles', 'department'])
            ->orderBy($request->sort_by ?? 'created_at', $request->sort_dir ?? 'desc')
            ->paginate($request->per_page ?? 15);

        return response()->json($users);
    }

    public function store(Request $request): JsonResponse
    {
        // Direct model creation in the controller
        $user = User::create($request->validated());
        $user->roles()->sync($request->input('role_ids', []));

        return response()->json($user->load('roles'), 201);
    }

    public function findActiveByDepartment(int $departmentId): JsonResponse
    {
        // Same query patterns duplicated across multiple controllers/services
        $users = User::where('department_id', $departmentId)
            ->whereNotNull('email_verified_at')
            ->where('is_active', true)
            ->with('roles')
            ->get();

        return response()->json($users);
    }
}

// Problems:
// - Query logic duplicated across controllers, services, jobs
// - Cannot unit-test business logic without hitting the database
// - Changing the "active user" definition requires updating dozens of files
// - Controller is doing data-access work instead of HTTP concerns
```

**Correct**

```php
// Step 1: Define the interface

// app/Domains/User/Repositories/UserRepositoryInterface.php
namespace App\Domains\User\Repositories;

use App\Domains\User\Models\User;
use Illuminate\Contracts\Pagination\LengthAwarePaginator;
use Illuminate\Database\Eloquent\Collection;

interface UserRepositoryInterface
{
    public function findOrFail(int $id): User;

    public function search(array $filters, int $perPage = 15): LengthAwarePaginator;

    public function findActiveByDepartment(int $departmentId): Collection;

    public function create(array $attributes): User;

    public function update(User $user, array $attributes): User;

    public function syncRoles(User $user, array $roleIds): void;
}

// Step 2: Implement with Eloquent

// app/Domains/User/Repositories/EloquentUserRepository.php
namespace App\Domains\User\Repositories;

use App\Domains\User\Models\User;
use Illuminate\Contracts\Pagination\LengthAwarePaginator;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Collection;

class EloquentUserRepository implements UserRepositoryInterface
{
    public function findOrFail(int $id): User
    {
        return User::with(['roles', 'department'])->findOrFail($id);
    }

    public function search(array $filters, int $perPage = 15): LengthAwarePaginator
    {
        return User::query()
            ->when($filters['search'] ?? null, function (Builder $query, string $search) {
                $query->where(function (Builder $q) use ($search) {
                    $q->where('name', 'like', "%{$search}%")
                      ->orWhere('email', 'like', "%{$search}%");
                });
            })
            ->when($filters['role'] ?? null, function (Builder $query, string $role) {
                $query->whereHas('roles', fn (Builder $q) => $q->where('slug', $role));
            })
            ->when(($filters['status'] ?? null) === 'active', function (Builder $query) {
                $this->scopeActive($query);
            })
            ->with(['roles', 'department'])
            ->orderBy(
                $filters['sort_by'] ?? 'created_at',
                $filters['sort_dir'] ?? 'desc',
            )
            ->paginate($perPage);
    }

    public function findActiveByDepartment(int $departmentId): Collection
    {
        return User::where('department_id', $departmentId)
            ->where(fn (Builder $query) => $this->scopeActive($query))
            ->with('roles')
            ->get();
    }

    public function create(array $attributes): User
    {
        return User::create($attributes);
    }

    public function update(User $user, array $attributes): User
    {
        $user->update($attributes);

        return $user->refresh();
    }

    public function syncRoles(User $user, array $roleIds): void
    {
        $user->roles()->sync($roleIds);
    }

    /**
     * Reusable "active user" scope -- defined once, used everywhere.
     */
    private function scopeActive(Builder $query): Builder
    {
        return $query->whereNotNull('email_verified_at')
                     ->where('is_active', true);
    }
}

// Step 3: Bind in the Service Provider

// app/Domains/User/UserServiceProvider.php
namespace App\Domains\User;

use App\Domains\User\Repositories\EloquentUserRepository;
use App\Domains\User\Repositories\UserRepositoryInterface;
use Illuminate\Support\ServiceProvider;

class UserServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind(UserRepositoryInterface::class, EloquentUserRepository::class);
    }
}

// Step 4: Use the interface in services and controllers

// app/Domains/User/Services/UserService.php
namespace App\Domains\User\Services;

use App\Domains\User\Models\User;
use App\Domains\User\Repositories\UserRepositoryInterface;
use App\Domains\User\Events\UserRegistered;

class UserService
{
    public function __construct(
        private readonly UserRepositoryInterface $users,
    ) {}

    public function register(array $data, array $roleIds = []): User
    {
        $user = $this->users->create($data);

        if ($roleIds) {
            $this->users->syncRoles($user, $roleIds);
        }

        UserRegistered::dispatch($user);

        return $user;
    }
}

// app/Domains/User/Controllers/UserController.php
namespace App\Domains\User\Controllers;

use App\Domains\User\Repositories\UserRepositoryInterface;
use App\Domains\User\Requests\IndexUsersRequest;
use Illuminate\Http\JsonResponse;
use Illuminate\Routing\Controller;

class UserController extends Controller
{
    public function __construct(
        private readonly UserRepositoryInterface $users,
    ) {}

    public function index(IndexUsersRequest $request): JsonResponse
    {
        $result = $this->users->search(
            filters: $request->validated(),
            perPage: $request->integer('per_page', 15),
        );

        return response()->json($result);
    }
}

// In tests, mock the repository easily:
//
// $mock = $this->mock(UserRepositoryInterface::class);
// $mock->shouldReceive('search')
//      ->with(['search' => 'jane'], 15)
//      ->andReturn(new LengthAwarePaginator([$fakeUser], 1, 15));
```

Reference: [Laravel Service Container](https://laravel.com/docs/container)

---

## 2. Dependency Injection

**Section Impact: CRITICAL**

### 2.1 Avoid Service Locator Anti-Pattern

**Impact: CRITICAL** — Hides dependencies and breaks testability

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

---

### 2.2 Apply Interface Segregation Principle

**Impact: HIGH** — Prevents bloated interfaces that force unnecessary implementations

No client should be forced to depend on methods it does not use. When an interface grows too large, implementing classes are burdened with methods they do not need, leading to empty stubs, thrown exceptions for unsupported operations, or tightly coupled code. Split large interfaces into smaller, focused ones so that each consumer depends only on the behavior it actually requires.

**Incorrect**

```php
<?php

namespace App\Contracts;

use App\Models\User;
use Illuminate\Support\Collection;

// Fat interface - forces every implementation to handle all concerns
interface UserRepositoryInterface
{
    public function find(int $id): ?User;
    public function findByEmail(string $email): ?User;
    public function findByUsername(string $username): ?User;
    public function all(): Collection;
    public function paginate(int $perPage = 15): LengthAwarePaginator;
    public function create(array $attributes): User;
    public function update(int $id, array $attributes): User;
    public function delete(int $id): bool;
    public function forceDelete(int $id): bool;
    public function restore(int $id): bool;
    public function search(string $query): Collection;
    public function searchByFilters(array $filters): Collection;
    public function reindex(): void;
    public function export(string $format): string;
    public function import(string $data, string $format): int;
}

// A read-only API consumer is forced to implement write methods it never uses
class ApiUserRepository implements UserRepositoryInterface
{
    public function find(int $id): ?User { /* ... */ }
    public function findByEmail(string $email): ?User { /* ... */ }
    public function findByUsername(string $username): ?User { /* ... */ }
    public function all(): Collection { /* ... */ }
    public function paginate(int $perPage = 15): LengthAwarePaginator { /* ... */ }

    // Forced to stub out methods that make no sense for this implementation
    public function create(array $attributes): User
    {
        throw new \BadMethodCallException('Not supported');
    }

    public function update(int $id, array $attributes): User
    {
        throw new \BadMethodCallException('Not supported');
    }

    public function delete(int $id): bool
    {
        throw new \BadMethodCallException('Not supported');
    }

    public function forceDelete(int $id): bool
    {
        throw new \BadMethodCallException('Not supported');
    }

    public function restore(int $id): bool
    {
        throw new \BadMethodCallException('Not supported');
    }

    public function search(string $query): Collection
    {
        throw new \BadMethodCallException('Not supported');
    }

    public function searchByFilters(array $filters): Collection
    {
        throw new \BadMethodCallException('Not supported');
    }

    public function reindex(): void
    {
        throw new \BadMethodCallException('Not supported');
    }

    public function export(string $format): string
    {
        throw new \BadMethodCallException('Not supported');
    }

    public function import(string $data, string $format): int
    {
        throw new \BadMethodCallException('Not supported');
    }
}
```

**Correct**

```php
<?php

namespace App\Contracts;

use App\Models\User;
use Illuminate\Support\Collection;
use Illuminate\Pagination\LengthAwarePaginator;

// Focused interface for reading users
interface UserReadableInterface
{
    public function find(int $id): ?User;
    public function findByEmail(string $email): ?User;
    public function findByUsername(string $username): ?User;
    public function all(): Collection;
    public function paginate(int $perPage = 15): LengthAwarePaginator;
}

// Focused interface for writing users
interface UserWritableInterface
{
    public function create(array $attributes): User;
    public function update(int $id, array $attributes): User;
    public function delete(int $id): bool;
    public function forceDelete(int $id): bool;
    public function restore(int $id): bool;
}

// Focused interface for searching users
interface UserSearchableInterface
{
    public function search(string $query): Collection;
    public function searchByFilters(array $filters): Collection;
    public function reindex(): void;
}

// Focused interface for import/export
interface UserTransferableInterface
{
    public function export(string $format): string;
    public function import(string $data, string $format): int;
}

// Full Eloquent implementation composes all interfaces it supports
class EloquentUserRepository implements
    UserReadableInterface,
    UserWritableInterface,
    UserSearchableInterface,
    UserTransferableInterface
{
    public function find(int $id): ?User
    {
        return User::find($id);
    }

    public function findByEmail(string $email): ?User
    {
        return User::where('email', $email)->first();
    }

    public function findByUsername(string $username): ?User
    {
        return User::where('username', $username)->first();
    }

    public function all(): Collection
    {
        return User::all();
    }

    public function paginate(int $perPage = 15): LengthAwarePaginator
    {
        return User::paginate($perPage);
    }

    public function create(array $attributes): User
    {
        return User::create($attributes);
    }

    public function update(int $id, array $attributes): User
    {
        $user = User::findOrFail($id);
        $user->update($attributes);
        return $user->fresh();
    }

    public function delete(int $id): bool
    {
        return User::findOrFail($id)->delete();
    }

    public function forceDelete(int $id): bool
    {
        return User::withTrashed()->findOrFail($id)->forceDelete();
    }

    public function restore(int $id): bool
    {
        return User::withTrashed()->findOrFail($id)->restore();
    }

    public function search(string $query): Collection
    {
        return User::where('name', 'like', "%{$query}%")
            ->orWhere('email', 'like', "%{$query}%")
            ->get();
    }

    public function searchByFilters(array $filters): Collection
    {
        $query = User::query();
        foreach ($filters as $field => $value) {
            $query->where($field, $value);
        }
        return $query->get();
    }

    public function reindex(): void
    {
        User::all()->searchable();
    }

    public function export(string $format): string { /* ... */ }
    public function import(string $data, string $format): int { /* ... */ }
}

// Read-only consumer only implements what it needs - no stubs
class ApiUserRepository implements UserReadableInterface
{
    public function __construct(
        private readonly HttpClient $client,
    ) {}

    public function find(int $id): ?User { /* ... */ }
    public function findByEmail(string $email): ?User { /* ... */ }
    public function findByUsername(string $username): ?User { /* ... */ }
    public function all(): Collection { /* ... */ }
    public function paginate(int $perPage = 15): LengthAwarePaginator { /* ... */ }
}

// Controllers type-hint only the interface they need
class UserProfileController extends Controller
{
    public function __construct(
        private readonly UserReadableInterface $users,
    ) {}

    public function show(int $id): JsonResponse
    {
        $user = $this->users->find($id);
        return response()->json($user);
    }
}

class UserAdminController extends Controller
{
    public function __construct(
        private readonly UserReadableInterface $reader,
        private readonly UserWritableInterface $writer,
    ) {}

    public function update(Request $request, int $id): JsonResponse
    {
        $user = $this->writer->update($id, $request->validated());
        return response()->json($user);
    }
}
```

Reference: [Laravel Contracts](https://laravel.com/docs/contracts)

---

### 2.3 Honor Liskov Substitution Principle

**Impact: HIGH** — Ensures implementations are truly interchangeable

Any implementation of an interface or subclass of a base class must be fully substitutable for it without altering the correctness of the program. If swapping one implementation for another causes exceptions, changed return semantics, or violated preconditions, the Liskov Substitution Principle (LSP) is broken. This makes polymorphism unreliable and defeats the purpose of coding against abstractions.

**Incorrect**

```php
<?php

namespace App\Contracts;

use App\Models\User;
use Illuminate\Support\Collection;

interface UserRepositoryInterface
{
    public function find(int $id): ?User;
    public function all(): Collection;
    public function create(array $attributes): User;
    public function delete(int $id): bool;
}

namespace App\Repositories;

use App\Contracts\UserRepositoryInterface;
use App\Models\User;
use Illuminate\Support\Collection;

class EloquentUserRepository implements UserRepositoryInterface
{
    public function find(int $id): ?User
    {
        return User::find($id);
    }

    public function all(): Collection
    {
        return User::all();
    }

    public function create(array $attributes): User
    {
        return User::create($attributes);
    }

    public function delete(int $id): bool
    {
        return User::findOrFail($id)->delete();
    }
}

// Violates LSP - changes the behavior contract of the interface
class CacheUserRepository extends EloquentUserRepository
{
    public function __construct(
        private readonly CacheManager $cache,
    ) {}

    public function find(int $id): ?User
    {
        // Strengthened postcondition: throws instead of returning null
        // The interface says ?User (nullable), but this never returns null
        return $this->cache->remember("user.{$id}", 3600, function () use ($id) {
            $user = parent::find($id);
            if ($user === null) {
                // Violates contract - callers expect null, not an exception
                throw new \RuntimeException("User {$id} not found in cache layer");
            }
            return $user;
        });
    }

    public function create(array $attributes): User
    {
        // Weakened postcondition: silently skips cache invalidation
        // Callers relying on consistent state will get stale data
        return parent::create($attributes);
    }

    public function delete(int $id): bool
    {
        // Violates contract - refuses to perform the operation
        throw new \BadMethodCallException(
            'CacheUserRepository does not support deletion. Use EloquentUserRepository.'
        );
    }

    public function all(): Collection
    {
        // Changed semantics: returns only cached users, not all users
        // Subtly different behavior that breaks callers expecting complete data
        $cachedKeys = $this->cache->get('user.all.keys', []);
        return collect($cachedKeys)
            ->map(fn (int $id) => $this->cache->get("user.{$id}"))
            ->filter();
    }
}
```

**Correct**

```php
<?php

namespace App\Contracts;

use App\Models\User;
use Illuminate\Support\Collection;

interface UserRepositoryInterface
{
    /**
     * Find a user by ID.
     *
     * @return User|null Returns null when user does not exist.
     */
    public function find(int $id): ?User;

    /**
     * Retrieve all users.
     *
     * @return Collection Complete collection of all users.
     */
    public function all(): Collection;

    /**
     * Create a new user.
     *
     * @return User The persisted user instance.
     */
    public function create(array $attributes): User;

    /**
     * Delete a user by ID.
     *
     * @return bool True if deletion succeeded.
     * @throws \Illuminate\Database\Eloquent\ModelNotFoundException
     */
    public function delete(int $id): bool;
}

namespace App\Repositories;

use App\Contracts\UserRepositoryInterface;
use App\Models\User;
use Illuminate\Support\Collection;

class EloquentUserRepository implements UserRepositoryInterface
{
    public function find(int $id): ?User
    {
        return User::find($id);
    }

    public function all(): Collection
    {
        return User::all();
    }

    public function create(array $attributes): User
    {
        return User::create($attributes);
    }

    public function delete(int $id): bool
    {
        return User::findOrFail($id)->delete();
    }
}

// Honors LSP - decorates without changing behavioral contract
class CachingUserRepository implements UserRepositoryInterface
{
    public function __construct(
        private readonly UserRepositoryInterface $inner,
        private readonly CacheManager $cache,
        private readonly int $ttl = 3600,
    ) {}

    public function find(int $id): ?User
    {
        // Preserves nullable return - null is cached as a valid result
        return $this->cache->remember(
            "user.{$id}",
            $this->ttl,
            fn () => $this->inner->find($id)
        );
    }

    public function all(): Collection
    {
        // Returns the complete collection, just from cache when available
        return $this->cache->remember(
            'users.all',
            $this->ttl,
            fn () => $this->inner->all()
        );
    }

    public function create(array $attributes): User
    {
        $user = $this->inner->create($attributes);

        // Invalidate relevant caches to maintain consistency
        $this->cache->forget('users.all');
        $this->cache->put("user.{$user->id}", $user, $this->ttl);

        return $user;
    }

    public function delete(int $id): bool
    {
        // Honors the full contract - delegates then cleans up cache
        $result = $this->inner->delete($id);

        $this->cache->forget("user.{$id}");
        $this->cache->forget('users.all');

        return $result;
    }
}

// Service provider wires the decorator chain
namespace App\Providers;

use App\Contracts\UserRepositoryInterface;
use App\Repositories\CachingUserRepository;
use App\Repositories\EloquentUserRepository;
use Illuminate\Support\ServiceProvider;

class RepositoryServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind(UserRepositoryInterface::class, function ($app) {
            return new CachingUserRepository(
                inner: new EloquentUserRepository(),
                cache: $app->make('cache.store'),
            );
        });
    }
}

// Any consumer can rely on consistent behavior regardless of implementation
class UserController extends Controller
{
    public function __construct(
        private readonly UserRepositoryInterface $users,
    ) {}

    public function show(int $id): JsonResponse
    {
        // Works identically with EloquentUserRepository or CachingUserRepository
        $user = $this->users->find($id);

        if ($user === null) {
            abort(404, 'User not found');
        }

        return response()->json($user);
    }

    public function destroy(int $id): JsonResponse
    {
        // No need to check which implementation is behind the interface
        $this->users->delete($id);
        return response()->json(['message' => 'User deleted']);
    }
}
```

Reference: [Laravel Service Container](https://laravel.com/docs/container)

---

### 2.4 Prefer Constructor Injection

**Impact: HIGH** — Makes dependencies explicit and testable

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

---

### 2.5 Understand Provider Scopes

**Impact: MEDIUM-HIGH** — Wrong scope causes subtle bugs in long-running processes

Laravel's service container offers three binding scopes: `bind()` creates a new instance every time the service is resolved, `singleton()` creates one shared instance for the entire application lifecycle, and `scoped()` creates one instance per request lifecycle (reset between requests). Choosing the wrong scope leads to subtle bugs that are especially dangerous in long-running processes like Laravel Octane, queue workers, and Reverb. A singleton that caches request-specific data will leak state between requests under Octane. A bind where a singleton is needed may cause unnecessary object creation and inconsistent state.

**Incorrect**

```php
<?php

namespace App\Providers;

use App\Services\AuthContext;
use App\Services\ReportGenerator;
use App\Services\ShoppingCart;
use App\Services\DatabaseConnectionPool;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // WRONG: Singleton for request-specific data
        // Under Octane, the authenticated user from request #1
        // will leak into request #2
        $this->app->singleton(AuthContext::class, function ($app) {
            return new AuthContext(
                user: $app->make('auth')->user(),  // Captured once, stale forever
                ipAddress: $app->make('request')->ip(),
            );
        });

        // WRONG: Singleton for stateful per-request object
        // Shopping cart items from one user bleed into the next request
        $this->app->singleton(ShoppingCart::class, function ($app) {
            return new ShoppingCart();
        });

        // WRONG: bind() for expensive shared resource
        // Creates a new connection pool on every resolution - wasteful and dangerous
        $this->app->bind(DatabaseConnectionPool::class, function ($app) {
            return new DatabaseConnectionPool(
                host: config('database.connections.mysql.host'),
                maxConnections: 10,
            );
        });

        // WRONG: bind() when consistency within a request matters
        // Different parts of the same request get different instances
        // with potentially different state
        $this->app->bind(ReportGenerator::class, function ($app) {
            return new ReportGenerator(
                startedAt: now(),  // Different timestamp each resolution
                requestId: Str::uuid(),  // Different ID each resolution
            );
        });
    }
}
```

**Correct**

```php
<?php

namespace App\Providers;

use App\Contracts\ConnectionPoolInterface;
use App\Services\AuthContext;
use App\Services\DatabaseConnectionPool;
use App\Services\ReportGenerator;
use App\Services\ShoppingCart;
use App\Services\TemporaryFileManager;
use Illuminate\Support\ServiceProvider;
use Illuminate\Support\Str;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // scoped() - Reset between requests, safe for Octane
        // Perfect for request-specific data that should be consistent
        // within a single request but fresh for each new request
        $this->app->scoped(AuthContext::class, function ($app) {
            return new AuthContext(
                user: $app->make('auth')->user(),
                ipAddress: $app->make('request')->ip(),
            );
        });

        // scoped() - Per-request stateful object
        // Each request gets its own cart, automatically cleared between requests
        $this->app->scoped(ShoppingCart::class, function ($app) {
            return new ShoppingCart();
        });

        // singleton() - Shared expensive resource that is stateless
        // Connection pool is thread-safe and should be reused across requests
        $this->app->singleton(ConnectionPoolInterface::class, function ($app) {
            return new DatabaseConnectionPool(
                host: config('database.connections.mysql.host'),
                maxConnections: 10,
            );
        });

        // scoped() - Consistent identity within a request
        // Same instance returned throughout the request lifecycle,
        // so requestId and startedAt remain consistent
        $this->app->scoped(ReportGenerator::class, function ($app) {
            return new ReportGenerator(
                startedAt: now(),
                requestId: Str::uuid()->toString(),
            );
        });

        // bind() - Truly stateless, independent instances needed each time
        // Each resolution should be a fresh, isolated object
        $this->app->bind(TemporaryFileManager::class, function ($app) {
            return new TemporaryFileManager(
                directory: storage_path('app/temp/' . Str::random(16)),
            );
        });
    }
}

// Quick reference for choosing the right scope:
//
// singleton() - Use when:
//   - The object is expensive to create (connection pools, HTTP clients)
//   - The object holds NO request-specific state
//   - The object is thread-safe / stateless between requests
//   - Examples: config readers, HTTP client pools, third-party SDK clients
//
// scoped() - Use when:
//   - The object holds request-specific state (auth, cart, request ID)
//   - You need the same instance within a request but a fresh one per request
//   - Running under Octane, Reverb, or any long-lived process
//   - Examples: auth context, shopping cart, request loggers, tenant context
//
// bind() - Use when:
//   - Every resolution should produce an independent instance
//   - The object is lightweight and short-lived
//   - No shared state is needed between resolutions
//   - Examples: DTOs, value objects, temp file handlers, one-off builders

// Octane-specific considerations:
// In config/octane.php, list singletons that need flushing between requests:
//
// 'flush' => [
//     AuthContext::class,
//     ShoppingCart::class,
// ],
//
// Or better yet, use scoped() so Laravel handles it automatically.
```

Reference: [Laravel Service Container](https://laravel.com/docs/container)

---

### 2.6 Use Injection Tokens for Interfaces

**Impact: HIGH** — Essential for swappable implementations

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

---

## 3. Error Handling

**Section Impact: HIGH**

### 3.1 Handle Queue and Job Errors Properly

**Impact: HIGH** — Unhandled job failures cause silent data loss

Queued jobs run outside the HTTP request lifecycle, so unhandled exceptions disappear silently unless you explicitly configure retry logic and failure handling. Without the `failed()` method, retry properties, and proper exception handling, a job that throws an exception will retry indefinitely with no backoff (wasting resources) or fail permanently with no notification. The `failed_jobs` table only captures the final failure -- if you never inspect it or set up alerts, data loss goes unnoticed for days.

Always define `$tries`, `$backoff`, and `$maxExceptions` on your jobs, implement the `failed()` method for cleanup and alerting, and catch specific exceptions to decide whether to retry or permanently fail.

**Incorrect**

```php
// app/Jobs/ProcessPayment.php
namespace App\Jobs;

use App\Models\Order;
use App\Services\PaymentGateway;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class ProcessPayment implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(
        public readonly Order $order,
    ) {}

    public function handle(PaymentGateway $gateway): void
    {
        // No try/catch -- any exception causes silent retry with no backoff.
        // No $tries property -- job retries indefinitely, hammering the payment API.
        // No failed() method -- when the job finally dies, nobody is notified.
        // No distinction between transient and permanent errors.
        $gateway->charge(
            $this->order->user->payment_method_id,
            $this->order->total_cents,
        );

        $this->order->update(['status' => 'paid']);
    }
}

// Dispatching with no awareness of failure handling
// app/Http/Controllers/CheckoutController.php
class CheckoutController extends Controller
{
    public function store(Request $request): JsonResponse
    {
        $order = Order::create($request->validated());

        // Fire and forget -- caller has no way to know if this ever succeeds
        ProcessPayment::dispatch($order);

        return response()->json(['message' => 'Order placed']);
    }
}
```

**Correct**

```php
// app/Jobs/ProcessPayment.php
namespace App\Jobs;

use App\Events\PaymentFailed;
use App\Exceptions\PaymentDeclinedException;
use App\Models\Order;
use App\Services\PaymentGateway;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\MaxAttemptsExceededException;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;
use Throwable;

class ProcessPayment implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    /**
     * Maximum number of attempts before the job is sent to failed_jobs.
     */
    public int $tries = 5;

    /**
     * Exponential backoff intervals (in seconds) between retries.
     */
    public array $backoff = [10, 30, 60, 120, 300];

    /**
     * Stop retrying after this many unhandled exceptions (prevents
     * burning through all $tries on a permanently broken payload).
     */
    public int $maxExceptions = 3;

    /**
     * Timeout in seconds for a single attempt.
     */
    public int $timeout = 30;

    public function __construct(
        public readonly Order $order,
    ) {}

    public function handle(PaymentGateway $gateway): void
    {
        try {
            $result = $gateway->charge(
                $this->order->user->payment_method_id,
                $this->order->total_cents,
            );

            $this->order->update([
                'status' => 'paid',
                'transaction_id' => $result->transactionId,
                'paid_at' => now(),
            ]);

            Log::info('Payment processed successfully', [
                'order_id' => $this->order->id,
                'transaction_id' => $result->transactionId,
            ]);
        } catch (PaymentDeclinedException $e) {
            // Permanent failure -- card declined, do NOT retry
            Log::warning('Payment declined permanently', [
                'order_id' => $this->order->id,
                'reason' => $e->declineReason(),
            ]);

            $this->order->update(['status' => 'payment_declined']);
            $this->fail($e); // Marks as failed immediately, skips remaining retries
        }
        // Transient exceptions (network timeouts, 5xx from gateway) are NOT caught
        // here, so they bubble up and trigger automatic retry with backoff.
    }

    /**
     * Called when the job has exhausted all retries or was manually failed.
     * Use this for cleanup, alerting, and compensating actions.
     */
    public function failed(?Throwable $exception): void
    {
        Log::error('Payment job failed permanently', [
            'order_id' => $this->order->id,
            'exception' => $exception?->getMessage(),
            'attempts' => $this->attempts(),
        ]);

        $this->order->update(['status' => 'payment_failed']);

        // Notify stakeholders so the failure does not go unnoticed
        PaymentFailed::dispatch($this->order, $exception);
    }

    /**
     * Determine the middleware the job should pass through.
     * Rate-limit to avoid hammering the payment gateway.
     */
    public function middleware(): array
    {
        return [
            new \Illuminate\Queue\Middleware\RateLimited('payments'),
        ];
    }
}

// config/queue.php -- ensure failed_jobs table is configured
// Run: php artisan queue:failed-table && php artisan migrate
//
// 'failed' => [
//     'driver' => env('QUEUE_FAILED_DRIVER', 'database-uuids'),
//     'database' => env('DB_CONNECTION', 'mysql'),
//     'table' => 'failed_jobs',
// ],

// app/Providers/AppServiceProvider.php -- global failure monitoring
namespace App\Providers;

use Illuminate\Queue\Events\JobFailed;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Queue;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Global listener that fires for ANY failed job across the application
        Queue::failing(function (JobFailed $event) {
            Log::channel('slack')->critical('Queue job failed', [
                'connection' => $event->connectionName,
                'queue' => $event->job->getQueue(),
                'job' => $event->job->resolveName(),
                'exception' => $event->exception->getMessage(),
            ]);
        });
    }
}
```

Reference: [Laravel Queues](https://laravel.com/docs/queues)

---

### 3.2 Throw HTTP Exceptions from Services

**Impact: HIGH** — Consistent error responses improve API reliability

Services that return error arrays like `['error' => 'Not found']` force every caller to inspect the return value and translate it into an HTTP response manually. This leads to inconsistent status codes, duplicated error-formatting logic, and forgotten checks that leak raw error data to clients.

Instead, throw typed exceptions -- `ModelNotFoundException`, `AuthorizationException`, `ValidationException`, or `abort()` -- and let Laravel's exception handler convert them into proper HTTP responses automatically. For domain-specific errors, create custom exception classes with a `render()` method so the exception itself knows how to present as an API response.

**Incorrect**

```php
// app/Services/InvoiceService.php
namespace App\Services;

use App\Models\Invoice;
use App\Models\User;

class InvoiceService
{
    public function getInvoice(int $invoiceId, User $user): array
    {
        $invoice = Invoice::find($invoiceId);

        // Returning error arrays forces every caller to check and handle these
        if (! $invoice) {
            return ['error' => 'Not found', 'code' => 404];
        }

        if ($invoice->user_id !== $user->id) {
            return ['error' => 'Forbidden', 'code' => 403];
        }

        if ($invoice->is_draft) {
            return ['error' => 'Invoice is still in draft', 'code' => 422];
        }

        return ['data' => $invoice->toArray()];
    }
}

// app/Http/Controllers/InvoiceController.php
namespace App\Http\Controllers;

use App\Services\InvoiceService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class InvoiceController extends Controller
{
    public function show(int $id, Request $request, InvoiceService $service): JsonResponse
    {
        $result = $service->getInvoice($id, $request->user());

        // Every controller that calls the service must duplicate this check
        if (isset($result['error'])) {
            return response()->json(
                ['message' => $result['error']],
                $result['code'],
            );
        }

        return response()->json($result['data']);
    }
}
```

**Correct**

```php
// app/Services/InvoiceService.php
namespace App\Services;

use App\Exceptions\InvoiceDraftException;
use App\Models\Invoice;
use App\Models\User;
use Illuminate\Auth\Access\AuthorizationException;
use Illuminate\Database\Eloquent\ModelNotFoundException;

class InvoiceService
{
    /**
     * Always returns a valid Invoice or throws.
     * Callers never need to inspect error arrays.
     */
    public function getInvoice(int $invoiceId, User $user): Invoice
    {
        // findOrFail throws ModelNotFoundException -> 404 automatically
        $invoice = Invoice::findOrFail($invoiceId);

        if ($invoice->user_id !== $user->id) {
            throw new AuthorizationException(
                'You do not have access to this invoice.'
            );
        }

        if ($invoice->is_draft) {
            throw new InvoiceDraftException($invoice);
        }

        return $invoice;
    }
}

// app/Exceptions/InvoiceDraftException.php
namespace App\Exceptions;

use App\Models\Invoice;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Symfony\Component\HttpKernel\Exception\HttpException;

class InvoiceDraftException extends HttpException
{
    public function __construct(
        public readonly Invoice $invoice,
    ) {
        parent::__construct(
            statusCode: 422,
            message: "Invoice #{$invoice->id} is still in draft and cannot be viewed.",
        );
    }

    /**
     * Render the exception as an HTTP response.
     * Laravel calls this automatically when the exception is thrown during a request.
     */
    public function render(Request $request): JsonResponse
    {
        return response()->json([
            'message' => $this->getMessage(),
            'invoice_id' => $this->invoice->id,
            'status' => 'draft',
        ], $this->getStatusCode());
    }
}

// app/Http/Controllers/InvoiceController.php
namespace App\Http\Controllers;

use App\Http\Resources\InvoiceResource;
use App\Services\InvoiceService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class InvoiceController extends Controller
{
    /**
     * No error handling needed -- exceptions propagate to the Handler,
     * which returns the correct HTTP status and JSON structure.
     */
    public function show(int $id, Request $request, InvoiceService $service): JsonResponse
    {
        $invoice = $service->getInvoice($id, $request->user());

        return InvoiceResource::make($invoice)->response();
    }
}

// Using abort() for quick guards in simpler scenarios
// app/Http/Controllers/ReportController.php
namespace App\Http\Controllers;

use App\Models\Report;
use Illuminate\Http\Request;

class ReportController extends Controller
{
    public function download(int $id, Request $request)
    {
        $report = Report::findOrFail($id);  // 404 if missing

        abort_unless($report->team_id === $request->user()->team_id, 403);
        abort_unless($report->is_generated, 422, 'Report has not been generated yet.');

        return response()->download($report->file_path);
    }
}
```

Reference: [Laravel Error Handling](https://laravel.com/docs/errors)

---

### 3.3 Use Exception Handler for Error Handling

**Impact: CRITICAL** — Centralized error handling prevents information leaks

Scattering try/catch blocks across every controller method leads to duplicated error-formatting logic, inconsistent response structures, and the inevitable forgotten catch that leaks a stack trace to the client in production. Laravel's Exception Handler (`bootstrap/app.php` in Laravel 11+, or `app/Exceptions/Handler.php` in earlier versions) gives you a single place to map exception types to HTTP responses, configure logging channels, suppress sensitive details, and report to external services.

Move all cross-cutting error handling into the Handler's `register()` method using `reportable()` and `renderable()` callbacks. Controllers should stay clean -- let exceptions propagate naturally.

**Incorrect**

```php
// app/Http/Controllers/OrderController.php
namespace App\Http\Controllers;

use App\Models\Order;
use App\Services\OrderService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Illuminate\Validation\ValidationException;
use Throwable;

class OrderController extends Controller
{
    public function store(Request $request, OrderService $service): JsonResponse
    {
        // Duplicated try/catch in every single controller method
        try {
            $validated = $request->validate([
                'product_id' => 'required|exists:products,id',
                'quantity' => 'required|integer|min:1',
            ]);

            $order = $service->createOrder($request->user(), $validated);

            return response()->json($order, 201);
        } catch (ValidationException $e) {
            return response()->json(['errors' => $e->errors()], 422);
        } catch (Throwable $e) {
            // Logging is done ad-hoc with different formats in every controller
            Log::error('Order creation failed: ' . $e->getMessage());

            // In production this leaks the exception message to the client.
            // Different controllers return different error shapes.
            return response()->json([
                'error' => $e->getMessage(),
                'trace' => $e->getTraceAsString(), // SECURITY: stack trace exposed!
            ], 500);
        }
    }

    public function show(int $id, Request $request): JsonResponse
    {
        // Same pattern repeated -- 20 controllers x 5 methods = 100 try/catch blocks
        try {
            $order = Order::findOrFail($id);

            if ($order->user_id !== $request->user()->id) {
                return response()->json(['error' => 'Forbidden'], 403);
            }

            return response()->json($order);
        } catch (Throwable $e) {
            Log::error($e->getMessage());
            return response()->json(['error' => 'Something went wrong'], 500);
        }
    }
}
```

**Correct**

```php
// bootstrap/app.php (Laravel 11+)
// All exception handling is configured in one place.
use App\Exceptions\ExternalServiceException;
use App\Exceptions\InsufficientInventoryException;
use Illuminate\Auth\Access\AuthorizationException;
use Illuminate\Auth\AuthenticationException;
use Illuminate\Database\Eloquent\ModelNotFoundException;
use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;
use Illuminate\Http\Request;
use Illuminate\Validation\ValidationException;
use Symfony\Component\HttpKernel\Exception\HttpException;
use Throwable;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__ . '/../routes/web.php',
        api: __DIR__ . '/../routes/api.php',
    )
    ->withMiddleware(function (Middleware $middleware) {
        //
    })
    ->withExceptions(function (Exceptions $exceptions) {

        // -------------------------------------------------------
        // Reporting: control what gets logged and where
        // -------------------------------------------------------

        // Don't log exceptions that are normal user errors
        $exceptions->dontReport([
            AuthorizationException::class,
            HttpException::class,
            ModelNotFoundException::class,
            ValidationException::class,
        ]);

        // Report external service failures to Sentry/Bugsnag AND the default log
        $exceptions->reportable(function (ExternalServiceException $e) {
            app('sentry')->captureException($e);

            // Return false to ALSO log via the default channel.
            // Return true (or nothing) to stop propagation.
            return false;
        });

        // Report all unexpected errors to an external monitoring service
        $exceptions->reportable(function (Throwable $e) {
            if (app()->bound('sentry') && $e->getCode() >= 500) {
                app('sentry')->captureException($e);
            }
        });

        // -------------------------------------------------------
        // Rendering: map exception types to JSON responses
        // -------------------------------------------------------

        // Consistent 404 shape for missing models
        $exceptions->renderable(function (ModelNotFoundException $e, Request $request) {
            if ($request->expectsJson()) {
                $modelName = class_basename($e->getModel());

                return response()->json([
                    'message' => "{$modelName} not found.",
                    'type' => 'resource_not_found',
                ], 404);
            }
        });

        // Domain exception with business context
        $exceptions->renderable(function (InsufficientInventoryException $e, Request $request) {
            if ($request->expectsJson()) {
                return response()->json([
                    'message' => $e->getMessage(),
                    'type' => 'insufficient_inventory',
                    'product_id' => $e->productId,
                    'requested' => $e->requested,
                    'available' => $e->available,
                ], 422);
            }
        });

        // Catch-all for unexpected errors -- hide internals in production
        $exceptions->renderable(function (Throwable $e, Request $request) {
            if ($request->expectsJson() && ! $e instanceof HttpException) {
                $status = method_exists($e, 'getStatusCode')
                    ? $e->getStatusCode()
                    : 500;

                $body = [
                    'message' => $status >= 500
                        ? 'An internal error occurred.'
                        : $e->getMessage(),
                    'type' => 'server_error',
                ];

                // Only include debug info in non-production environments
                if (config('app.debug')) {
                    $body['debug'] = [
                        'exception' => get_class($e),
                        'message' => $e->getMessage(),
                        'file' => $e->getFile(),
                        'line' => $e->getLine(),
                    ];
                }

                return response()->json($body, $status);
            }
        });

    })->create();

// app/Exceptions/InsufficientInventoryException.php
namespace App\Exceptions;

use RuntimeException;

class InsufficientInventoryException extends RuntimeException
{
    public function __construct(
        public readonly int $productId,
        public readonly int $requested,
        public readonly int $available,
    ) {
        parent::__construct(
            "Product #{$productId}: requested {$requested}, only {$available} available."
        );
    }
}

// app/Exceptions/ExternalServiceException.php
namespace App\Exceptions;

use RuntimeException;

class ExternalServiceException extends RuntimeException
{
    public function __construct(
        public readonly string $service,
        string $message,
        int $code = 0,
        ?\Throwable $previous = null,
    ) {
        parent::__construct("[{$service}] {$message}", $code, $previous);
    }
}

// app/Http/Controllers/OrderController.php
// Controllers are clean -- no try/catch, no error formatting.
namespace App\Http\Controllers;

use App\Http\Requests\StoreOrderRequest;
use App\Http\Resources\OrderResource;
use App\Models\Order;
use App\Services\OrderService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class OrderController extends Controller
{
    public function store(StoreOrderRequest $request, OrderService $service): JsonResponse
    {
        // Validation is handled by StoreOrderRequest (throws ValidationException)
        // Inventory check throws InsufficientInventoryException
        // All mapped to correct responses by the Handler
        $order = $service->createOrder(
            $request->user(),
            $request->validated(),
        );

        return OrderResource::make($order)
            ->response()
            ->setStatusCode(201);
    }

    public function show(int $id, Request $request): JsonResponse
    {
        // findOrFail -> ModelNotFoundException -> 404 via Handler
        $order = Order::findOrFail($id);

        // authorize() -> AuthorizationException -> 403 via Handler
        $this->authorize('view', $order);

        return OrderResource::make($order)->response();
    }
}
```

Reference: [Laravel Error Handling](https://laravel.com/docs/errors)

---

## 4. Security

**Section Impact: HIGH**

### 4.1 Implement Secure JWT Authentication

**Impact: CRITICAL** — Essential for secure APIs

Never roll your own JWT authentication. Manual token generation with hardcoded secrets, missing expiration, and no revocation mechanism creates severe security vulnerabilities. Laravel provides two battle-tested packages: **Sanctum** for SPA and mobile token authentication, and **Passport** for full OAuth2 server implementations. Both handle token signing, expiration, revocation, and scope management out of the box.

Sanctum is the recommended choice for most applications (SPAs, mobile apps, simple API tokens). Use Passport only when you need full OAuth2 compliance (authorization codes, client credentials, third-party API access).

**Incorrect**

```php
// app/Http/Controllers/AuthController.php
namespace App\Http\Controllers;

use App\Models\User;
use Firebase\JWT\JWT;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;

class AuthController extends Controller
{
    // Hardcoded secret in source code - will leak via version control
    private const JWT_SECRET = 'my-super-secret-key-123';

    public function login(Request $request): JsonResponse
    {
        $user = User::where('email', $request->email)->first();

        if (! $user || ! Hash::check($request->password, $user->password)) {
            return response()->json(['error' => 'Invalid credentials'], 401);
        }

        // No expiration set - token is valid forever
        // No token ID - cannot be revoked individually
        // No audience/issuer claims - vulnerable to token confusion attacks
        $token = JWT::encode(
            ['sub' => $user->id, 'email' => $user->email],
            self::JWT_SECRET,
            'HS256'
        );

        // Token returned but never stored - no way to revoke it
        return response()->json(['token' => $token]);
    }

    public function logout(Request $request): JsonResponse
    {
        // No actual token invalidation - the token remains valid
        return response()->json(['message' => 'Logged out']);
    }
}

// routes/api.php
// No middleware protection, no rate limiting on auth endpoints
Route::post('/login', [AuthController::class, 'login']);
Route::get('/user', function (Request $request) {
    // Manual token parsing in every route - error-prone and inconsistent
    $token = $request->bearerToken();
    $payload = JWT::decode($token, new Key(AuthController::JWT_SECRET, 'HS256'));
    return User::find($payload->sub);
});
```

**Correct**

```php
// Using Laravel Sanctum for SPA/Mobile Token Authentication

// Install: composer require laravel/sanctum
// Publish: php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"

// config/sanctum.php - configure token expiration
return [
    'expiration' => 60 * 24, // Tokens expire after 24 hours (in minutes)
    'token_prefix' => '',
    'middleware' => [
        'authenticate_session' => Laravel\Sanctum\Http\Middleware\AuthenticateSession::class,
        'encrypt_cookies' => Illuminate\Cookie\Middleware\EncryptCookies::class,
        'validate_csrf_token' => Illuminate\Foundation\Http\Middleware\ValidateCsrfToken::class,
    ],
];

// app/Http/Controllers/Auth/TokenAuthController.php
namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use App\Http\Requests\Auth\LoginRequest;
use App\Models\User;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\ValidationException;

class TokenAuthController extends Controller
{
    public function login(LoginRequest $request): JsonResponse
    {
        $user = User::where('email', $request->validated('email'))->first();

        if (! $user || ! Hash::check($request->validated('password'), $user->password)) {
            throw ValidationException::withMessages([
                'email' => ['The provided credentials are incorrect.'],
            ]);
        }

        // Define token abilities (scopes) based on user role
        $abilities = $this->getAbilitiesForUser($user);

        // Create token with specific abilities and optional expiration override
        $token = $user->createToken(
            name: $request->validated('device_name', 'api-token'),
            abilities: $abilities,
            expiresAt: now()->addHours(24),
        );

        return response()->json([
            'token' => $token->plainTextToken,
            'expires_at' => $token->accessToken->expires_at,
            'abilities' => $abilities,
        ]);
    }

    public function logout(Request $request): JsonResponse
    {
        // Revoke the specific token used for this request
        $request->user()->currentAccessToken()->delete();

        return response()->json(['message' => 'Token revoked successfully.']);
    }

    public function logoutAll(Request $request): JsonResponse
    {
        // Revoke all tokens for this user (e.g., password change, security breach)
        $request->user()->tokens()->delete();

        return response()->json(['message' => 'All tokens revoked successfully.']);
    }

    private function getAbilitiesForUser(User $user): array
    {
        return match ($user->role) {
            'admin' => ['*'],
            'editor' => ['posts:read', 'posts:write', 'comments:read', 'comments:write'],
            'viewer' => ['posts:read', 'comments:read'],
            default => ['posts:read'],
        };
    }
}

// routes/api.php
use App\Http\Controllers\Auth\TokenAuthController;
use App\Http\Controllers\PostController;

Route::post('/login', [TokenAuthController::class, 'login'])
    ->middleware('throttle:5,1'); // 5 attempts per minute

Route::middleware('auth:sanctum')->group(function () {
    Route::post('/logout', [TokenAuthController::class, 'logout']);
    Route::post('/logout-all', [TokenAuthController::class, 'logoutAll']);

    // Protected routes with token ability checks
    Route::get('/posts', [PostController::class, 'index'])
        ->middleware('ability:posts:read');
    Route::post('/posts', [PostController::class, 'store'])
        ->middleware('ability:posts:write');
});

// app/Http/Controllers/PostController.php
namespace App\Http\Controllers;

use Illuminate\Http\Request;

class PostController extends Controller
{
    public function index(Request $request): JsonResponse
    {
        // Token abilities are already verified by middleware
        // Access the authenticated user safely
        $user = $request->user();

        // Check for a specific ability programmatically when needed
        if ($user->tokenCan('posts:write')) {
            // Include draft posts for users who can write
            return response()->json($user->posts()->withDrafts()->paginate());
        }

        return response()->json($user->posts()->published()->paginate());
    }
}

// Using Laravel Passport for Full OAuth2 (when OAuth2 compliance is required)

// Install: composer require laravel/passport
// Run: php artisan passport:install

// app/Models/User.php
namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens;
}

// app/Providers/AppServiceProvider.php
namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Laravel\Passport\Passport;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Define OAuth2 scopes
        Passport::tokensCan([
            'read-posts' => 'Read posts',
            'write-posts' => 'Create and update posts',
            'admin' => 'Full administrative access',
        ]);

        Passport::tokensExpireIn(now()->addDays(15));
        Passport::refreshTokensExpireIn(now()->addDays(30));
        Passport::personalAccessTokensExpireIn(now()->addMonths(6));
    }
}

// routes/api.php (Passport)
Route::middleware('auth:api')->group(function () {
    Route::get('/user', function (Request $request) {
        return $request->user();
    })->middleware('scope:read-posts');
});
```

Reference: [Laravel Sanctum](https://laravel.com/docs/sanctum) | [Laravel Passport](https://laravel.com/docs/passport)

---

### 4.2 Implement Rate Limiting

**Impact: HIGH** — Prevents brute force and DoS attacks

Without rate limiting, your application is vulnerable to brute force password attacks, credential stuffing, API abuse, and denial-of-service attacks. Laravel provides a built-in rate limiter that integrates with the `throttle` middleware. Configure rate limits per endpoint type: stricter limits for authentication endpoints, moderate limits for write operations, and relaxed limits for read-heavy API endpoints.

Always define rate limiters in your `AppServiceProvider` (or `RouteServiceProvider` in older Laravel versions) and apply them via middleware on route groups or individual routes.

**Incorrect**

```php
// routes/api.php

// No rate limiting at all - endpoints are wide open to abuse
Route::post('/login', [AuthController::class, 'login']);
Route::post('/register', [AuthController::class, 'register']);
Route::post('/forgot-password', [PasswordResetController::class, 'send']);

// API routes with no throttling - a single client can make unlimited requests
Route::middleware('auth:sanctum')->group(function () {
    Route::get('/users', [UserController::class, 'index']);
    Route::post('/posts', [PostController::class, 'store']);
    Route::post('/comments', [CommentController::class, 'store']);
    Route::post('/upload', [UploadController::class, 'store']);
});

// Webhook endpoint with no protection - can be flooded
Route::post('/webhooks/stripe', [StripeWebhookController::class, 'handle']);
```

**Correct**

```php
// app/Providers/AppServiceProvider.php
namespace App\Providers;

use App\Models\User;
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        $this->configureRateLimiting();
    }

    protected function configureRateLimiting(): void
    {
        // Strict limit for authentication endpoints (brute force protection)
        RateLimiter::for('auth', function (Request $request) {
            return Limit::perMinute(5)
                ->by($request->ip() . '|' . $request->input('email', ''))
                ->response(function (Request $request, array $headers) {
                    return response()->json([
                        'message' => 'Too many login attempts. Please try again later.',
                        'retry_after' => $headers['Retry-After'] ?? 60,
                    ], 429, $headers);
                });
        });

        // Password reset: very strict to prevent email bombing
        RateLimiter::for('password-reset', function (Request $request) {
            return Limit::perMinute(3)
                ->by($request->ip())
                ->response(function () {
                    return response()->json([
                        'message' => 'Too many password reset attempts. Please wait before trying again.',
                    ], 429);
                });
        });

        // Standard API rate limit: per authenticated user
        RateLimiter::for('api', function (Request $request) {
            return $request->user()
                ? Limit::perMinute(60)->by($request->user()->id)
                : Limit::perMinute(20)->by($request->ip());
        });

        // Higher limit for read-heavy endpoints
        RateLimiter::for('api-reads', function (Request $request) {
            return $request->user()
                ? Limit::perMinute(120)->by($request->user()->id)
                : Limit::perMinute(30)->by($request->ip());
        });

        // Strict limit for write/mutation endpoints
        RateLimiter::for('api-writes', function (Request $request) {
            return $request->user()
                ? Limit::perMinute(30)->by($request->user()->id)
                : Limit::none(); // Unauthenticated users cannot write (handled by auth middleware)
        });

        // Uploads: very strict due to resource cost
        RateLimiter::for('uploads', function (Request $request) {
            return Limit::perMinute(10)->by($request->user()?->id ?: $request->ip());
        });

        // Tiered rate limits based on user plan
        RateLimiter::for('api-tiered', function (Request $request) {
            $user = $request->user();

            if (! $user) {
                return Limit::perMinute(10)->by($request->ip());
            }

            return match ($user->plan) {
                'enterprise' => Limit::perMinute(500)->by($user->id),
                'pro'        => Limit::perMinute(120)->by($user->id),
                'basic'      => Limit::perMinute(60)->by($user->id),
                default      => Limit::perMinute(30)->by($user->id),
            };
        });

        // Webhook endpoints: limit by source IP
        RateLimiter::for('webhooks', function (Request $request) {
            return Limit::perMinute(100)->by($request->ip());
        });

        // Multiple limits: per-second burst + per-minute sustained
        RateLimiter::for('api-strict', function (Request $request) {
            $key = $request->user()?->id ?: $request->ip();

            return [
                Limit::perSecond(3)->by($key),   // Prevent burst abuse
                Limit::perMinute(60)->by($key),   // Sustained limit
                Limit::perHour(500)->by($key),    // Hourly cap
            ];
        });
    }
}

// routes/api.php
use App\Http\Controllers\Auth\AuthController;
use App\Http\Controllers\Auth\PasswordResetController;
use App\Http\Controllers\CommentController;
use App\Http\Controllers\PostController;
use App\Http\Controllers\UploadController;
use App\Http\Controllers\UserController;
use App\Http\Controllers\Webhook\StripeWebhookController;

// Authentication routes with strict throttling
Route::middleware('throttle:auth')->group(function () {
    Route::post('/login', [AuthController::class, 'login']);
    Route::post('/register', [AuthController::class, 'register']);
});

Route::post('/forgot-password', [PasswordResetController::class, 'send'])
    ->middleware('throttle:password-reset');

// Authenticated API routes with tiered rate limiting
Route::middleware(['auth:sanctum'])->group(function () {
    // Read endpoints: higher limit
    Route::middleware('throttle:api-reads')->group(function () {
        Route::get('/users', [UserController::class, 'index']);
        Route::get('/posts', [PostController::class, 'index']);
        Route::get('/posts/{post}', [PostController::class, 'show']);
        Route::get('/posts/{post}/comments', [CommentController::class, 'index']);
    });

    // Write endpoints: lower limit
    Route::middleware('throttle:api-writes')->group(function () {
        Route::post('/posts', [PostController::class, 'store']);
        Route::put('/posts/{post}', [PostController::class, 'update']);
        Route::delete('/posts/{post}', [PostController::class, 'destroy']);
        Route::post('/comments', [CommentController::class, 'store']);
    });

    // Upload endpoints: strictest limit
    Route::post('/upload', [UploadController::class, 'store'])
        ->middleware('throttle:uploads');
});

// Webhook with IP-based throttling
Route::post('/webhooks/stripe', [StripeWebhookController::class, 'handle'])
    ->middleware('throttle:webhooks');

// Custom rate limit response with detailed headers

// app/Http/Controllers/Auth/AuthController.php
namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

class AuthController extends Controller
{
    public function login(Request $request): JsonResponse
    {
        // Manual rate limiting for additional control within a controller
        $throttleKey = 'login:' . $request->ip() . '|' . $request->input('email');

        if (RateLimiter::tooManyAttempts($throttleKey, 5)) {
            $seconds = RateLimiter::availableIn($throttleKey);

            return response()->json([
                'message' => "Too many login attempts. Please try again in {$seconds} seconds.",
                'retry_after' => $seconds,
            ], 429);
        }

        RateLimiter::increment($throttleKey, 60); // Decay after 60 seconds

        // ... perform authentication logic ...

        // On successful login, clear the rate limiter
        RateLimiter::clear($throttleKey);

        return response()->json(['token' => $token]);
    }
}
```

Reference: [Laravel Rate Limiting](https://laravel.com/docs/routing#rate-limiting)

---

### 4.3 Sanitize Output to Prevent XSS

**Impact: HIGH** — XSS attacks can steal user sessions and data

Cross-Site Scripting (XSS) attacks occur when user-controlled data is rendered as executable HTML or JavaScript in the browser. An attacker can steal session cookies, redirect users to phishing sites, or perform actions on behalf of the victim. Laravel's Blade templating engine auto-escapes output with `{{ }}`, but developers must avoid bypassing this with `{!! !!}` for user-generated content and must handle API responses, rich text, and dynamic attributes carefully.

Always treat all user-supplied data as untrusted. Use `{{ }}` for output, the `e()` helper in code, HTMLPurifier for rich text, and Content-Security-Policy headers as a defense-in-depth layer.

**Incorrect**

```php
// resources/views/posts/show.blade.php

{{-- Using {!! !!} with user-supplied content - renders raw HTML including scripts --}}
<h1>{!! $post->title !!}</h1>
<div class="content">{!! $post->body !!}</div>

{{-- User-supplied data in HTML attributes without escaping --}}
<a href="{{ $post->website_url }}">Visit Website</a>
{{-- An attacker sets website_url to: javascript:alert(document.cookie) --}}

<img src="{{ $user->avatar_url }}" onerror="alert('xss')">

{{-- Injecting user data into inline JavaScript --}}
<script>
    var userName = "{{ $user->name }}"; // Breaks if name contains quotes/script tags
    var config = {!! json_encode($userSettings) !!}; // Raw output of user-controlled data
</script>

// app/Http/Controllers/CommentController.php
namespace App\Http\Controllers;

use App\Models\Comment;
use Illuminate\Http\Request;

class CommentController extends Controller
{
    public function store(Request $request): JsonResponse
    {
        // Storing raw HTML from user input
        $comment = Comment::create([
            'body' => $request->input('body'), // Could contain <script> tags
            'user_id' => $request->user()->id,
        ]);

        // Returning unsanitized data in API response
        // Frontend might render this with v-html or innerHTML
        return response()->json($comment);
    }
}
```

**Correct**

```php
// resources/views/posts/show.blade.php

{{-- {{ }} automatically escapes HTML entities - safe for user content --}}
<h1>{{ $post->title }}</h1>

{{-- For rich text that MUST contain HTML, sanitize server-side first (see below) --}}
<div class="content">{!! $post->sanitized_body !!}</div>

{{-- Safe URL handling - validate protocol to prevent javascript: URIs --}}
@php
    $safeUrl = filter_var($post->website_url, FILTER_VALIDATE_URL)
        && in_array(parse_url($post->website_url, PHP_URL_SCHEME), ['http', 'https'])
        ? $post->website_url
        : '#';
@endphp
<a href="{{ $safeUrl }}" rel="noopener noreferrer nofollow">Visit Website</a>

{{-- Safe JavaScript data passing with @js directive (Laravel 9+) --}}
<script>
    var userName = @js($user->name);
    var config = @js($safeConfig);
</script>

{{-- Or use data attributes instead of inline scripts --}}
<div id="app" data-user-name="{{ $user->name }}" data-config="{{ e(json_encode($safeConfig)) }}">
</div>

// app/Services/HtmlSanitizer.php
namespace App\Services;

use HTMLPurifier;
use HTMLPurifier_Config;

class HtmlSanitizer
{
    private HTMLPurifier $purifier;

    public function __construct()
    {
        $config = HTMLPurifier_Config::createDefault();

        // Allow only safe HTML tags and attributes
        $config->set('HTML.Allowed', 'p,br,strong,em,ul,ol,li,a[href|title],blockquote,code,pre,h2,h3,h4');
        $config->set('HTML.TargetBlank', true);       // Add target="_blank" to links
        $config->set('URI.AllowedSchemes', ['http', 'https', 'mailto']);
        $config->set('Attr.AllowedFrameTargets', ['_blank']);
        $config->set('AutoFormat.RemoveEmpty', true);  // Remove empty tags
        $config->set('CSS.AllowedProperties', []);     // Strip all inline CSS
        $config->set('Cache.SerializerPath', storage_path('app/purifier'));

        $this->purifier = new HTMLPurifier($config);
    }

    public function sanitize(string $html): string
    {
        return $this->purifier->purify($html);
    }
}

// app/Http/Controllers/CommentController.php
namespace App\Http\Controllers;

use App\Http\Requests\StoreCommentRequest;
use App\Models\Comment;
use Illuminate\Http\JsonResponse;

class CommentController extends Controller
{
    public function store(StoreCommentRequest $request): JsonResponse
    {
        // Plain text comments: strip all HTML tags
        $comment = Comment::create([
            'body' => strip_tags($request->validated('body')),
            'user_id' => $request->user()->id,
        ]);

        return response()->json($comment);
    }
}

// app/Http/Controllers/PostController.php
namespace App\Http\Controllers;

use App\Http\Requests\StorePostRequest;
use App\Models\Post;
use App\Services\HtmlSanitizer;
use Illuminate\Http\JsonResponse;

class PostController extends Controller
{
    public function __construct(
        private readonly HtmlSanitizer $sanitizer,
    ) {}

    public function store(StorePostRequest $request): JsonResponse
    {
        // Rich text: sanitize HTML to allow safe formatting
        $post = Post::create([
            'title' => $request->validated('title'),
            'body' => $request->validated('body'),              // Raw stored for editing
            'sanitized_body' => $this->sanitizer->sanitize(     // Sanitized for display
                $request->validated('body')
            ),
            'user_id' => $request->user()->id,
        ]);

        return response()->json($post);
    }
}

// app/Models/Post.php - Accessor for safe output in API responses
namespace App\Models;

use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
    protected function sanitizedBody(): Attribute
    {
        return Attribute::make(
            get: fn (?string $value) => $value,
        );
    }

    // Hide raw body from JSON serialization; only expose sanitized version
    protected $hidden = ['body'];
}

// app/Http/Middleware/SecurityHeaders.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class SecurityHeaders
{
    public function handle(Request $request, Closure $next): Response
    {
        $response = $next($request);

        // Content-Security-Policy: defense-in-depth against XSS
        $response->headers->set(
            'Content-Security-Policy',
            "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self'; connect-src 'self'; frame-ancestors 'none';"
        );

        // Prevent MIME type sniffing
        $response->headers->set('X-Content-Type-Options', 'nosniff');

        // Prevent clickjacking
        $response->headers->set('X-Frame-Options', 'DENY');

        // Enable browser XSS filter (legacy browsers)
        $response->headers->set('X-XSS-Protection', '1; mode=block');

        // Control referrer information
        $response->headers->set('Referrer-Policy', 'strict-origin-when-cross-origin');

        return $response;
    }
}

// bootstrap/app.php - Register the middleware globally
// ->withMiddleware(function (Middleware $middleware) {
//     $middleware->append(SecurityHeaders::class);
// })

// Using e() helper in non-Blade contexts (emails, notifications, API transformers)

// app/Http/Resources/CommentResource.php
namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class CommentResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'body' => e($this->body), // Escaped for safe frontend rendering
            'author' => $this->user->name,
            'created_at' => $this->created_at->toIso8601String(),
        ];
    }
}
```

Reference: [Laravel Blade Templates](https://laravel.com/docs/blade)

---

### 4.4 Use Guards for Authentication and Authorization

**Impact: HIGH** — Missing authorization = unauthorized data access

Never perform manual authentication or authorization checks scattered throughout your controllers. Laravel provides Guards for verifying identity (who is the user?) and Gates/Policies for verifying permissions (what can the user do?). Policies are the preferred approach for model-based authorization because they co-locate all permission logic for a model in one class, support auto-discovery, and integrate cleanly with controllers and Blade templates.

Missing or inconsistent authorization checks are the leading cause of Insecure Direct Object Reference (IDOR) vulnerabilities, where users can access or modify resources belonging to other users.

**Incorrect**

```php
// app/Http/Controllers/PostController.php
namespace App\Http\Controllers;

use App\Models\Post;
use Illuminate\Http\Request;

class PostController extends Controller
{
    public function update(Request $request, int $id): JsonResponse
    {
        $post = Post::findOrFail($id);

        // Manual auth check - easy to forget, inconsistent, hard to test
        if ($request->user()->id !== $post->user_id) {
            abort(403, 'Unauthorized');
        }

        // What about admins? Editors? This logic gets duplicated everywhere.
        $post->update($request->all());

        return response()->json($post);
    }

    public function destroy(int $id): JsonResponse
    {
        $post = Post::findOrFail($id);

        // Forgot the auth check entirely - any authenticated user can delete any post
        $post->delete();

        return response()->json(null, 204);
    }

    public function publish(int $id): JsonResponse
    {
        $post = Post::findOrFail($id);

        // Checking role with string comparison - fragile and scattered
        if (auth()->user()->role !== 'admin' && auth()->user()->role !== 'editor') {
            return response()->json(['error' => 'Not allowed'], 403);
        }

        $post->update(['published_at' => now()]);

        return response()->json($post);
    }
}
```

**Correct**

```php
// Step 1: Define Gates for simple, non-model actions

// app/Providers/AppServiceProvider.php
namespace App\Providers;

use App\Models\User;
use Illuminate\Support\Facades\Gate;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Gates for non-model actions
        Gate::define('access-admin-dashboard', function (User $user): bool {
            return $user->isAdmin();
        });

        Gate::define('view-analytics', function (User $user): bool {
            return $user->hasAnyRole(['admin', 'analyst']);
        });

        // Super-admin bypass: runs before all Gates and Policies
        Gate::before(function (User $user, string $ability): ?bool {
            if ($user->isSuperAdmin()) {
                return true; // Bypass all checks
            }

            return null; // Fall through to normal checks
        });
    }
}

// Step 2: Create a Policy for model authorization
// php artisan make:policy PostPolicy --model=Post

// app/Policies/PostPolicy.php
namespace App\Policies;

use App\Models\Post;
use App\Models\User;

class PostPolicy
{
    /**
     * Determine whether the user can view any posts.
     */
    public function viewAny(User $user): bool
    {
        return true; // All authenticated users can list posts
    }

    /**
     * Determine whether the user can view the post.
     */
    public function view(User $user, Post $post): bool
    {
        // Published posts are visible to everyone; drafts only to the author
        if ($post->isPublished()) {
            return true;
        }

        return $user->id === $post->user_id;
    }

    /**
     * Determine whether the user can create posts.
     */
    public function create(User $user): bool
    {
        return $user->hasAnyRole(['admin', 'editor', 'author']);
    }

    /**
     * Determine whether the user can update the post.
     */
    public function update(User $user, Post $post): bool
    {
        // Authors can edit their own posts; editors can edit any post
        return $user->id === $post->user_id
            || $user->hasRole('editor');
    }

    /**
     * Determine whether the user can delete the post.
     */
    public function delete(User $user, Post $post): bool
    {
        return $user->id === $post->user_id
            || $user->hasRole('admin');
    }

    /**
     * Determine whether the user can publish the post.
     */
    public function publish(User $user, Post $post): bool
    {
        return $user->hasAnyRole(['admin', 'editor']);
    }

    /**
     * Determine whether the user can restore a soft-deleted post.
     */
    public function restore(User $user, Post $post): bool
    {
        return $user->hasRole('admin');
    }

    /**
     * Determine whether the user can permanently delete the post.
     */
    public function forceDelete(User $user, Post $post): bool
    {
        return $user->hasRole('admin');
    }
}

// Step 3: Policy auto-discovery works when naming follows convention:
//   App\Models\Post  ->  App\Policies\PostPolicy
// No manual registration needed. For non-standard names, register in AppServiceProvider:
//   Gate::policy(Post::class, PostPolicy::class);

// Step 4: Use authorization in controllers

// app/Http/Controllers/PostController.php
namespace App\Http\Controllers;

use App\Http\Requests\StorePostRequest;
use App\Http\Requests\UpdatePostRequest;
use App\Models\Post;
use Illuminate\Http\JsonResponse;

class PostController extends Controller
{
    public function index(): JsonResponse
    {
        $this->authorize('viewAny', Post::class);

        $posts = Post::published()->paginate();

        return response()->json($posts);
    }

    public function store(StorePostRequest $request): JsonResponse
    {
        $this->authorize('create', Post::class);

        $post = $request->user()->posts()->create($request->validated());

        return response()->json($post, 201);
    }

    public function show(Post $post): JsonResponse
    {
        $this->authorize('view', $post);

        return response()->json($post->load('author', 'comments'));
    }

    public function update(UpdatePostRequest $request, Post $post): JsonResponse
    {
        $this->authorize('update', $post);

        $post->update($request->validated());

        return response()->json($post);
    }

    public function destroy(Post $post): JsonResponse
    {
        $this->authorize('delete', $post);

        $post->delete();

        return response()->json(null, 204);
    }

    public function publish(Post $post): JsonResponse
    {
        $this->authorize('publish', $post);

        $post->update(['published_at' => now()]);

        return response()->json($post);
    }
}

// Step 5: Apply authorization via route middleware as an alternative

// routes/api.php
use App\Http\Controllers\PostController;

Route::middleware('auth:sanctum')->group(function () {
    Route::apiResource('posts', PostController::class);

    // Middleware-based authorization for specific routes
    Route::patch('/posts/{post}/publish', [PostController::class, 'publish'])
        ->middleware('can:publish,post');

    // Gate-based middleware for non-model actions
    Route::get('/admin/dashboard', [AdminController::class, 'index'])
        ->middleware('can:access-admin-dashboard');
});

// Step 6: Use policies in Blade templates

// resources/views/posts/show.blade.php
// @can('update', $post)
//     <a href="{{ route('posts.edit', $post) }}">Edit Post</a>
// @endcan
//
// @can('delete', $post)
//     <form method="POST" action="{{ route('posts.destroy', $post) }}">
//         @csrf @method('DELETE')
//         <button type="submit">Delete</button>
//     </form>
// @endcan
//
// @cannot('publish', $post)
//     <p>You do not have permission to publish this post.</p>
// @endcannot
```

Reference: [Laravel Authorization](https://laravel.com/docs/authorization)

---

### 4.5 Validate All Input with Form Requests

**Impact: CRITICAL** — Unvalidated input is the #1 attack vector

Never trust raw user input. Unvalidated data leads to SQL injection, XSS, mass assignment, type confusion, and business logic bypasses. Always use dedicated Form Request classes to validate, authorize, and sanitize input before it reaches your controllers or services. Form Requests centralize validation logic, keep controllers clean, and provide consistent error responses.

Avoid calling `$request->all()` or `$request->input()` without validation. Always use `$request->validated()` to ensure only validated fields are passed downstream.

**Incorrect**

```php
// app/Http/Controllers/UserController.php
namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;

class UserController extends Controller
{
    public function store(Request $request): JsonResponse
    {
        // Using $request->all() passes every field including unexpected ones
        // Vulnerable to mass assignment even with $fillable (extra fields leak into logic)
        $user = User::create($request->all());

        return response()->json($user, 201);
    }

    public function update(Request $request, User $user): JsonResponse
    {
        // No validation at all - accepts any data type, any length, any format
        // An attacker could send: {"role": "admin", "email_verified_at": "2024-01-01"}
        $user->update($request->input());

        return response()->json($user);
    }

    public function search(Request $request): JsonResponse
    {
        // Raw input used directly in query - SQL injection risk with some drivers
        // No type checking - $request->input('per_page') could be "999999" or "-1"
        $users = User::where('name', 'like', '%' . $request->input('q') . '%')
            ->paginate($request->input('per_page'));

        return response()->json($users);
    }
}
```

**Correct**

```php
// Create Form Requests: php artisan make:request StoreUserRequest

// app/Http/Requests/StoreUserRequest.php
namespace App\Http\Requests;

use App\Enums\UserRole;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;
use Illuminate\Validation\Rules\Password;

class StoreUserRequest extends FormRequest
{
    /**
     * Determine if the user is authorized to make this request.
     */
    public function authorize(): bool
    {
        // Use policies for complex authorization; simple checks can go here
        return $this->user()->can('create', User::class);
    }

    /**
     * Get the validation rules that apply to the request.
     */
    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'min:2', 'max:255'],
            'email' => [
                'required',
                'string',
                'email:rfc,dns', // Validates format AND checks DNS for MX record
                'max:255',
                Rule::unique('users', 'email'),
            ],
            'password' => [
                'required',
                'confirmed',
                Password::min(8)
                    ->mixedCase()
                    ->numbers()
                    ->symbols()
                    ->uncompromised(), // Checks against Have I Been Pwned API
            ],
            'role' => ['sometimes', Rule::enum(UserRole::class)],
            'profile.bio' => ['nullable', 'string', 'max:1000'],       // Nested validation
            'profile.avatar' => ['nullable', 'image', 'max:2048'],     // Max 2MB
            'tags' => ['nullable', 'array', 'max:10'],                 // Array validation
            'tags.*' => ['string', 'max:50', 'distinct'],              // Each tag validated
        ];
    }

    /**
     * Custom error messages for specific rules.
     */
    public function messages(): array
    {
        return [
            'email.unique' => 'An account with this email already exists.',
            'password.uncompromised' => 'This password has appeared in a data breach. Please choose a different one.',
            'tags.max' => 'You may assign a maximum of 10 tags.',
        ];
    }

    /**
     * Prepare the data for validation (sanitize before validating).
     */
    protected function prepareForValidation(): void
    {
        $this->merge([
            'email' => strtolower(trim($this->email ?? '')),
            'name' => trim($this->name ?? ''),
        ]);
    }
}

// app/Http/Requests/UpdateUserRequest.php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;
use Illuminate\Validation\Rules\Password;

class UpdateUserRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('update', $this->route('user'));
    }

    public function rules(): array
    {
        $userId = $this->route('user')->id;

        return [
            'name' => ['sometimes', 'string', 'min:2', 'max:255'],
            'email' => [
                'sometimes',
                'string',
                'email:rfc,dns',
                'max:255',
                Rule::unique('users', 'email')->ignore($userId),
            ],
            'password' => [
                'sometimes',
                'confirmed',
                Password::defaults(),
            ],
            // Conditional validation: require reason when deactivating
            'is_active' => ['sometimes', 'boolean'],
            'deactivation_reason' => [
                Rule::requiredIf(fn () => $this->input('is_active') === false),
                'nullable',
                'string',
                'max:500',
            ],
        ];
    }
}

// app/Http/Requests/SearchUsersRequest.php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class SearchUsersRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'q' => ['nullable', 'string', 'max:100'],
            'role' => ['nullable', 'string', Rule::in(['admin', 'editor', 'viewer'])],
            'per_page' => ['nullable', 'integer', 'min:1', 'max:100'],
            'sort_by' => ['nullable', 'string', Rule::in(['name', 'email', 'created_at'])],
            'sort_dir' => ['nullable', 'string', Rule::in(['asc', 'desc'])],
        ];
    }
}

// app/Http/Controllers/UserController.php
namespace App\Http\Controllers;

use App\Http\Requests\SearchUsersRequest;
use App\Http\Requests\StoreUserRequest;
use App\Http\Requests\UpdateUserRequest;
use App\Models\User;
use Illuminate\Http\JsonResponse;

class UserController extends Controller
{
    public function store(StoreUserRequest $request): JsonResponse
    {
        // Only validated fields are used - no mass assignment risk
        $user = User::create($request->validated());

        return response()->json($user, 201);
    }

    public function update(UpdateUserRequest $request, User $user): JsonResponse
    {
        // safe() returns only validated data, safe()->only() for a subset
        $user->update($request->safe()->except(['role']));

        return response()->json($user);
    }

    public function search(SearchUsersRequest $request): JsonResponse
    {
        $validated = $request->validated();

        $users = User::query()
            ->when($validated['q'] ?? null, fn ($query, $search) =>
                $query->where('name', 'like', '%' . $search . '%')
            )
            ->when($validated['role'] ?? null, fn ($query, $role) =>
                $query->where('role', $role)
            )
            ->orderBy(
                $validated['sort_by'] ?? 'created_at',
                $validated['sort_dir'] ?? 'desc',
            )
            ->paginate($validated['per_page'] ?? 15);

        return response()->json($users);
    }
}

// Custom Validation Rule: php artisan make:rule NotDisposableEmail

// app/Rules/NotDisposableEmail.php
namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ValidationRule;

class NotDisposableEmail implements ValidationRule
{
    private const DISPOSABLE_DOMAINS = [
        'mailinator.com', 'guerrillamail.com', 'tempmail.com',
    ];

    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        $domain = strtolower(substr(strrchr($value, '@'), 1));

        if (in_array($domain, self::DISPOSABLE_DOMAINS, true)) {
            $fail('Disposable email addresses are not allowed.');
        }
    }
}
```

Reference: [Laravel Validation](https://laravel.com/docs/validation)

---

## 5. Performance

**Section Impact: HIGH**

### 5.1 Use Lazy Loading and Route Caching

**Impact: MEDIUM-HIGH** — "Reduces memory usage and startup time"

Standard Eloquent collections load all results into memory at once, which becomes a problem with large datasets. Laravel's `LazyCollection` streams results one at a time using PHP generators, keeping memory usage constant regardless of dataset size. Combined with production caching commands (`route:cache`, `config:cache`, `event:cache`), this significantly reduces both memory footprint and request latency.

In development, use `Model::preventLazyLoading()` to catch accidental N+1 queries early -- it throws an exception whenever a relationship is lazy-loaded instead of eager-loaded.

**Incorrect**

```php
// app/Http/Controllers/ExportController.php
namespace App\Http\Controllers;

use App\Models\Transaction;
use App\Models\User;

class ExportController extends Controller
{
    public function exportTransactions()
    {
        // BAD: Loading 100k+ records into memory at once
        // This will consume hundreds of MB of RAM and may trigger OOM errors
        $transactions = Transaction::all();

        $csv = fopen('php://output', 'w');
        header('Content-Type: text/csv');

        foreach ($transactions as $transaction) {
            // BAD: Lazy loading user on each iteration (N+1 problem)
            fputcsv($csv, [
                $transaction->id,
                $transaction->user->name,  // Fires a query per row
                $transaction->amount,
                $transaction->created_at,
            ]);
        }

        fclose($csv);
    }

    public function generateReport()
    {
        // BAD: get() loads ALL matching users into a Collection
        $users = User::where('is_active', true)->get();

        // BAD: Chaining collection methods on 50k+ records in memory
        $highValueUsers = $users
            ->filter(fn ($user) => $user->orders->sum('total') > 1000)
            ->sortByDesc(fn ($user) => $user->orders->sum('total'))
            ->values();

        return view('reports.high-value', compact('highValueUsers'));
    }
}

// app/Console/Commands/ProcessUsers.php
namespace App\Console\Commands;

use App\Models\User;
use Illuminate\Console\Command;

class ProcessUsers extends Command
{
    protected $signature = 'users:process';

    public function handle(): void
    {
        // BAD: Loading all users into memory to process them
        $users = User::all();

        foreach ($users as $user) {
            $this->processUser($user);
        }
    }
}
```

**Correct**

```php
// app/Http/Controllers/ExportController.php
namespace App\Http\Controllers;

use App\Models\Transaction;
use App\Models\User;
use Illuminate\Support\LazyCollection;
use Symfony\Component\HttpFoundation\StreamedResponse;

class ExportController extends Controller
{
    public function exportTransactions(): StreamedResponse
    {
        return response()->streamDownload(function () {
            $csv = fopen('php://output', 'w');
            fputcsv($csv, ['ID', 'User', 'Amount', 'Date']);

            // GOOD: cursor() returns a LazyCollection -- one row in memory at a time
            // Eager load 'user' to prevent N+1 queries
            Transaction::query()
                ->select(['id', 'user_id', 'amount', 'created_at'])
                ->with('user:id,name')
                ->orderBy('id')
                ->cursor()
                ->each(function (Transaction $transaction) use ($csv) {
                    fputcsv($csv, [
                        $transaction->id,
                        $transaction->user->name,
                        $transaction->amount,
                        $transaction->created_at->toDateString(),
                    ]);
                });

            fclose($csv);
        }, 'transactions.csv', ['Content-Type' => 'text/csv']);
    }

    public function generateReport()
    {
        // GOOD: Use database aggregation instead of loading everything into PHP
        $highValueUsers = User::query()
            ->select(['users.id', 'users.name', 'users.email'])
            ->where('is_active', true)
            ->withSum('orders', 'total')
            ->having('orders_sum_total', '>', 1000)
            ->orderByDesc('orders_sum_total')
            ->paginate(25);

        return view('reports.high-value', compact('highValueUsers'));
    }
}

// app/Console/Commands/ProcessUsers.php
namespace App\Console\Commands;

use App\Models\User;
use Illuminate\Console\Command;

class ProcessUsers extends Command
{
    protected $signature = 'users:process';

    public function handle(): void
    {
        // GOOD: chunk() processes 500 records at a time, freeing memory between chunks
        User::query()
            ->select(['id', 'name', 'email'])
            ->orderBy('id')
            ->chunk(500, function ($users) {
                foreach ($users as $user) {
                    $this->processUser($user);
                }
            });

        // ALTERNATIVE: chunkById() is safer if records are modified during processing
        // It uses WHERE id > ? instead of OFFSET, avoiding skipped/duplicated rows
        User::query()
            ->where('needs_processing', true)
            ->chunkById(500, function ($users) {
                foreach ($users as $user) {
                    $this->processUser($user);
                    $user->update(['needs_processing' => false]);
                }
            });

        // ALTERNATIVE: cursor() for minimal memory when no batching is needed
        User::query()
            ->select(['id', 'name', 'email'])
            ->cursor()
            ->each(fn (User $user) => $this->processUser($user));
    }
}

// app/Providers/AppServiceProvider.php
namespace App\Providers;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // GOOD: Prevent lazy loading in non-production environments
        // Throws a LazyLoadingViolationException when a relationship is accessed
        // without being eager-loaded, catching N+1 issues during development
        Model::preventLazyLoading(! $this->app->isProduction());

        // GOOD: Prevent silently discarding attributes not in $fillable
        Model::preventSilentlyDiscardingAttributes(! $this->app->isProduction());

        // GOOD: Prevent accessing missing attributes (returns null by default)
        Model::preventAccessingMissingAttributes(! $this->app->isProduction());
    }
}

// Production deployment: run these caching commands after every deploy
//
// php artisan route:cache   -- Caches all route registrations into a single file.
//                              Dramatically speeds up route resolution, especially
//                              with hundreds of routes. Must re-run after any route change.
//
// php artisan config:cache  -- Merges all config files into one cached file.
//                              Eliminates filesystem reads for config on each request.
//                              WARNING: config() works but env() returns null after caching.
//                              Always use config() in application code, never env().
//
// php artisan event:cache   -- Caches the event-to-listener mapping so Laravel
//                              doesn't scan provider boot() methods on each request.
//
// php artisan view:cache    -- Pre-compiles all Blade templates so they don't need
//                              to be compiled on first render in production.
//
// Example deploy script:
// php artisan config:cache && php artisan route:cache && php artisan event:cache && php artisan view:cache
```

Reference: [Laravel Collections](https://laravel.com/docs/collections)

---

### 5.2 Optimize Database Queries

**Impact: HIGH** — "Database is the most common performance bottleneck"

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

---

### 5.3 Use Service Provider Lifecycle Correctly

**Impact: HIGH** — "Wrong lifecycle usage slows application boot"

Laravel Service Providers have two distinct lifecycle methods: `register()` and `boot()`. Misusing these methods leads to slower application startup, unexpected binding resolution errors, and tightly coupled initialization logic. The `register()` method should only bind things into the container -- never resolve services, run database queries, or perform I/O. The `boot()` method runs after all providers are registered, making it safe to resolve other services and perform complex initialization.

For providers that are not needed on every request, implement the `DeferrableProvider` interface so Laravel only instantiates them when their bindings are actually resolved.

**Incorrect**

```php
// app/Providers/ReportingServiceProvider.php
namespace App\Providers;

use App\Services\Reporting\ReportingService;
use App\Services\Billing\BillingService;
use Illuminate\Support\ServiceProvider;
use Illuminate\Support\Facades\DB;

class ReportingServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // BAD: Running a database query during register()
        // This executes on EVERY request, even when ReportingService is never used
        $config = DB::table('reporting_config')->first();

        // BAD: Resolving another service during register()
        // BillingServiceProvider may not be registered yet
        $billing = $this->app->make(BillingService::class);

        $this->app->singleton(ReportingService::class, function ($app) use ($config, $billing) {
            return new ReportingService($billing, $config->driver ?? 'default');
        });

        // BAD: Registering event listeners in register()
        // Event system may not be fully initialized
        \Event::listen('report.generated', function ($report) {
            logger()->info('Report generated: ' . $report->id);
        });
    }
}

// app/Providers/NotificationServiceProvider.php
namespace App\Providers;

use App\Services\Notification\NotificationService;
use App\Services\Notification\Channels\SlackChannel;
use App\Services\Notification\Channels\EmailChannel;
use Illuminate\Support\ServiceProvider;

class NotificationServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // BAD: Heavy initialization logic in register()
        // This runs on every request even if notifications are never sent
        $this->app->singleton(NotificationService::class, function ($app) {
            $service = new NotificationService();

            // BAD: Resolving config-dependent channels during registration
            $service->addChannel(new SlackChannel(
                $app->make('config')->get('services.slack.webhook'),
                $app->make('http')->timeout(30),
            ));

            $service->addChannel(new EmailChannel(
                $app->make('mailer'),
            ));

            return $service;
        });
    }
}
```

**Correct**

```php
// app/Providers/ReportingServiceProvider.php
namespace App\Providers;

use App\Services\Reporting\ReportingService;
use App\Services\Billing\BillingService;
use Illuminate\Contracts\Support\DeferrableProvider;
use Illuminate\Support\ServiceProvider;

class ReportingServiceProvider extends ServiceProvider implements DeferrableProvider
{
    /**
     * Only bind into the container. No DB queries, no resolving other services.
     */
    public function register(): void
    {
        $this->app->singleton(ReportingService::class, function ($app) {
            // Safe: closures are evaluated lazily, only when the service is resolved
            return new ReportingService(
                $app->make(BillingService::class),
                $app->make('config')->get('reporting.driver', 'default'),
            );
        });
    }

    /**
     * Boot runs after ALL providers are registered. Safe to resolve services,
     * register event listeners, and perform complex initialization here.
     */
    public function boot(): void
    {
        \Event::listen('report.generated', function ($report) {
            logger()->info('Report generated: ' . $report->id);
        });
    }

    /**
     * DeferrableProvider: tell Laravel which bindings this provider offers.
     * The provider is only instantiated when one of these is resolved.
     */
    public function provides(): array
    {
        return [ReportingService::class];
    }
}

// app/Providers/NotificationServiceProvider.php
namespace App\Providers;

use App\Services\Notification\NotificationService;
use App\Services\Notification\Channels\SlackChannel;
use App\Services\Notification\Channels\EmailChannel;
use Illuminate\Contracts\Support\DeferrableProvider;
use Illuminate\Support\ServiceProvider;

class NotificationServiceProvider extends ServiceProvider implements DeferrableProvider
{
    public function register(): void
    {
        // Only bind -- channel registration happens in boot() or lazily
        $this->app->singleton(NotificationService::class);

        $this->app->singleton(SlackChannel::class);
        $this->app->singleton(EmailChannel::class);
    }

    public function boot(): void
    {
        // Safe to resolve services and configure channels here
        $this->app->afterResolving(NotificationService::class, function ($service, $app) {
            $service->addChannel($app->make(SlackChannel::class));
            $service->addChannel($app->make(EmailChannel::class));
        });
    }

    public function provides(): array
    {
        return [
            NotificationService::class,
            SlackChannel::class,
            EmailChannel::class,
        ];
    }
}

// Store configuration in config/reporting.php instead of the database
// config/reporting.php
return [
    'driver' => env('REPORTING_DRIVER', 'default'),
    'export_path' => storage_path('app/reports'),
];
```

Reference: [Laravel Service Providers](https://laravel.com/docs/providers)

---

### 5.4 Use Caching Strategically

**Impact: HIGH** — "Proper caching can reduce response times by 90%+"

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

---

## 6. Testing

**Section Impact: MEDIUM-HIGH**

### 6.1 Use HTTP Tests for Feature Testing

**Impact: HIGH** — HTTP tests catch integration issues that unit tests miss

Laravel provides a fluent HTTP testing API that simulates real requests through the full middleware and routing stack. Use `$this->getJson()`, `postJson()`, `putJson()`, `deleteJson()`, and related methods to test your application as a client would interact with it. These methods exercise routing, middleware, validation, controllers, and database persistence together, catching integration issues that isolated unit tests miss.

Never instantiate controllers directly or call their methods in tests. This bypasses middleware, route model binding, form request validation, and other framework behavior that runs in production.

**Incorrect**

```php
<?php

// tests/Feature/PostControllerTest.php
// BAD: Testing controllers by calling methods directly

namespace Tests\Feature;

use App\Http\Controllers\PostController;
use App\Models\Post;
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class PostControllerTest extends TestCase
{
    use RefreshDatabase;

    public function test_create_post(): void
    {
        // BAD: Instantiating controller directly bypasses middleware,
        // route model binding, form requests, and dependency injection
        $controller = new PostController();

        $request = new Request([
            'title' => 'My Post',
            'body' => 'Content here',
        ]);

        // BAD: No middleware check, no validation, no auth
        $response = $controller->store($request);

        $this->assertEquals(201, $response->getStatusCode());
    }

    public function test_list_posts(): void
    {
        // BAD: Calling index() directly skips pagination middleware,
        // query parameter handling, and response transformation
        $controller = new PostController();
        $request = new Request();

        $result = $controller->index($request);

        $this->assertNotEmpty($result);
    }
}
```

**Correct**

```php
<?php

// tests/Feature/PostControllerTest.php

namespace Tests\Feature;

use App\Models\Post;
use App\Models\User;
use App\Models\Comment;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;
use Tests\TestCase;

class PostControllerTest extends TestCase
{
    use RefreshDatabase;

    // -------------------------------------------------------
    // Basic CRUD with JSON assertions
    // -------------------------------------------------------

    public function test_authenticated_user_can_create_post(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)
            ->postJson('/api/posts', [
                'title' => 'Understanding Laravel Testing',
                'body' => 'HTTP tests are essential for verifying application behavior.',
                'category_id' => 3,
            ]);

        $response->assertCreated()
            ->assertJson([
                'data' => [
                    'title' => 'Understanding Laravel Testing',
                    'author' => [
                        'id' => $user->id,
                        'name' => $user->name,
                    ],
                ],
            ]);

        $this->assertDatabaseHas('posts', [
            'title' => 'Understanding Laravel Testing',
            'user_id' => $user->id,
            'category_id' => 3,
        ]);
    }

    public function test_can_list_posts_with_pagination(): void
    {
        Post::factory()->count(25)->create();

        $response = $this->getJson('/api/posts?page=1&per_page=10');

        $response->assertOk()
            ->assertJsonCount(10, 'data')
            ->assertJsonStructure([
                'data' => [
                    '*' => ['id', 'title', 'body', 'author', 'created_at'],
                ],
                'meta' => ['current_page', 'last_page', 'per_page', 'total'],
            ]);
    }

    public function test_can_show_single_post(): void
    {
        $post = Post::factory()->create();

        $response = $this->getJson("/api/posts/{$post->id}");

        $response->assertOk()
            ->assertJson([
                'data' => [
                    'id' => $post->id,
                    'title' => $post->title,
                ],
            ]);
    }

    public function test_author_can_update_own_post(): void
    {
        $user = User::factory()->create();
        $post = Post::factory()->for($user, 'author')->create();

        $response = $this->actingAs($user)
            ->putJson("/api/posts/{$post->id}", [
                'title' => 'Updated Title',
                'body' => 'Updated content.',
            ]);

        $response->assertOk()
            ->assertJson([
                'data' => [
                    'title' => 'Updated Title',
                ],
            ]);

        $this->assertDatabaseHas('posts', [
            'id' => $post->id,
            'title' => 'Updated Title',
        ]);
    }

    public function test_author_can_delete_own_post(): void
    {
        $user = User::factory()->create();
        $post = Post::factory()->for($user, 'author')->create();

        $response = $this->actingAs($user)
            ->deleteJson("/api/posts/{$post->id}");

        $response->assertNoContent();

        $this->assertDatabaseMissing('posts', [
            'id' => $post->id,
        ]);
    }

    // -------------------------------------------------------
    // Validation error responses
    // -------------------------------------------------------

    public function test_creating_post_requires_title_and_body(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)
            ->postJson('/api/posts', [
                'title' => '',
                'body' => '',
            ]);

        $response->assertUnprocessable()
            ->assertJsonValidationErrors(['title', 'body']);
    }

    public function test_title_must_be_unique(): void
    {
        $user = User::factory()->create();
        Post::factory()->create(['title' => 'Existing Title']);

        $response = $this->actingAs($user)
            ->postJson('/api/posts', [
                'title' => 'Existing Title',
                'body' => 'Some content.',
            ]);

        $response->assertUnprocessable()
            ->assertJsonValidationErrorFor('title');
    }

    // -------------------------------------------------------
    // Authentication and authorization
    // -------------------------------------------------------

    public function test_guest_cannot_create_post(): void
    {
        $response = $this->postJson('/api/posts', [
            'title' => 'Unauthorized Post',
            'body' => 'Should be rejected.',
        ]);

        $response->assertUnauthorized();
    }

    public function test_non_author_cannot_update_post(): void
    {
        $author = User::factory()->create();
        $otherUser = User::factory()->create();
        $post = Post::factory()->for($author, 'author')->create();

        $response = $this->actingAs($otherUser)
            ->putJson("/api/posts/{$post->id}", [
                'title' => 'Hijacked Title',
            ]);

        $response->assertForbidden();
    }

    public function test_admin_can_delete_any_post(): void
    {
        $admin = User::factory()->admin()->create();
        $post = Post::factory()->create();

        $response = $this->actingAs($admin)
            ->deleteJson("/api/posts/{$post->id}");

        $response->assertNoContent();
        $this->assertDatabaseMissing('posts', ['id' => $post->id]);
    }

    // -------------------------------------------------------
    // File uploads
    // -------------------------------------------------------

    public function test_user_can_upload_post_cover_image(): void
    {
        Storage::fake('public');

        $user = User::factory()->create();
        $post = Post::factory()->for($user, 'author')->create();

        $file = UploadedFile::fake()->image('cover.jpg', 1200, 800);

        $response = $this->actingAs($user)
            ->postJson("/api/posts/{$post->id}/cover", [
                'cover_image' => $file,
            ]);

        $response->assertOk()
            ->assertJsonPath('data.has_cover', true);

        Storage::disk('public')->assertExists("covers/{$post->id}/{$file->hashName()}");
    }

    public function test_cover_image_must_be_valid_image(): void
    {
        Storage::fake('public');

        $user = User::factory()->create();
        $post = Post::factory()->for($user, 'author')->create();

        $file = UploadedFile::fake()->create('document.pdf', 500, 'application/pdf');

        $response = $this->actingAs($user)
            ->postJson("/api/posts/{$post->id}/cover", [
                'cover_image' => $file,
            ]);

        $response->assertUnprocessable()
            ->assertJsonValidationErrorFor('cover_image');
    }

    // -------------------------------------------------------
    // Testing middleware behavior
    // -------------------------------------------------------

    public function test_rate_limiting_is_enforced(): void
    {
        $user = User::factory()->create();

        // Exceed the rate limit (assuming 60 requests per minute)
        for ($i = 0; $i < 61; $i++) {
            $response = $this->actingAs($user)
                ->getJson('/api/posts');
        }

        $response->assertStatus(429);
    }

    // -------------------------------------------------------
    // Testing response headers and structures
    // -------------------------------------------------------

    public function test_list_posts_returns_correct_headers(): void
    {
        Post::factory()->count(5)->create();

        $response = $this->getJson('/api/posts');

        $response->assertOk()
            ->assertHeader('Content-Type', 'application/json');
    }

    // -------------------------------------------------------
    // Testing relationships and includes
    // -------------------------------------------------------

    public function test_can_include_comments_with_post(): void
    {
        $post = Post::factory()
            ->has(Comment::factory()->count(3))
            ->create();

        $response = $this->getJson("/api/posts/{$post->id}?include=comments");

        $response->assertOk()
            ->assertJsonCount(3, 'data.comments')
            ->assertJsonStructure([
                'data' => [
                    'id',
                    'title',
                    'comments' => [
                        '*' => ['id', 'body', 'author', 'created_at'],
                    ],
                ],
            ]);
    }
}
```

Reference: [Laravel HTTP Tests](https://laravel.com/docs/http-tests)

---

### 6.2 Mock External Services in Tests

**Impact: MEDIUM-HIGH** — External service calls make tests slow and flaky

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

---

### 6.3 Use Laravel TestCase and RefreshDatabase

**Impact: HIGH** — Enables proper isolated testing with fresh database state

Every test in a Laravel application should extend `Tests\TestCase` and use the `RefreshDatabase` trait to guarantee proper test isolation. Laravel's TestCase bootstraps the entire application, giving tests access to the service container, helpers, and testing utilities. The `RefreshDatabase` trait ensures each test starts with a clean, migrated database by wrapping test execution in a transaction that is rolled back afterward (or by re-migrating when necessary).

Without these, tests lack access to Laravel's testing infrastructure, cannot use factories or assertions like `assertDatabaseHas()`, and risk leaking state between tests.

For test suites where migration speed is a concern, `DatabaseTransactions` can be used instead of `RefreshDatabase` when the database schema is already up to date (e.g., in CI environments where migrations run once before the suite).

**Incorrect**

```php
<?php

// tests/Unit/UserServiceTest.php
// BAD: Using raw PHPUnit without Laravel bootstrapping

use PHPUnit\Framework\TestCase;

class UserServiceTest extends TestCase
{
    protected function setUp(): void
    {
        parent::setUp();

        // Manual database connection - fragile and error-prone
        $this->pdo = new PDO('mysql:host=localhost;dbname=test_db', 'root', '');
        $this->pdo->beginTransaction();
    }

    protected function tearDown(): void
    {
        // Manual cleanup - easy to forget, leaks state on failure
        $this->pdo->rollBack();
        parent::tearDown();
    }

    public function test_user_can_be_created(): void
    {
        // Manual SQL insertion - no factory support, no model events
        $this->pdo->exec(
            "INSERT INTO users (name, email, password) VALUES ('John', 'john@example.com', 'hashed')"
        );

        $stmt = $this->pdo->query("SELECT * FROM users WHERE email = 'john@example.com'");
        $user = $stmt->fetch(PDO::FETCH_ASSOC);

        $this->assertEquals('John', $user['name']);
    }

    public function test_user_requires_authentication(): void
    {
        // No way to simulate authentication without Laravel's auth system
        // Manual session/token handling is brittle
        $_SESSION['user_id'] = 1;
        $controller = new UserController();
        $result = $controller->profile();

        $this->assertNotNull($result);
    }
}
```

**Correct**

```php
<?php

// tests/Feature/UserServiceTest.php (PHPUnit style)

namespace Tests\Feature;

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class UserServiceTest extends TestCase
{
    use RefreshDatabase;

    public function test_user_can_be_created(): void
    {
        $user = User::factory()->create([
            'name' => 'John Doe',
            'email' => 'john@example.com',
        ]);

        $this->assertDatabaseHas('users', [
            'email' => 'john@example.com',
            'name' => 'John Doe',
        ]);

        $this->assertInstanceOf(User::class, $user);
        $this->assertEquals('John Doe', $user->name);
    }

    public function test_user_can_access_profile_when_authenticated(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)
            ->getJson('/api/user/profile');

        $response->assertOk()
            ->assertJson([
                'id' => $user->id,
                'name' => $user->name,
                'email' => $user->email,
            ]);
    }

    public function test_guest_cannot_access_profile(): void
    {
        $response = $this->getJson('/api/user/profile');

        $response->assertUnauthorized();
    }
}
```

```php
<?php

// tests/Feature/UserServiceTest.php (Pest style)

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

it('can create a user', function () {
    $user = User::factory()->create([
        'name' => 'John Doe',
        'email' => 'john@example.com',
    ]);

    $this->assertDatabaseHas('users', [
        'email' => 'john@example.com',
        'name' => 'John Doe',
    ]);

    expect($user)
        ->toBeInstanceOf(User::class)
        ->name->toBe('John Doe');
});

it('allows authenticated users to access their profile', function () {
    $user = User::factory()->create();

    $response = $this->actingAs($user)
        ->getJson('/api/user/profile');

    $response->assertOk()
        ->assertJson([
            'id' => $user->id,
            'name' => $user->name,
        ]);
});

it('rejects guest access to profile', function () {
    $this->getJson('/api/user/profile')
        ->assertUnauthorized();
});
```

```php
<?php

// tests/Feature/ReportGenerationTest.php
// Using DatabaseTransactions for faster tests when schema is already migrated

namespace Tests\Feature;

use App\Models\User;
use App\Models\Report;
use Illuminate\Foundation\Testing\DatabaseTransactions;
use Tests\TestCase;

class ReportGenerationTest extends TestCase
{
    // DatabaseTransactions wraps each test in a transaction and rolls back.
    // Faster than RefreshDatabase because it skips migration checks,
    // but requires the database schema to already be up to date.
    use DatabaseTransactions;

    public function test_user_can_generate_report(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)
            ->postJson('/api/reports', [
                'title' => 'Monthly Sales',
                'type' => 'sales',
                'date_range' => ['2026-01-01', '2026-01-31'],
            ]);

        $response->assertCreated();

        $this->assertDatabaseHas('reports', [
            'user_id' => $user->id,
            'title' => 'Monthly Sales',
        ]);
    }
}
```

Reference: [Laravel Testing - Getting Started](https://laravel.com/docs/testing)

---

## 7. Database & ORM

**Section Impact: MEDIUM-HIGH**

### 7.1 Avoid N+1 Query Problems

**Impact: HIGH** — N+1 queries are one of the most common performance killers

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

---

### 7.2 Use Database Migrations

**Impact: HIGH** — Manual schema changes cause deployment failures

Always use migrations for every schema change. Migrations are version-controlled, reproducible, and run automatically during deployments. Never modify the database directly through a GUI tool or raw SQL scripts outside the migration system -- manual changes are invisible to your team, cannot be rolled back reliably, and will cause deployment failures when environments drift apart.

**Incorrect**

```php
// Running raw SQL directly against the database
DB::statement('ALTER TABLE users ADD COLUMN phone VARCHAR(20)');

// Or worse: manually running SQL in a database GUI tool
// "I'll just add the column in phpMyAdmin real quick..."

// Migrations without a proper down() method
public function up(): void
{
    Schema::table('users', function (Blueprint $table) {
        $table->string('phone', 20)->nullable();
    });
}

public function down(): void
{
    // Empty - can't roll back
}

// Modifying an existing migration that has already been run
// This does nothing because Laravel tracks which migrations have run.
// Other developers and environments will never see the change.

// Putting too many unrelated changes in a single migration
public function up(): void
{
    Schema::create('orders', function (Blueprint $table) { /* ... */ });
    Schema::create('invoices', function (Blueprint $table) { /* ... */ });
    Schema::create('shipping_labels', function (Blueprint $table) { /* ... */ });
    Schema::table('users', function (Blueprint $table) {
        $table->string('stripe_id')->nullable();
    });
    // If any of these fail, you can't roll back the ones that succeeded
}
```

**Correct**

```php
// Generate a migration with artisan
// php artisan make:migration add_phone_to_users_table --table=users

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->string('phone', 20)->nullable()->after('email');
            $table->index('phone');
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropIndex(['phone']);
            $table->dropColumn('phone');
        });
    }
};

// Creating a table with proper foreign keys and indexes
return new class extends Migration
{
    public function up(): void
    {
        Schema::create('orders', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->foreignId('product_id')->constrained()->restrictOnDelete();
            $table->string('status', 30)->default('pending');
            $table->decimal('total', 10, 2);
            $table->timestamp('shipped_at')->nullable();
            $table->timestamps();
            $table->softDeletes();

            $table->index('status');
            $table->index(['user_id', 'status']); // Composite index for common queries
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('orders');
    }
};

// Zero-downtime migration strategy for adding a required column
// Step 1: Add column as nullable (safe, no locks on large tables)
return new class extends Migration
{
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->string('timezone', 50)->nullable()->after('email');
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn('timezone');
        });
    }
};

// Step 2: Backfill existing rows (separate migration or artisan command)
return new class extends Migration
{
    public function up(): void
    {
        DB::table('users')
            ->whereNull('timezone')
            ->update(['timezone' => 'UTC']);
    }

    public function down(): void
    {
        // No rollback needed for data backfill
    }
};

// Step 3: Add the NOT NULL constraint once all rows are populated
return new class extends Migration
{
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->string('timezone', 50)->default('UTC')->change();
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->string('timezone', 50)->nullable()->change();
        });
    }
};

// Renaming columns safely (requires doctrine/dbal)
return new class extends Migration
{
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->renameColumn('name', 'full_name');
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->renameColumn('full_name', 'name');
        });
    }
};

// Squash migrations once your app is mature to speed up fresh installs
// php artisan schema:dump
// php artisan schema:dump --prune  (dumps then deletes old migration files)
// This creates a database/schema directory with a SQL dump that runs before
// any remaining migrations, dramatically speeding up migrate:fresh.
```

**Reference:** [Laravel Migrations](https://laravel.com/docs/migrations)

---

### 7.3 Use Transactions for Multi-Step Operations

**Impact: HIGH** — Partial writes cause data inconsistency

When a business operation requires multiple database writes that must succeed or fail together, wrap them in a transaction. Without a transaction, a failure partway through leaves your data in an inconsistent state -- for example, an order is created but the inventory is never decremented, or money is deducted from one account but never credited to another.

**Incorrect**

```php
// Multiple writes without a transaction - if any step fails,
// previous steps are already committed and cannot be undone
public function placeOrder(Request $request): Order
{
    $order = Order::create([
        'user_id' => $request->user()->id,
        'total' => $request->total,
    ]);

    foreach ($request->items as $item) {
        $order->items()->create($item); // What if this fails on item 3 of 5?
    }

    $order->user->decrement('balance', $request->total); // What if this fails?

    Payment::create([
        'order_id' => $order->id,
        'amount' => $request->total,
        'status' => 'completed',
    ]);

    return $order;
}

// Catching exceptions but still having partial writes
public function transferFunds(Account $from, Account $to, float $amount): void
{
    try {
        $from->decrement('balance', $amount); // This commits immediately
        $to->increment('balance', $amount);   // If this fails, money vanishes
    } catch (\Exception $e) {
        // Too late - the decrement is already committed
        Log::error('Transfer failed: ' . $e->getMessage());
    }
}
```

**Correct**

```php
use Illuminate\Support\Facades\DB;

// DB::transaction() closure - the cleanest approach
// Automatically commits on success, rolls back on any exception
public function placeOrder(Request $request): Order
{
    return DB::transaction(function () use ($request) {
        $order = Order::create([
            'user_id' => $request->user()->id,
            'total' => $request->total,
        ]);

        foreach ($request->items as $item) {
            $order->items()->create($item);
        }

        $order->user->decrement('balance', $request->total);

        Payment::create([
            'order_id' => $order->id,
            'amount' => $request->total,
            'status' => 'completed',
        ]);

        return $order;
    });
}

// Handling deadlocks with automatic retries (second argument = attempts)
public function transferFunds(Account $from, Account $to, float $amount): void
{
    DB::transaction(function () use ($from, $to, $amount) {
        // Lock the rows to prevent concurrent modification
        $from = Account::lockForUpdate()->find($from->id);
        $to = Account::lockForUpdate()->find($to->id);

        if ($from->balance < $amount) {
            throw new InsufficientFundsException('Insufficient balance.');
        }

        $from->decrement('balance', $amount);
        $to->increment('balance', $amount);

        TransferLog::create([
            'from_account_id' => $from->id,
            'to_account_id' => $to->id,
            'amount' => $amount,
        ]);
    }, attempts: 3); // Retry up to 3 times on deadlock
}

// Manual transaction control for complex flows where you need
// fine-grained control over when to commit or roll back
public function importUsers(array $records): ImportResult
{
    $result = new ImportResult();

    DB::beginTransaction();

    try {
        foreach ($records as $record) {
            $user = User::create($record);
            $user->assignRole('member');
            $result->addSuccess($user);
        }

        DB::commit();
    } catch (\Exception $e) {
        DB::rollBack();
        $result->setError($e->getMessage());
    }

    return $result;
}

// Nested transactions with savepoints
// Laravel automatically uses savepoints for nested transaction calls
public function processTeamSignup(array $data): Team
{
    return DB::transaction(function () use ($data) {
        $team = Team::create(['name' => $data['team_name']]);

        // This creates a savepoint - if it fails, only this inner
        // transaction rolls back, not the outer one
        try {
            DB::transaction(function () use ($team, $data) {
                foreach ($data['members'] as $memberData) {
                    $user = User::create($memberData);
                    $team->members()->attach($user);
                }
            });
        } catch (\Exception $e) {
            // Inner transaction rolled back to savepoint.
            // The team is still created; we can handle members later.
            Log::warning("Member import failed for team {$team->id}: {$e->getMessage()}");
            $team->update(['member_import_status' => 'failed']);
        }

        return $team;
    });
}

// Transaction events for cache invalidation and side effects
// These ensure side effects only run when the transaction actually commits
use Illuminate\Support\Facades\DB;

public function updateProduct(Product $product, array $data): Product
{
    return DB::transaction(function () use ($product, $data) {
        $product->update($data);
        $product->variants()->upsert($data['variants'], ['sku'], ['price', 'stock']);

        // This callback only fires if the transaction commits successfully.
        // If the transaction rolls back, the cache is never cleared,
        // which is exactly what you want.
        DB::afterCommit(function () use ($product) {
            Cache::tags(['products'])->forget("product:{$product->id}");
            Cache::tags(['products'])->forget('product:listing');
            event(new ProductUpdated($product));
        });

        return $product;
    });
}

// Combining transactions with queue dispatches
// Use afterCommit on jobs to ensure they only dispatch after the transaction commits
public function createInvoice(Order $order): Invoice
{
    return DB::transaction(function () use ($order) {
        $invoice = Invoice::create([
            'order_id' => $order->id,
            'amount' => $order->total,
            'status' => 'pending',
        ]);

        $order->update(['invoiced_at' => now()]);

        // afterCommit() ensures the job is only dispatched after the
        // transaction commits. Without this, the job might run before
        // the invoice row is visible to other database connections.
        SendInvoiceEmail::dispatch($invoice)->afterCommit();

        return $invoice;
    });
}
```

**Reference:** [Laravel Database Transactions](https://laravel.com/docs/database#database-transactions)

---

## 8. API Design

**Section Impact: MEDIUM**

### 8.1 Use API Resources for Response Serialization

**Impact: MEDIUM-HIGH** — Inconsistent API responses break client applications

Returning raw Eloquent models or hand-crafted arrays from controllers leads to inconsistent response structures, accidental exposure of sensitive fields, and tightly coupled clients. Laravel API Resources provide a dedicated transformation layer between your models and the JSON responses delivered to consumers. They give you full control over which fields are included, how relationships are nested, and how pagination metadata is shaped.

**Incorrect**

```php
// Returning raw models exposes all attributes including hidden internals
class UserController extends Controller
{
    public function index()
    {
        // Leaks database columns, timestamps, pivot data, and any appended attributes
        return User::all();
    }

    public function show(User $user)
    {
        // Hand-crafted arrays are fragile and inconsistent across endpoints
        return response()->json([
            'id' => $user->id,
            'name' => $user->name,
            'email' => $user->email,
            'created' => $user->created_at->format('Y-m-d'),
            // Easy to forget fields or format them differently elsewhere
        ]);
    }

    public function store(Request $request)
    {
        $user = User::create($request->all());

        // Different endpoints return different shapes for the same entity
        return response()->json([
            'success' => true,
            'data' => $user->toArray(),
        ], 201);
    }
}
```

**Correct**

```php
// app/Http/Resources/UserResource.php
namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'avatar_url' => $this->avatar_url,
            'created_at' => $this->created_at->toIso8601String(),

            // Conditional attributes - only included when explicitly loaded
            'posts_count' => $this->when($this->posts_count !== null, $this->posts_count),
            'email_verified' => $this->when($request->user()?->isAdmin(), $this->hasVerifiedEmail()),

            // Merge a set of attributes conditionally
            $this->mergeWhen($request->user()?->is($this->resource), [
                'phone' => $this->phone,
                'two_factor_enabled' => $this->two_factor_confirmed_at !== null,
            ]),

            // Conditional relationships - only serialized when loaded on the model
            'posts' => PostResource::collection($this->whenLoaded('posts')),
            'role' => new RoleResource($this->whenLoaded('role')),

            // Links for HATEOAS-style responses
            'links' => [
                'self' => route('api.users.show', $this->id),
                'posts' => route('api.users.posts.index', $this->id),
            ],
        ];
    }
}

// app/Http/Resources/UserCollection.php
namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\ResourceCollection;

class UserCollection extends ResourceCollection
{
    public $collects = UserResource::class;

    public function toArray(Request $request): array
    {
        return [
            'data' => $this->collection,
            'meta' => [
                'total_admins' => $this->collection->where('is_admin', true)->count(),
            ],
        ];
    }
}

// app/Http/Controllers/Api/UserController.php
namespace App\Http\Controllers\Api;

use App\Http\Resources\UserCollection;
use App\Http\Resources\UserResource;
use App\Models\User;

class UserController extends Controller
{
    public function index()
    {
        $users = User::with('role')
            ->withCount('posts')
            ->paginate(20);

        // ResourceCollection wraps paginated results automatically
        return new UserCollection($users);
    }

    public function show(User $user)
    {
        $user->loadMissing(['role', 'posts' => fn ($q) => $q->latest()->limit(5)]);

        return new UserResource($user);
    }

    public function store(StoreUserRequest $request)
    {
        $user = User::create($request->validated());

        // Consistent shape with proper status code
        return (new UserResource($user))
            ->response()
            ->setStatusCode(201);
    }
}
```

Reference: [Laravel API Resources](https://laravel.com/docs/eloquent-resources)

---

### 8.2 Use Middleware for Cross-Cutting Concerns

**Impact: MEDIUM** — Duplicated logic across controllers increases bugs

Cross-cutting concerns such as logging, response formatting, locale detection, and content-type enforcement should not be scattered across individual controllers. Duplicating this logic violates DRY, makes it easy to miss an endpoint, and turns every new controller into a copy-paste exercise. Laravel Middleware provides a clean pipeline for intercepting requests and responses in a centralized, testable, and composable way.

**Incorrect**

```php
// Every controller duplicates the same boilerplate logic
class OrderController extends Controller
{
    public function index(Request $request)
    {
        // Duplicated logging in every method
        Log::info('API request', [
            'method' => $request->method(),
            'path' => $request->path(),
            'user' => $request->user()?->id,
            'ip' => $request->ip(),
        ]);

        // Duplicated locale detection
        $locale = $request->header('Accept-Language', 'en');
        app()->setLocale(substr($locale, 0, 2));

        // Duplicated JSON enforcement
        if (!$request->expectsJson()) {
            return response()->json(['error' => 'JSON required'], 406);
        }

        $orders = Order::paginate(20);

        $duration = microtime(true) - LARAVEL_START;
        Log::info('API response', ['duration_ms' => $duration * 1000]);

        // Duplicated response wrapping
        return response()->json([
            'success' => true,
            'data' => $orders->items(),
            'meta' => ['page' => $orders->currentPage()],
        ]);
    }

    // Same boilerplate repeated in store(), show(), update(), destroy()...
}
```

**Correct**

```php
// app/Http/Middleware/ForceJsonResponse.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class ForceJsonResponse
{
    /**
     * Ensure every API request accepts and returns JSON.
     */
    public function handle(Request $request, Closure $next): Response
    {
        // Force the Accept header so exception handler renders JSON
        $request->headers->set('Accept', 'application/json');

        $response = $next($request);

        // Guarantee content-type on outbound responses
        $response->headers->set('Content-Type', 'application/json');

        return $response;
    }
}

// app/Http/Middleware/ApiRequestLogger.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Symfony\Component\HttpFoundation\Response;

class ApiRequestLogger
{
    /**
     * Log incoming request and outgoing response as a terminable middleware.
     * The terminate() method runs after the response has been sent to the client,
     * so it does not add latency.
     */
    public function handle(Request $request, Closure $next): Response
    {
        $request->attributes->set('api_start_time', microtime(true));

        return $next($request);
    }

    public function terminate(Request $request, Response $response): void
    {
        $duration = microtime(true) - $request->attributes->get('api_start_time', microtime(true));

        Log::channel('api')->info('API call', [
            'method' => $request->method(),
            'path' => $request->path(),
            'status' => $response->getStatusCode(),
            'duration_ms' => round($duration * 1000, 2),
            'user_id' => $request->user()?->id,
            'ip' => $request->ip(),
        ]);
    }
}

// app/Http/Middleware/SetLocaleFromHeader.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class SetLocaleFromHeader
{
    private const SUPPORTED_LOCALES = ['en', 'es', 'fr', 'pt', 'de'];

    public function handle(Request $request, Closure $next): Response
    {
        $preferred = substr($request->header('Accept-Language', 'en'), 0, 2);

        $locale = in_array($preferred, self::SUPPORTED_LOCALES, true)
            ? $preferred
            : 'en';

        app()->setLocale($locale);

        return $next($request);
    }
}

// app/Http/Middleware/WrapApiResponse.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class WrapApiResponse
{
    /**
     * Wrap successful JSON responses in a consistent envelope.
     */
    public function handle(Request $request, Closure $next): Response
    {
        $response = $next($request);

        if (!$response instanceof JsonResponse || $response->getStatusCode() >= 400) {
            return $response;
        }

        $original = $response->getData(true);

        // Don't double-wrap if already wrapped (e.g. from a ResourceCollection)
        if (isset($original['data'])) {
            return $response;
        }

        $response->setData([
            'success' => true,
            'data' => $original,
        ]);

        return $response;
    }
}

// bootstrap/app.php (Laravel 11+)
use App\Http\Middleware\ApiRequestLogger;
use App\Http\Middleware\ForceJsonResponse;
use App\Http\Middleware\SetLocaleFromHeader;
use App\Http\Middleware\WrapApiResponse;

return Application::configure(basePath: dirname(__DIR__))
    ->withMiddleware(function (Middleware $middleware) {
        $middleware->api(prepend: [
            ForceJsonResponse::class,
            SetLocaleFromHeader::class,
        ]);

        $middleware->api(append: [
            WrapApiResponse::class,
        ]);

        // Terminable middleware - runs after response is sent
        $middleware->append(ApiRequestLogger::class);

        // Set priority so ForceJsonResponse always runs first
        $middleware->priority([
            ForceJsonResponse::class,
            \Illuminate\Auth\Middleware\Authenticate::class,
            SetLocaleFromHeader::class,
        ]);
    })
    ->create();

// Controller is now clean and focused on business logic
class OrderController extends Controller
{
    public function index()
    {
        return OrderResource::collection(
            Order::with('customer')->paginate(20)
        );
    }

    public function store(StoreOrderRequest $request)
    {
        $order = Order::create($request->validated());

        return (new OrderResource($order))
            ->response()
            ->setStatusCode(201);
    }
}
```

Reference: [Laravel Middleware](https://laravel.com/docs/middleware)

---

### 8.3 Use Form Requests for Input Transformation

**Impact: MEDIUM** — Inconsistent input handling causes subtle bugs

Performing manual validation and ad-hoc input transformation inside controllers creates inconsistent handling, duplicated logic, and subtle bugs when the same rules are applied differently across endpoints. Laravel Form Requests centralize validation rules, authorization, input sanitization, and post-validation processing into a single, reusable object that is automatically resolved before your controller method executes.

**Incorrect**

```php
class ProductController extends Controller
{
    public function store(Request $request)
    {
        // Manual validation clutters the controller
        $validator = Validator::make($request->all(), [
            'name' => 'required|string|max:255',
            'price' => 'required|numeric|min:0',
            'sku' => 'required|string|unique:products',
            'tags' => 'sometimes|string',
            'slug' => 'sometimes|string',
        ]);

        if ($validator->fails()) {
            return response()->json(['errors' => $validator->errors()], 422);
        }

        $data = $validator->validated();

        // Scattered transformation logic - easy to forget in other endpoints
        $data['name'] = trim($data['name']);
        $data['sku'] = strtoupper(trim($data['sku']));
        $data['slug'] = $data['slug'] ?? Str::slug($data['name']);
        $data['price'] = (int) round($data['price'] * 100); // convert to cents
        $data['tags'] = isset($data['tags'])
            ? array_map('trim', explode(',', $data['tags']))
            : [];

        $product = Product::create($data);

        return response()->json($product, 201);
    }

    public function update(Request $request, Product $product)
    {
        // Same validation and transformation duplicated here, possibly with drift
        $data = $request->validate([
            'name' => 'sometimes|string|max:255',
            'price' => 'sometimes|numeric|min:0',
            'sku' => 'sometimes|string|unique:products,sku,' . $product->id,
            'tags' => 'sometimes|string',
        ]);

        // Oops - forgot to uppercase the SKU here, causing inconsistency
        $data['price'] = isset($data['price'])
            ? (int) round($data['price'] * 100)
            : $product->price;

        $product->update($data);

        return response()->json($product);
    }
}
```

**Correct**

```php
// app/Http/Requests/StoreProductRequest.php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Support\Str;

class StoreProductRequest extends FormRequest
{
    /**
     * Gate authorization to the request level.
     */
    public function authorize(): bool
    {
        return $this->user()->can('create', Product::class);
    }

    /**
     * Sanitize and normalize input before validation runs.
     * This is the place for trimming, case normalization, and format coercion.
     */
    protected function prepareForValidation(): void
    {
        $this->merge([
            'name' => trim($this->input('name', '')),
            'sku' => strtoupper(trim($this->input('sku', ''))),
            'slug' => $this->input('slug') ?: Str::slug($this->input('name', '')),

            // Normalize comma-separated string into array before the 'array' rule
            'tags' => is_string($this->input('tags'))
                ? array_filter(array_map('trim', explode(',', $this->input('tags'))))
                : ($this->input('tags') ?? []),
        ]);
    }

    /**
     * Validation rules applied to the already-sanitized input.
     */
    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'price' => ['required', 'numeric', 'min:0'],
            'sku' => ['required', 'string', 'max:64', 'unique:products,sku'],
            'slug' => ['required', 'string', 'max:255', 'unique:products,slug'],
            'tags' => ['sometimes', 'array'],
            'tags.*' => ['string', 'max:50'],
            'category_id' => ['required', 'exists:categories,id'],
        ];
    }

    /**
     * Transform validated data after validation passes.
     * Use this for business-level conversions like currency to cents.
     */
    protected function passedValidation(): void
    {
        $this->replace(
            array_merge($this->validated(), [
                // Store price in cents to avoid floating point issues
                'price' => (int) round($this->validated('price') * 100),
            ])
        );
    }

    /**
     * Custom attribute names for clearer error messages.
     */
    public function attributes(): array
    {
        return [
            'sku' => 'product SKU',
            'category_id' => 'category',
        ];
    }

    /**
     * Custom error messages for specific rules.
     */
    public function messages(): array
    {
        return [
            'sku.unique' => 'This SKU is already assigned to another product.',
            'price.min' => 'Price cannot be negative.',
        ];
    }
}

// app/Http/Requests/UpdateProductRequest.php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Support\Str;

class UpdateProductRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('update', $this->route('product'));
    }

    protected function prepareForValidation(): void
    {
        $merged = [];

        if ($this->has('name')) {
            $merged['name'] = trim($this->input('name'));
        }
        if ($this->has('sku')) {
            $merged['sku'] = strtoupper(trim($this->input('sku')));
        }
        if ($this->has('name') && !$this->has('slug')) {
            $merged['slug'] = Str::slug($this->input('name'));
        }
        if ($this->has('tags') && is_string($this->input('tags'))) {
            $merged['tags'] = array_filter(array_map('trim', explode(',', $this->input('tags'))));
        }

        $this->merge($merged);
    }

    public function rules(): array
    {
        $product = $this->route('product');

        return [
            'name' => ['sometimes', 'string', 'max:255'],
            'price' => ['sometimes', 'numeric', 'min:0'],
            'sku' => ['sometimes', 'string', 'max:64', "unique:products,sku,{$product->id}"],
            'slug' => ['sometimes', 'string', 'max:255', "unique:products,slug,{$product->id}"],
            'tags' => ['sometimes', 'array'],
            'tags.*' => ['string', 'max:50'],
            'category_id' => ['sometimes', 'exists:categories,id'],
        ];
    }

    protected function passedValidation(): void
    {
        if ($this->has('price')) {
            $this->replace(
                array_merge($this->validated(), [
                    'price' => (int) round($this->validated('price') * 100),
                ])
            );
        }
    }
}

// app/Http/Controllers/Api/ProductController.php
namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Requests\StoreProductRequest;
use App\Http\Requests\UpdateProductRequest;
use App\Http\Resources\ProductResource;
use App\Models\Product;

class ProductController extends Controller
{
    /**
     * Controller stays thin - all validation and transformation
     * has already happened by the time these methods run.
     */
    public function store(StoreProductRequest $request)
    {
        $product = Product::create($request->validated());

        if ($tags = $request->validated('tags')) {
            $product->tags()->sync($tags);
        }

        return (new ProductResource($product->load('category', 'tags')))
            ->response()
            ->setStatusCode(201);
    }

    public function update(UpdateProductRequest $request, Product $product)
    {
        $product->update($request->validated());

        if ($request->has('tags')) {
            $product->tags()->sync($request->validated('tags'));
        }

        return new ProductResource($product->fresh('category', 'tags'));
    }
}
```

Reference: [Laravel Form Requests](https://laravel.com/docs/validation#form-request-validation)

---

### 8.4 Use API Versioning for Breaking Changes

**Impact: MEDIUM** — Unversioned APIs break existing clients on updates

Without versioning, every structural change to your API risks breaking existing consumers. Mobile apps, third-party integrations, and SPAs all depend on a stable contract. API versioning lets you evolve your endpoints while giving clients time to migrate. URL prefix versioning is the most common and most explicit strategy in the Laravel ecosystem.

**Incorrect**

```php
// routes/api.php - single unversioned route file
// Any breaking change here immediately affects every client
use App\Http\Controllers\UserController;
use App\Http\Controllers\OrderController;

Route::apiResource('users', UserController::class);
Route::apiResource('orders', OrderController::class);
Route::get('users/{user}/orders', [UserController::class, 'orders']);

// When you need to rename a field, change a response shape, or remove
// an endpoint, every consumer breaks simultaneously with no migration path.
```

**Correct**

```php
// routes/api_v1.php
use App\Http\Controllers\Api\V1\UserController;
use App\Http\Controllers\Api\V1\OrderController;

Route::prefix('v1')->as('v1.')->group(function () {
    Route::apiResource('users', UserController::class);
    Route::apiResource('orders', OrderController::class);
    Route::get('users/{user}/orders', [UserController::class, 'orders']);
});

// routes/api_v2.php
use App\Http\Controllers\Api\V2\UserController;
use App\Http\Controllers\Api\V2\OrderController;

Route::prefix('v2')->as('v2.')->group(function () {
    Route::apiResource('users', UserController::class);
    Route::apiResource('orders', OrderController::class);
    // V2 introduces a dedicated nested resource instead of a custom route
    Route::apiResource('users.orders', \App\Http\Controllers\Api\V2\UserOrderController::class)
        ->shallow();
});

// bootstrap/app.php (Laravel 11+)
return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        api: __DIR__ . '/../routes/api.php',
        then: function () {
            Route::middleware('api')
                ->prefix('api')
                ->group(base_path('routes/api_v1.php'));

            Route::middleware('api')
                ->prefix('api')
                ->group(base_path('routes/api_v2.php'));
        },
    )
    ->create();

// app/Http/Controllers/Api/V1/UserController.php
namespace App\Http\Controllers\Api\V1;

use App\Http\Controllers\Controller;
use App\Http\Resources\V1\UserResource;
use App\Models\User;

class UserController extends Controller
{
    public function show(User $user)
    {
        return new UserResource($user->load('role'));
    }
}

// app/Http/Resources/V1/UserResource.php
namespace App\Http\Resources\V1;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,           // V1: single name field
            'email' => $this->email,
            'role' => $this->whenLoaded('role', fn () => $this->role->name),
            'created_at' => $this->created_at->toIso8601String(),
        ];
    }
}

// app/Http/Controllers/Api/V2/UserController.php
namespace App\Http\Controllers\Api\V2;

use App\Http\Controllers\Controller;
use App\Http\Resources\V2\UserResource;
use App\Models\User;

class UserController extends Controller
{
    public function show(User $user)
    {
        return new UserResource($user->load('role'));
    }
}

// app/Http/Resources/V2/UserResource.php
namespace App\Http\Resources\V2;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'first_name' => $this->first_name,  // V2: split name into parts
            'last_name' => $this->last_name,
            'email' => $this->email,
            'role' => new RoleResource($this->whenLoaded('role')),
            'created_at' => $this->created_at->toIso8601String(),
            'updated_at' => $this->updated_at->toIso8601String(),
        ];
    }
}

// --- Header-based versioning alternative ---
// app/Http/Middleware/ApiVersionFromHeader.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class ApiVersionFromHeader
{
    /**
     * Resolve API version from Accept header.
     * Example: Accept: application/vnd.myapp.v2+json
     */
    public function handle(Request $request, Closure $next): Response
    {
        $accept = $request->header('Accept', '');

        preg_match('/application\/vnd\.myapp\.v(\d+)\+json/', $accept, $matches);

        $version = isset($matches[1]) ? (int) $matches[1] : 1;

        $request->attributes->set('api_version', $version);

        return $next($request);
    }
}

// --- Deprecation middleware for sunsetting old versions ---
// app/Http/Middleware/DeprecateApiVersion.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class DeprecateApiVersion
{
    /**
     * Add deprecation headers to warn clients about upcoming removal.
     * Apply to old version route groups: ->middleware(DeprecateApiVersion::class.':2026-06-01')
     */
    public function handle(Request $request, Closure $next, string $sunsetDate): Response
    {
        $response = $next($request);

        $response->headers->set('Deprecation', 'true');
        $response->headers->set('Sunset', $sunsetDate);
        $response->headers->set(
            'Link',
            '<https://api.example.com/docs/migration>; rel="deprecation"; type="text/html"'
        );

        return $response;
    }
}

// Apply deprecation to V1 routes
// routes/api_v1.php
Route::prefix('v1')
    ->as('v1.')
    ->middleware(DeprecateApiVersion::class . ':2026-06-01')
    ->group(function () {
        Route::apiResource('users', V1\UserController::class);
    });
```

Reference: [Laravel Routing](https://laravel.com/docs/routing)

---

## 9. Microservices

**Section Impact: MEDIUM**

### 9.1 Implement Health Checks for Microservices

**Impact: MEDIUM** — Without health checks, orchestrators can't manage services

Every microservice should expose health check endpoints so that container orchestrators (Kubernetes, Docker Swarm) and load balancers can determine whether the service is alive and ready to accept traffic. Without proper health checks, failed services continue receiving requests, leading to cascading failures and poor user experience. Separate liveness probes (is the process running?) from readiness probes (can it handle requests?) for fine-grained control.

**Incorrect (no health checks or naive implementation):**

```php
// routes/web.php - A trivial health check that tells you nothing
Route::get('/health', function () {
    return response()->json(['status' => 'ok']);
});

// This always returns 200 even when:
// - The database connection is down
// - Redis is unreachable
// - The disk is full
// - Queue workers have stopped processing
// The orchestrator thinks the service is healthy when it isn't.
```

**Correct (comprehensive health check with liveness and readiness probes):**

```php
// app/Services/HealthCheck/HealthCheckService.php
namespace App\Services\HealthCheck;

use App\Services\HealthCheck\Checkers\HealthCheckerInterface;

class HealthCheckService
{
    /** @var array<string, HealthCheckerInterface> */
    private array $checkers = [];

    public function registerChecker(string $name, HealthCheckerInterface $checker): void
    {
        $this->checkers[$name] = $checker;
    }

    /**
     * Run all registered health checks.
     *
     * @return array{healthy: bool, checks: array<string, array{status: string, message: string, duration_ms: float}>}
     */
    public function check(): array
    {
        $results = [];
        $allHealthy = true;

        foreach ($this->checkers as $name => $checker) {
            $start = microtime(true);

            try {
                $result = $checker->check();
                $results[$name] = [
                    'status' => $result->healthy ? 'pass' : 'fail',
                    'message' => $result->message,
                    'duration_ms' => round((microtime(true) - $start) * 1000, 2),
                ];

                if (! $result->healthy) {
                    $allHealthy = false;
                }
            } catch (\Throwable $e) {
                $allHealthy = false;
                $results[$name] = [
                    'status' => 'fail',
                    'message' => $e->getMessage(),
                    'duration_ms' => round((microtime(true) - $start) * 1000, 2),
                ];
            }
        }

        return [
            'healthy' => $allHealthy,
            'checks' => $results,
        ];
    }
}

// app/Services/HealthCheck/Checkers/HealthCheckerInterface.php
namespace App\Services\HealthCheck\Checkers;

class CheckResult
{
    public function __construct(
        public readonly bool $healthy,
        public readonly string $message,
    ) {}
}

interface HealthCheckerInterface
{
    public function check(): CheckResult;
}

// app/Services/HealthCheck/Checkers/DatabaseChecker.php
namespace App\Services\HealthCheck\Checkers;

use Illuminate\Support\Facades\DB;

class DatabaseChecker implements HealthCheckerInterface
{
    public function check(): CheckResult
    {
        try {
            DB::connection()->getPdo();
            DB::select('SELECT 1');

            return new CheckResult(true, 'Database connection is active.');
        } catch (\Throwable $e) {
            return new CheckResult(false, 'Database unreachable: ' . $e->getMessage());
        }
    }
}

// app/Services/HealthCheck/Checkers/RedisChecker.php
namespace App\Services\HealthCheck\Checkers;

use Illuminate\Support\Facades\Redis;

class RedisChecker implements HealthCheckerInterface
{
    public function check(): CheckResult
    {
        try {
            $response = Redis::ping();

            if ($response === true || $response === 'PONG' || (string) $response === '+PONG') {
                return new CheckResult(true, 'Redis connection is active.');
            }

            return new CheckResult(false, 'Redis returned unexpected response.');
        } catch (\Throwable $e) {
            return new CheckResult(false, 'Redis unreachable: ' . $e->getMessage());
        }
    }
}

// app/Services/HealthCheck/Checkers/QueueChecker.php
namespace App\Services\HealthCheck\Checkers;

use Illuminate\Support\Facades\Cache;

class QueueChecker implements HealthCheckerInterface
{
    /**
     * Check if queue workers are processing jobs by verifying
     * that a heartbeat key was recently set by a scheduled task.
     */
    public function check(): CheckResult
    {
        $lastHeartbeat = Cache::get('queue:worker:heartbeat');

        if ($lastHeartbeat === null) {
            return new CheckResult(false, 'No queue worker heartbeat detected.');
        }

        $secondsAgo = now()->diffInSeconds($lastHeartbeat);

        if ($secondsAgo > 120) {
            return new CheckResult(false, "Queue worker heartbeat is {$secondsAgo}s old (threshold: 120s).");
        }

        return new CheckResult(true, "Queue worker active, last heartbeat {$secondsAgo}s ago.");
    }
}

// app/Services/HealthCheck/Checkers/DiskSpaceChecker.php
namespace App\Services\HealthCheck\Checkers;

class DiskSpaceChecker implements HealthCheckerInterface
{
    public function __construct(
        private readonly float $thresholdPercent = 90.0,
    ) {}

    public function check(): CheckResult
    {
        $storagePath = storage_path();
        $totalSpace = disk_total_space($storagePath);
        $freeSpace = disk_free_space($storagePath);

        if ($totalSpace === false || $freeSpace === false) {
            return new CheckResult(false, 'Unable to determine disk space.');
        }

        $usedPercent = round((($totalSpace - $freeSpace) / $totalSpace) * 100, 2);

        if ($usedPercent >= $this->thresholdPercent) {
            return new CheckResult(false, "Disk usage at {$usedPercent}% (threshold: {$this->thresholdPercent}%).");
        }

        return new CheckResult(true, "Disk usage at {$usedPercent}%.");
    }
}

// app/Providers/HealthCheckServiceProvider.php
namespace App\Providers;

use App\Services\HealthCheck\Checkers\DatabaseChecker;
use App\Services\HealthCheck\Checkers\DiskSpaceChecker;
use App\Services\HealthCheck\Checkers\QueueChecker;
use App\Services\HealthCheck\Checkers\RedisChecker;
use App\Services\HealthCheck\HealthCheckService;
use Illuminate\Support\ServiceProvider;

class HealthCheckServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->singleton(HealthCheckService::class, function () {
            $service = new HealthCheckService();
            $service->registerChecker('database', new DatabaseChecker());
            $service->registerChecker('redis', new RedisChecker());
            $service->registerChecker('queue', new QueueChecker());
            $service->registerChecker('disk', new DiskSpaceChecker(thresholdPercent: 90.0));

            return $service;
        });
    }
}

// routes/api.php - Separate liveness and readiness probes
use App\Services\HealthCheck\HealthCheckService;

// Liveness probe: is the PHP process running?
// Kubernetes uses this to decide whether to restart the container.
Route::get('/health/live', function () {
    return response()->json([
        'status' => 'alive',
        'timestamp' => now()->toIso8601String(),
    ]);
});

// Readiness probe: can the service handle traffic?
// Kubernetes uses this to decide whether to route traffic to the pod.
Route::get('/health/ready', function (HealthCheckService $health) {
    $result = $health->check();

    return response()->json([
        'status' => $result['healthy'] ? 'ready' : 'not_ready',
        'timestamp' => now()->toIso8601String(),
        'checks' => $result['checks'],
    ], $result['healthy'] ? 200 : 503);
});

// Full health check for internal monitoring dashboards
Route::get('/health', function (HealthCheckService $health) {
    $result = $health->check();

    return response()->json([
        'status' => $result['healthy'] ? 'healthy' : 'unhealthy',
        'service' => config('app.name'),
        'version' => config('app.version', '1.0.0'),
        'timestamp' => now()->toIso8601String(),
        'checks' => $result['checks'],
    ], $result['healthy'] ? 200 : 503);
})->middleware('auth:api-internal');
```

Reference: [Laravel Documentation](https://laravel.com/docs)

---

### 9.2 Use Message and Event Patterns Correctly

**Impact: MEDIUM** — Wrong patterns cause lost messages and tight coupling

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

---

### 9.3 Use Message Queues for Background Jobs

**Impact: MEDIUM-HIGH** — Synchronous heavy processing blocks user requests

Heavy or time-consuming operations such as sending emails, generating reports, processing images, or calling external APIs should never run synchronously inside a controller. This blocks the HTTP response, increases server resource usage, and degrades user experience. Use Laravel Queues with dedicated Job classes to offload work to background workers. Combine this with Laravel Horizon for monitoring, job chaining for sequential workflows, batching for parallel tasks, and job middleware for deduplication and throttling.

**Incorrect (synchronous heavy processing in the controller):**

```php
// app/Http/Controllers/ReportController.php
namespace App\Http\Controllers;

use App\Mail\ReportReady;
use App\Models\Report;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Mail;
use Barryvdh\DomPDF\Facade\Pdf;

class ReportController extends Controller
{
    public function generate(Request $request)
    {
        // Heavy processing directly in the controller - blocks the request
        $data = Report::query()
            ->with('entries', 'entries.transactions')
            ->whereBetween('created_at', [
                $request->input('start_date'),
                $request->input('end_date'),
            ])
            ->get();

        // CPU-intensive PDF generation - can take 30+ seconds
        $pdf = Pdf::loadView('reports.monthly', ['data' => $data]);
        $path = storage_path("app/reports/report-{$request->user()->id}.pdf");
        $pdf->save($path);

        // Sending email synchronously - waits for SMTP response
        Mail::to($request->user())->send(new ReportReady($path));

        // User has been waiting 30-60 seconds for this response
        return response()->json(['message' => 'Report generated and emailed.']);
    }
}
```

**Correct (queued jobs with chaining, batching, rate limiting, and Horizon monitoring):**

```php
// app/Jobs/GenerateReport.php
namespace App\Jobs;

use App\Models\Report;
use App\Models\User;
use Barryvdh\DomPDF\Facade\Pdf;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\Middleware\WithoutOverlapping;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Storage;

class GenerateReport implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;

    public int $timeout = 300; // 5 minutes max

    public int $backoff = 60;

    public function __construct(
        public readonly User $user,
        public readonly string $startDate,
        public readonly string $endDate,
        public readonly string $reportId,
    ) {}

    /**
     * Job middleware: prevent duplicate report generation for the same user.
     */
    public function middleware(): array
    {
        return [
            new WithoutOverlapping("report:{$this->user->id}"),
        ];
    }

    public function handle(): void
    {
        $data = Report::query()
            ->with('entries', 'entries.transactions')
            ->whereBetween('created_at', [$this->startDate, $this->endDate])
            ->get();

        $pdf = Pdf::loadView('reports.monthly', ['data' => $data]);
        $filename = "reports/{$this->reportId}.pdf";

        Storage::disk('s3')->put($filename, $pdf->output());

        // Update the report record with the file location
        Report::where('id', $this->reportId)->update([
            'status' => 'generated',
            'file_path' => $filename,
            'generated_at' => now(),
        ]);
    }

    public function failed(\Throwable $exception): void
    {
        Report::where('id', $this->reportId)->update([
            'status' => 'failed',
            'error' => $exception->getMessage(),
        ]);
    }
}

// app/Jobs/EmailReport.php
namespace App\Jobs;

use App\Mail\ReportReady;
use App\Models\Report;
use App\Models\User;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\Middleware\RateLimited;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Mail;

class EmailReport implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;

    public array $backoff = [30, 60, 120];

    public function __construct(
        public readonly User $user,
        public readonly string $reportId,
    ) {}

    /**
     * Throttle email sending to avoid hitting SMTP rate limits.
     */
    public function middleware(): array
    {
        return [
            new RateLimited('emails'),
        ];
    }

    public function handle(): void
    {
        $report = Report::findOrFail($this->reportId);

        Mail::to($this->user)->send(new ReportReady($report));

        $report->update(['emailed_at' => now()]);
    }
}

// app/Jobs/ProcessBulkReports.php - Batching for parallel work
namespace App\Jobs;

use App\Models\User;
use Illuminate\Bus\Batch;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Bus;

class ProcessBulkReports implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function handle(): void
    {
        $users = User::where('wants_monthly_report', true)->get();

        $jobs = $users->map(function (User $user) {
            $reportId = \Illuminate\Support\Str::uuid()->toString();

            return [
                new GenerateReport(
                    user: $user,
                    startDate: now()->subMonth()->startOfMonth()->toDateString(),
                    endDate: now()->subMonth()->endOfMonth()->toDateString(),
                    reportId: $reportId,
                ),
                new EmailReport(user: $user, reportId: $reportId),
            ];
        });

        // Bus::chain runs jobs sequentially per user
        // Bus::batch runs all user chains in parallel
        $chains = $jobs->map(fn (array $chain) => Bus::chain($chain));

        Bus::batch($chains->toArray())
            ->name('Monthly Reports - ' . now()->subMonth()->format('F Y'))
            ->onQueue('reports')
            ->allowFailures()
            ->then(function (Batch $batch) {
                // All jobs completed successfully
                \Illuminate\Support\Facades\Log::info('Bulk report batch completed.', [
                    'batch_id' => $batch->id,
                    'total' => $batch->totalJobs,
                ]);
            })
            ->catch(function (Batch $batch, \Throwable $e) {
                // First failure in the batch
                \Illuminate\Support\Facades\Log::warning('Bulk report batch had failures.', [
                    'batch_id' => $batch->id,
                    'failed' => $batch->failedJobs,
                ]);
            })
            ->dispatch();
    }
}

// app/Http/Controllers/ReportController.php
namespace App\Http\Controllers;

use App\Jobs\GenerateReport;
use App\Jobs\EmailReport;
use App\Models\Report;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Bus;
use Illuminate\Support\Str;

class ReportController extends Controller
{
    public function generate(Request $request)
    {
        $validated = $request->validate([
            'start_date' => ['required', 'date'],
            'end_date' => ['required', 'date', 'after:start_date'],
        ]);

        $reportId = Str::uuid()->toString();

        // Create a pending report record
        $report = Report::create([
            'id' => $reportId,
            'user_id' => $request->user()->id,
            'status' => 'pending',
            'start_date' => $validated['start_date'],
            'end_date' => $validated['end_date'],
        ]);

        // Chain jobs: generate first, then email
        Bus::chain([
            new GenerateReport(
                user: $request->user(),
                startDate: $validated['start_date'],
                endDate: $validated['end_date'],
                reportId: $reportId,
            ),
            new EmailReport(
                user: $request->user(),
                reportId: $reportId,
            ),
        ])->onQueue('reports')->dispatch();

        // Return immediately - the user can poll for status
        return response()->json([
            'message' => 'Report generation started.',
            'report_id' => $reportId,
            'status_url' => route('reports.status', $reportId),
        ], 202);
    }

    public function status(string $reportId)
    {
        $report = Report::where('id', $reportId)
            ->where('user_id', auth()->id())
            ->firstOrFail();

        return response()->json([
            'id' => $report->id,
            'status' => $report->status,
            'generated_at' => $report->generated_at,
            'emailed_at' => $report->emailed_at,
        ]);
    }
}

// app/Providers/AppServiceProvider.php - Rate limiter configuration
namespace App\Providers;

use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Rate limit for email sending jobs
        RateLimiter::for('emails', function (object $job) {
            return Limit::perMinute(30);
        });
    }
}

// config/horizon.php - Horizon configuration for monitoring and queue workers
return [
    'domain' => null,
    'path' => 'horizon',
    'use' => 'default',
    'prefix' => env('HORIZON_PREFIX', 'horizon:' . env('APP_NAME', 'laravel') . ':'),
    'middleware' => ['web', 'auth:admin'],

    'waits' => [
        'redis:default' => 60,
        'redis:reports' => 120,
    ],

    'trim' => [
        'recent' => 60,        // minutes
        'pending' => 60,
        'completed' => 60,
        'recent_failed' => 10080,  // 7 days
        'failed' => 10080,
        'monitored' => 10080,
    ],

    'environments' => [
        'production' => [
            'default-worker' => [
                'connection' => 'redis',
                'queue' => ['default'],
                'balance' => 'auto',
                'autoScalingStrategy' => 'time',
                'minProcesses' => 1,
                'maxProcesses' => 10,
                'balanceMaxShift' => 1,
                'balanceCooldown' => 3,
                'tries' => 3,
                'timeout' => 60,
            ],
            'reports-worker' => [
                'connection' => 'redis',
                'queue' => ['reports'],
                'balance' => 'auto',
                'autoScalingStrategy' => 'size',
                'minProcesses' => 1,
                'maxProcesses' => 5,
                'balanceMaxShift' => 1,
                'balanceCooldown' => 3,
                'tries' => 3,
                'timeout' => 600,  // Reports can take longer
            ],
            'notifications-worker' => [
                'connection' => 'redis',
                'queue' => ['notifications'],
                'balance' => 'simple',
                'processes' => 3,
                'tries' => 3,
                'timeout' => 30,
            ],
            'dead-letter-worker' => [
                'connection' => 'redis',
                'queue' => ['dead-letter'],
                'balance' => 'simple',
                'processes' => 1,
                'tries' => 1,
                'timeout' => 60,
            ],
        ],
    ],
];
```

Reference: [Laravel Queues Documentation](https://laravel.com/docs/queues)

---

## 10. DevOps & Deployment

**Section Impact: LOW-MEDIUM**

### 10.1 Implement Graceful Shutdown

**Impact: MEDIUM** — Ungraceful shutdown causes data loss in queue workers

Queue workers and long-running processes must be shut down gracefully to avoid losing jobs mid-execution. When a worker is killed abruptly with SIGKILL, any job currently being processed is lost -- it will not be retried because the worker never had a chance to release it back to the queue. Laravel provides built-in mechanisms for graceful restarts and zero-downtime deployments that prevent data loss during deployment cycles.

**Incorrect**

```php
// Deployment script that kills workers abruptly -- jobs in progress are lost
// deploy.sh
#!/bin/bash
git pull origin main
composer install --no-dev
php artisan migrate --force

# WRONG: SIGKILL gives workers zero time to finish current jobs
kill -9 $(pgrep -f "queue:work")

# WRONG: No maintenance mode, users see errors mid-deploy
php artisan config:cache
php artisan route:cache

# Workers restarted with no signal handling
php artisan queue:work &

// app/Console/Commands/ProcessReports.php
namespace App\Console\Commands;

use Illuminate\Console\Command;

class ProcessReports extends Command
{
    protected $signature = 'reports:process';

    public function handle()
    {
        // WRONG: No signal handling -- SIGTERM kills this mid-iteration
        // and partially processed data is left in an inconsistent state
        $reports = Report::where('status', 'pending')->get();

        foreach ($reports as $report) {
            // If killed here, report is partially processed with no way to recover
            $report->status = 'processing';
            $report->save();

            $this->generatePdf($report);
            $this->sendToClient($report);

            $report->status = 'completed';
            $report->save();
        }
    }
}

// Supervisor config with no graceful stop
// /etc/supervisor/conf.d/worker.conf
// [program:laravel-worker]
// command=php /var/www/artisan queue:work
// stopwaitsecs=1          ; Only 1 second before SIGKILL -- not enough
// stopsignal=KILL         ; WRONG: Should never use KILL as stop signal
```

**Correct**

```php
// deploy.sh -- Zero-downtime deployment with graceful shutdown
#!/bin/bash
set -e

echo "Starting zero-downtime deployment..."

# 1. Enter maintenance mode with retry header so load balancers can retry
php artisan down --retry=60 --refresh=15

# 2. Pull code and install dependencies
git pull origin main
composer install --no-dev --optimize-autoloader

# 3. Run migrations
php artisan migrate --force

# 4. Cache configuration, routes, and views
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache

# 5. Gracefully restart queue workers -- they finish current job then exit
php artisan queue:restart

# 6. If using Horizon, gracefully terminate (finishes current jobs)
php artisan horizon:terminate

# 7. Exit maintenance mode
php artisan up

echo "Deployment complete."

// app/Console/Commands/ProcessReports.php
namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\Log;

class ProcessReports extends Command
{
    protected $signature = 'reports:process';

    private bool $shouldStop = false;

    public function handle(): int
    {
        // Register signal handlers for graceful shutdown
        pcntl_async_signals(true);

        pcntl_signal(SIGTERM, function () {
            Log::info('ProcessReports received SIGTERM, finishing current report...');
            $this->shouldStop = true;
        });

        pcntl_signal(SIGINT, function () {
            Log::info('ProcessReports received SIGINT, finishing current report...');
            $this->shouldStop = true;
        });

        $reports = Report::where('status', 'pending')->get();

        foreach ($reports as $report) {
            // Check if we should stop before starting a new unit of work
            if ($this->shouldStop) {
                Log::info('ProcessReports stopping gracefully, remaining reports will be processed next run.');
                return self::SUCCESS;
            }

            // Wrap each report in a transaction for atomicity
            DB::transaction(function () use ($report) {
                $report->status = 'processing';
                $report->save();

                $this->generatePdf($report);
                $this->sendToClient($report);

                $report->status = 'completed';
                $report->save();
            });

            Log::info('Report processed successfully.', ['report_id' => $report->id]);
        }

        return self::SUCCESS;
    }
}

// app/Jobs/GenerateInvoice.php -- Queue job with proper timeout and retry
namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;

class GenerateInvoice implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    // Maximum number of attempts before marking as failed
    public int $tries = 3;

    // Timeout in seconds -- must be shorter than supervisor's stopwaitsecs
    public int $timeout = 120;

    // Retry after these many seconds on each attempt
    public function backoff(): array
    {
        return [30, 60, 120];
    }

    public function __construct(
        private readonly Invoice $invoice,
    ) {}

    public function handle(InvoiceService $service): void
    {
        Log::info('Generating invoice.', ['invoice_id' => $this->invoice->id]);

        $service->generate($this->invoice);
        $service->sendToCustomer($this->invoice);
    }

    // Called when all retry attempts are exhausted
    public function failed(\Throwable $exception): void
    {
        Log::error('Invoice generation failed permanently.', [
            'invoice_id' => $this->invoice->id,
            'error' => $exception->getMessage(),
        ]);

        $this->invoice->update(['status' => 'failed']);
    }
}

// Supervisor config with graceful shutdown settings
// /etc/supervisor/conf.d/worker.conf
//
// [program:laravel-worker]
// process_name=%(program_name)s_%(process_num)02d
// command=php /var/www/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600 --timeout=120
// autostart=true
// autorestart=true
// stopasgroup=true
// killasgroup=true
// numprocs=4
// stopsignal=TERM       ; Send SIGTERM first for graceful shutdown
// stopwaitsecs=130      ; Wait longer than job timeout before SIGKILL

// config/horizon.php -- Laravel Horizon graceful shutdown configuration
return [
    'environments' => [
        'production' => [
            'supervisor-1' => [
                'maxProcesses' => 10,
                'balanceMaxShift' => 1,
                'balanceCooldown' => 3,
                'tries' => 3,
                'timeout' => 120,
                // Horizon handles graceful shutdown automatically
                // Workers finish current job before terminating
            ],
        ],
    ],
];
```

Reference: [Laravel Queues](https://laravel.com/docs/queues), [Laravel Deployment](https://laravel.com/docs/deployment)

---

### 10.2 Use Config for Environment Configuration

**Impact: HIGH** — Direct env() calls outside config files break caching

Calling `env()` directly in application code (services, controllers, models) works during development but silently breaks in production when configuration is cached. Running `php artisan config:cache` compiles all config files into a single cached file and stops reading from `.env` entirely. Any `env()` call outside of config files will return `null` after caching, causing subtle and hard-to-diagnose failures. Always define environment values in config files and access them with `config()` throughout the application.

**Incorrect**

```php
// app/Services/PaymentService.php
namespace App\Services;

class PaymentService
{
    public function charge(Order $order): PaymentResult
    {
        // WRONG: env() returns null when config is cached
        $apiKey = env('STRIPE_SECRET_KEY');
        $webhookSecret = env('STRIPE_WEBHOOK_SECRET');

        $stripe = new \Stripe\StripeClient($apiKey);

        return $stripe->paymentIntents->create([
            'amount' => $order->total,
            'currency' => env('PAYMENT_CURRENCY', 'usd'), // Also breaks when cached
        ]);
    }
}

// app/Http/Controllers/NotificationController.php
namespace App\Http\Controllers;

class NotificationController extends Controller
{
    public function sendSlackAlert(string $message)
    {
        // WRONG: env() in a controller -- breaks with config caching
        $webhookUrl = env('SLACK_WEBHOOK_URL');
        $channel = env('SLACK_CHANNEL', '#general');

        Http::post($webhookUrl, [
            'channel' => $channel,
            'text' => $message,
        ]);
    }
}

// app/Models/User.php
namespace App\Models;

class User extends Authenticatable
{
    public function getAvatarUrlAttribute(): string
    {
        // WRONG: env() in a model accessor
        return env('CDN_URL') . '/avatars/' . $this->avatar_path;
    }
}

// routes/web.php
// WRONG: env() in route definitions
Route::get('/debug', function () {
    if (env('APP_DEBUG')) {  // Returns null when cached
        return response()->json(debug_info());
    }
    abort(404);
});
```

**Correct**

```php
// config/services.php -- Define all third-party service config here
return [
    'stripe' => [
        'key' => env('STRIPE_KEY'),
        'secret' => env('STRIPE_SECRET_KEY'),
        'webhook_secret' => env('STRIPE_WEBHOOK_SECRET'),
        'currency' => env('PAYMENT_CURRENCY', 'usd'),
    ],

    'slack' => [
        'webhook_url' => env('SLACK_WEBHOOK_URL'),
        'channel' => env('SLACK_CHANNEL', '#general'),
        'notifications_enabled' => env('SLACK_NOTIFICATIONS_ENABLED', true),
    ],
];

// config/cdn.php -- Custom config file for CDN settings
return [
    'url' => env('CDN_URL', 'https://cdn.example.com'),
    'assets_path' => env('CDN_ASSETS_PATH', '/assets'),
    'avatars_path' => env('CDN_AVATARS_PATH', '/avatars'),
];

// app/Services/PaymentService.php
namespace App\Services;

class PaymentService
{
    private readonly \Stripe\StripeClient $stripe;
    private readonly string $currency;

    public function __construct()
    {
        // CORRECT: config() works regardless of config caching
        $this->stripe = new \Stripe\StripeClient(config('services.stripe.secret'));
        $this->currency = config('services.stripe.currency');
    }

    public function charge(Order $order): PaymentResult
    {
        return $this->stripe->paymentIntents->create([
            'amount' => $order->total,
            'currency' => $this->currency,
        ]);
    }

    public function verifyWebhook(Request $request): bool
    {
        return \Stripe\Webhook::constructEvent(
            $request->getContent(),
            $request->header('Stripe-Signature'),
            config('services.stripe.webhook_secret'),
        );
    }
}

// app/Http/Controllers/NotificationController.php
namespace App\Http\Controllers;

use Illuminate\Support\Facades\Http;

class NotificationController extends Controller
{
    public function sendSlackAlert(string $message)
    {
        if (! config('services.slack.notifications_enabled')) {
            return;
        }

        Http::post(config('services.slack.webhook_url'), [
            'channel' => config('services.slack.channel'),
            'text' => $message,
        ]);
    }
}

// app/Models/User.php
namespace App\Models;

class User extends Authenticatable
{
    public function getAvatarUrlAttribute(): string
    {
        // CORRECT: config() reads from cached configuration
        return config('cdn.url') . config('cdn.avatars_path') . '/' . $this->avatar_path;
    }
}

// app/Providers/AppServiceProvider.php -- Validate critical config at boot
namespace App\Providers;

use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Validate required configuration in non-production to catch mistakes early
        if (! app()->isProduction()) {
            $required = [
                'services.stripe.secret',
                'services.stripe.webhook_secret',
                'services.slack.webhook_url',
            ];

            foreach ($required as $key) {
                if (empty(config($key))) {
                    throw new \RuntimeException("Missing required config: {$key}");
                }
            }
        }
    }
}

// Deployment: cache config for performance
// php artisan config:cache     -- Compiles all config into a single cached file
// php artisan config:clear     -- Removes the cached config file
// php artisan config:show      -- Displays the resolved config values (Laravel 11+)
```

Reference: [Laravel Configuration](https://laravel.com/docs/configuration)

---

### 10.3 Use Structured Logging

**Impact: MEDIUM** — Unstructured logs are impossible to search in production

Production logs must be structured and searchable. Using `error_log()`, `echo`, `dd()`, or string concatenation in log messages makes it nearly impossible to filter, aggregate, or alert on log data in tools like ELK, Datadog, or CloudWatch. Laravel's Log facade supports context arrays that produce structured key-value pairs, multiple channels for routing logs to different destinations, and contextual logging that attaches metadata to every log entry within a request lifecycle.

**Incorrect**

```php
// app/Services/OrderService.php
namespace App\Services;

class OrderService
{
    public function process(Order $order): void
    {
        // WRONG: error_log bypasses Laravel's logging entirely
        error_log("Processing order " . $order->id);

        // WRONG: echo/print pollute stdout and break JSON responses
        echo "Order total: " . $order->total;

        // WRONG: dd/dump left in code -- kills the request
        dump($order->toArray());

        try {
            $this->chargeCustomer($order);
        } catch (\Exception $e) {
            // WRONG: String concatenation -- impossible to parse or filter
            Log::error("Order failed for user " . $order->user_id . " with error: " . $e->getMessage() . " on order " . $order->id);

            // WRONG: Logging the full exception as a string loses stack trace structure
            Log::error("Exception: " . (string) $e);
        }

        // WRONG: No context, no way to correlate this log with a specific order
        Log::info("Order processed successfully");
    }
}

// config/logging.php -- Using only the default single channel
return [
    'default' => 'single',
    'channels' => [
        'single' => [
            'driver' => 'single',
            'path' => storage_path('logs/laravel.log'),
            // No rotation, no structured format, single monolithic file
        ],
    ],
];
```

**Correct**

```php
// config/logging.php -- Multiple channels with structured output
return [
    'default' => env('LOG_CHANNEL', 'stack'),

    'channels' => [
        // Stack sends to multiple channels simultaneously
        'stack' => [
            'driver' => 'stack',
            'channels' => ['daily', 'stderr'],
            'ignore_exceptions' => false,
        ],

        // Daily rotation prevents disk space issues
        'daily' => [
            'driver' => 'daily',
            'path' => storage_path('logs/laravel.log'),
            'level' => env('LOG_LEVEL', 'debug'),
            'days' => 14,
            'replace_placeholders' => true,
        ],

        // Stderr for container/cloud environments (Docker, Kubernetes)
        'stderr' => [
            'driver' => 'monolog',
            'level' => env('LOG_LEVEL', 'debug'),
            'handler' => StreamHandler::class,
            'with' => [
                'stream' => 'php://stderr',
            ],
            'formatter' => JsonFormatter::class,  // Structured JSON for log aggregators
        ],

        // Dedicated channel for payment events -- easy to monitor separately
        'payments' => [
            'driver' => 'daily',
            'path' => storage_path('logs/payments.log'),
            'level' => 'info',
            'days' => 30,
        ],

        // Slack alerts for critical errors only
        'slack' => [
            'driver' => 'slack',
            'url' => env('LOG_SLACK_WEBHOOK_URL'),
            'username' => 'Laravel Log',
            'emoji' => ':boom:',
            'level' => 'critical',
        ],

        // JSON channel for ELK/Datadog/CloudWatch ingestion
        'json' => [
            'driver' => 'monolog',
            'handler' => StreamHandler::class,
            'with' => [
                'stream' => storage_path('logs/app.json.log'),
            ],
            'formatter' => JsonFormatter::class,
            'formatter_with' => [
                'includeStacktraces' => true,
            ],
        ],
    ],
];

// app/Http/Middleware/AttachRequestContext.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Str;

class AttachRequestContext
{
    public function handle(Request $request, Closure $next)
    {
        // Attach context to every log entry for the duration of this request
        Log::withContext([
            'request_id' => $request->header('X-Request-ID', Str::uuid()->toString()),
            'user_id' => $request->user()?->id,
            'ip' => $request->ip(),
            'url' => $request->fullUrl(),
            'method' => $request->method(),
        ]);

        return $next($request);
    }
}

// app/Services/OrderService.php
namespace App\Services;

use Illuminate\Support\Facades\Log;

class OrderService
{
    public function process(Order $order): void
    {
        // CORRECT: Structured context as an array -- every field is searchable
        Log::info('Order processing started.', [
            'order_id' => $order->id,
            'user_id' => $order->user_id,
            'total' => $order->total,
            'items_count' => $order->items->count(),
        ]);

        try {
            $this->chargeCustomer($order);

            Log::info('Order payment charged successfully.', [
                'order_id' => $order->id,
                'payment_method' => $order->payment_method,
                'amount' => $order->total,
            ]);
        } catch (\Throwable $e) {
            // CORRECT: Pass exception as context -- preserves full stack trace
            Log::error('Order payment failed.', [
                'order_id' => $order->id,
                'user_id' => $order->user_id,
                'amount' => $order->total,
                'error' => $e->getMessage(),
                'exception' => $e,
            ]);

            // CORRECT: Route critical failures to a specific channel
            Log::channel('slack')->critical('Payment processing failure requires attention.', [
                'order_id' => $order->id,
                'error' => $e->getMessage(),
            ]);

            throw $e;
        }
    }
}

// app/Services/PaymentGateway.php -- Using a dedicated log channel
namespace App\Services;

use Illuminate\Support\Facades\Log;

class PaymentGateway
{
    private $log;

    public function __construct()
    {
        // Dedicated channel keeps payment logs separate and easy to audit
        $this->log = Log::channel('payments');
    }

    public function charge(string $customerId, int $amount): PaymentResult
    {
        $this->log->info('Initiating charge.', [
            'customer_id' => $customerId,
            'amount' => $amount,
            'currency' => config('services.stripe.currency'),
        ]);

        $result = $this->gateway->charge($customerId, $amount);

        $this->log->info('Charge completed.', [
            'customer_id' => $customerId,
            'transaction_id' => $result->transactionId,
            'status' => $result->status,
        ]);

        return $result;
    }
}

// app/Logging/CustomJsonFormatter.php -- Custom formatter for log aggregators
namespace App\Logging;

use Monolog\Formatter\JsonFormatter;
use Monolog\LogRecord;

class CustomJsonFormatter extends JsonFormatter
{
    public function format(LogRecord $record): string
    {
        $data = [
            'timestamp' => $record->datetime->format('c'),
            'level' => $record->level->getName(),
            'message' => $record->message,
            'context' => $record->context,
            'service' => config('app.name'),
            'environment' => config('app.env'),
            'hostname' => gethostname(),
        ];

        return json_encode($data, JSON_UNESCAPED_SLASHES) . "\n";
    }
}
```

Reference: [Laravel Logging](https://laravel.com/docs/logging)

---

## References

- [Laravel Documentation](https://laravel.com/docs)
- [Laravel API Reference](https://laravel.com/api)
- [Eloquent ORM](https://laravel.com/docs/eloquent)
- [Laravel Testing](https://laravel.com/docs/testing)
- [Laravel Security](https://laravel.com/docs/security)
- [Laravel Queues](https://laravel.com/docs/queues)
- [Laravel Sanctum](https://laravel.com/docs/sanctum)
- [Laravel Horizon](https://laravel.com/docs/horizon)

---

*Generated by build-agents.ts on 2026-03-31*
