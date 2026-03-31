---
title: Use HTTP Tests for Feature Testing
impact: HIGH
impactDescription: HTTP tests catch integration issues that unit tests miss
tags: [testing, http, feature-tests, api-testing, assertions]
---

## Use Laravel's HTTP Test Methods for Comprehensive Feature Testing

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
