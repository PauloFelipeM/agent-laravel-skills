---
title: Throw HTTP Exceptions from Services
impact: HIGH
impactDescription: Consistent error responses improve API reliability
tags: [error-handling, http-exceptions, abort, responses]
---

## Throw HTTP Exceptions from Services

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
