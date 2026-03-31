---
title: Organize by Feature Modules
impact: HIGH
impactDescription: Improves maintainability and team scalability
tags: [architecture, modules, organization, domain-driven]
---

## Organize Code by Feature / Domain Modules

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
