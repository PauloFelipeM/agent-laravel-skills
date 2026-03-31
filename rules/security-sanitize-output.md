---
title: Sanitize Output to Prevent XSS
impact: HIGH
impactDescription: XSS attacks can steal user sessions and data
tags: [security, xss, sanitization, blade, output]
---

## Sanitize Output to Prevent XSS

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
