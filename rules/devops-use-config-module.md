---
title: Use Config for Environment Configuration
impact: HIGH
impactDescription: Direct env() calls outside config files break caching
tags: devops, configuration, environment, config-caching, env
---

## Use Config for Environment Configuration

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
