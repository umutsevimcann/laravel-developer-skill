# Laravel Testing Rehberi

## Test Türleri

- **Unit Tests**: Tek bir sınıf veya method'u izole test eder
- **Feature Tests**: HTTP istekleri, veritabanı işlemleri gibi tam akışları test eder
- **Browser Tests** (Dusk): Gerçek tarayıcı ile end-to-end testler

## PHPUnit vs Pest

Laravel 12 her ikisini de destekler. Pest daha modern ve okunabilir syntax sunar.

```bash
# PHPUnit ile test çalıştır
php artisan test

# Pest ile
./vendor/bin/pest

# Paralel test
php artisan test --parallel

# Coverage
php artisan test --coverage --min=80
```

## Feature Testing

### HTTP Tests
```php
<?php

namespace Tests\Feature;

use App\Models\Post;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class PostTest extends TestCase
{
    use RefreshDatabase;

    public function test_can_list_posts(): void
    {
        Post::factory()->count(5)->create();

        $response = $this->getJson('/api/v1/posts');

        $response->assertOk()
            ->assertJsonCount(5, 'data')
            ->assertJsonStructure([
                'data' => [
                    '*' => ['id', 'title', 'slug', 'created_at'],
                ],
                'meta' => ['total', 'per_page'],
            ]);
    }

    public function test_can_create_post(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)
            ->postJson('/api/v1/posts', [
                'title' => 'Test Post',
                'content' => 'Test content that is long enough...',
                'status' => 'draft',
            ]);

        $response->assertCreated()
            ->assertJsonPath('data.title', 'Test Post');

        $this->assertDatabaseHas('posts', [
            'title' => 'Test Post',
            'user_id' => $user->id,
        ]);
    }

    public function test_validates_required_fields(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)
            ->postJson('/api/v1/posts', []);

        $response->assertUnprocessable()
            ->assertJsonValidationErrors(['title', 'content']);
    }

    public function test_guest_cannot_create_post(): void
    {
        $response = $this->postJson('/api/v1/posts', [
            'title' => 'Test',
            'content' => 'Content',
        ]);

        $response->assertUnauthorized();
    }

    public function test_user_cannot_update_others_post(): void
    {
        $post = Post::factory()->create();
        $otherUser = User::factory()->create();

        $response = $this->actingAs($otherUser)
            ->putJson("/api/v1/posts/{$post->id}", [
                'title' => 'Updated',
            ]);

        $response->assertForbidden();
    }
}
```

### Database Assertions
```php
// Kayıt var mı?
$this->assertDatabaseHas('posts', [
    'title' => 'Test Post',
    'user_id' => $user->id,
]);

// Kayıt yok mu?
$this->assertDatabaseMissing('posts', [
    'title' => 'Deleted Post',
]);

// Kayıt sayısı
$this->assertDatabaseCount('posts', 5);

// Soft delete
$this->assertSoftDeleted('posts', ['id' => $post->id]);
$this->assertNotSoftDeleted('posts', ['id' => $post->id]);
```

### Authentication Tests
```php
public function test_can_login_with_valid_credentials(): void
{
    $user = User::factory()->create([
        'password' => bcrypt('password'),
    ]);

    $response = $this->postJson('/api/v1/auth/login', [
        'email' => $user->email,
        'password' => 'password',
    ]);

    $response->assertOk()
        ->assertJsonStructure(['token', 'user']);
}

public function test_cannot_login_with_invalid_credentials(): void
{
    $user = User::factory()->create();

    $response = $this->postJson('/api/v1/auth/login', [
        'email' => $user->email,
        'password' => 'wrong-password',
    ]);

    $response->assertUnprocessable();
}

public function test_can_logout(): void
{
    $user = User::factory()->create();
    $token = $user->createToken('test')->plainTextToken;

    $response = $this->withToken($token)
        ->postJson('/api/v1/auth/logout');

    $response->assertOk();
    $this->assertDatabaseMissing('personal_access_tokens', [
        'tokenable_id' => $user->id,
    ]);
}
```

## Pest Testing

### Basic Pest Tests
```php
<?php

use App\Models\Post;
use App\Models\User;

test('can list posts', function () {
    Post::factory()->count(5)->create();

    $this->getJson('/api/v1/posts')
        ->assertOk()
        ->assertJsonCount(5, 'data');
});

test('can create post', function () {
    $user = User::factory()->create();

    $this->actingAs($user)
        ->postJson('/api/v1/posts', [
            'title' => 'Test Post',
            'content' => str_repeat('Content ', 20),
            'status' => 'draft',
        ])
        ->assertCreated();

    expect(Post::count())->toBe(1);
    expect(Post::first()->title)->toBe('Test Post');
});

test('guest cannot create post', function () {
    $this->postJson('/api/v1/posts', [])
        ->assertUnauthorized();
});
```

### Pest Expectations
```php
test('post has correct attributes', function () {
    $post = Post::factory()->create([
        'title' => 'Test Post',
        'status' => 'published',
    ]);

    expect($post)
        ->title->toBe('Test Post')
        ->status->toBe('published')
        ->user_id->toBeInt()
        ->created_at->toBeInstanceOf(Carbon::class);
});

test('collection expectations', function () {
    $posts = Post::factory()->count(5)->create();

    expect($posts)
        ->toHaveCount(5)
        ->each->toBeInstanceOf(Post::class);
});

test('json structure', function () {
    Post::factory()->create();

    $this->getJson('/api/v1/posts')
        ->assertOk()
        ->assertJson(fn (AssertableJson $json) =>
            $json->has('data', 1, fn ($json) =>
                $json->whereType('id', 'integer')
                    ->whereType('title', 'string')
                    ->etc()
            )
        );
});
```

### Pest Datasets
```php
dataset('invalid_emails', [
    'missing @' => ['invalidemail'],
    'missing domain' => ['test@'],
    'missing local' => ['@domain.com'],
]);

test('rejects invalid emails', function (string $email) {
    $response = $this->postJson('/api/v1/auth/register', [
        'email' => $email,
        'password' => 'password123',
    ]);

    $response->assertUnprocessable()
        ->assertJsonValidationErrors(['email']);
})->with('invalid_emails');
```

## Mocking

### Facades
```php
use Illuminate\Support\Facades\Mail;
use App\Mail\WelcomeEmail;

public function test_sends_welcome_email_on_registration(): void
{
    Mail::fake();

    $this->postJson('/api/v1/auth/register', [
        'name' => 'Test User',
        'email' => 'test@example.com',
        'password' => 'password123',
    ]);

    Mail::assertSent(WelcomeEmail::class, function ($mail) {
        return $mail->hasTo('test@example.com');
    });

    Mail::assertSent(WelcomeEmail::class, 1);
}
```

### Queue
```php
use Illuminate\Support\Facades\Queue;
use App\Jobs\ProcessPost;

public function test_dispatches_job_on_post_creation(): void
{
    Queue::fake();

    $user = User::factory()->create();

    $this->actingAs($user)
        ->postJson('/api/v1/posts', [
            'title' => 'Test',
            'content' => str_repeat('Content ', 20),
        ]);

    Queue::assertPushed(ProcessPost::class);
    Queue::assertPushed(ProcessPost::class, function ($job) {
        return $job->post->title === 'Test';
    });
}
```

### Events
```php
use Illuminate\Support\Facades\Event;
use App\Events\PostCreated;

public function test_fires_event_on_post_creation(): void
{
    Event::fake([PostCreated::class]);

    $user = User::factory()->create();

    $this->actingAs($user)
        ->postJson('/api/v1/posts', [...]);

    Event::assertDispatched(PostCreated::class);
}
```

### HTTP Client
```php
use Illuminate\Support\Facades\Http;

public function test_fetches_external_data(): void
{
    Http::fake([
        'api.example.com/*' => Http::response([
            'data' => ['name' => 'Test'],
        ], 200),
    ]);

    $response = $this->getJson('/api/v1/external-data');

    $response->assertOk()
        ->assertJsonPath('data.name', 'Test');

    Http::assertSent(function ($request) {
        return $request->url() === 'https://api.example.com/data';
    });
}
```

## Model Factories

```php
<?php

namespace Database\Factories;

use App\Enums\PostStatus;
use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;

class PostFactory extends Factory
{
    public function definition(): array
    {
        return [
            'user_id' => User::factory(),
            'title' => fake()->sentence(),
            'slug' => fake()->slug(),
            'content' => fake()->paragraphs(3, true),
            'status' => fake()->randomElement(PostStatus::cases()),
            'published_at' => fake()->optional()->dateTimeBetween('-1 year'),
            'meta' => [
                'seo_title' => fake()->sentence(),
                'seo_description' => fake()->text(160),
            ],
        ];
    }

    public function published(): static
    {
        return $this->state(fn (array $attributes) => [
            'status' => PostStatus::Published,
            'published_at' => now(),
        ]);
    }

    public function draft(): static
    {
        return $this->state(fn (array $attributes) => [
            'status' => PostStatus::Draft,
            'published_at' => null,
        ]);
    }

    public function byUser(User $user): static
    {
        return $this->state(fn (array $attributes) => [
            'user_id' => $user->id,
        ]);
    }
}
```

### Factory Kullanımı
```php
// Tek kayıt
$post = Post::factory()->create();
$post = Post::factory()->published()->create();

// Çoklu
$posts = Post::factory()->count(10)->create();

// İlişkili
$user = User::factory()
    ->has(Post::factory()->count(5))
    ->create();

// Sequence
$posts = Post::factory()
    ->count(3)
    ->sequence(
        ['status' => 'draft'],
        ['status' => 'published'],
        ['status' => 'archived'],
    )
    ->create();

// State combination
$posts = Post::factory()
    ->published()
    ->byUser($user)
    ->count(5)
    ->create();
```

## Test Database

### RefreshDatabase Trait
```php
use Illuminate\Foundation\Testing\RefreshDatabase;

class ExampleTest extends TestCase
{
    use RefreshDatabase;
    
    // Her test öncesi migration çalışır
}
```

### DatabaseTransactions Trait
```php
use Illuminate\Foundation\Testing\DatabaseTransactions;

class ExampleTest extends TestCase
{
    use DatabaseTransactions;
    
    // Her test transaction içinde çalışır, sonra rollback
}
```

### Seeding in Tests
```php
public function test_with_seeded_data(): void
{
    $this->seed(CategorySeeder::class);
    // veya
    $this->seed(); // DatabaseSeeder çalıştırır

    // Test...
}
```
