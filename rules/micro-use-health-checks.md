---
title: Implement Health Checks for Microservices
impact: MEDIUM
impactDescription: Without health checks, orchestrators can't manage services
tags: microservices, health-checks, monitoring, kubernetes, load-balancer
---

## Implement Health Checks for Microservices

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
