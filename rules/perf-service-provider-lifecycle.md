---
title: Use Service Provider Lifecycle Correctly
impact: HIGH
impactDescription: "Wrong lifecycle usage slows application boot"
tags: [performance, service-providers, boot, register, lifecycle]
---

## Use Service Provider Lifecycle Correctly

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
