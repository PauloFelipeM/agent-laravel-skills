---
title: Avoid Circular Dependencies
impact: CRITICAL
impactDescription: "#1 cause of runtime crashes"
tags: [architecture, service-providers, dependencies]
---

## Avoid Circular Dependencies Between Service Providers

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
