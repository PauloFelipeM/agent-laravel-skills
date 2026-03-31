---
title: Validate All Input with Form Requests
impact: CRITICAL
impactDescription: Unvalidated input is the #1 attack vector
tags: [security, validation, form-requests, input]
---

## Validate All Input with Form Requests

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
