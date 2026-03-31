---
title: Use Form Requests for Input Transformation
impact: MEDIUM
impactDescription: Inconsistent input handling causes subtle bugs
tags: api, form-requests, validation, transformation, input
---

## Use Form Requests for Input Transformation

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
