---
name: laravel-developer
description: >
  Laravel 12+ framework uzmanı. Modern PHP ve Laravel uygulamaları geliştirmek için kullanılır.
  Eloquent ORM, Blade şablonları, API geliştirme, authentication, authorization, queue sistemleri,
  event broadcasting, testing, deployment ve güvenlik konularında uzmanlık sağlar.
  
  Tetikleyiciler: Laravel, Eloquent, Blade, Artisan, migration, seeder, factory, middleware,
  controller, model, route, API, Sanctum, Passport, Livewire, Inertia, Filament, Nova,
  queue, job, event, listener, observer, policy, gate, resource, collection, request,
  validation, service provider, facade, helper, config, .env, composer, php artisan,
  tinker, sail, forge, vapor, horizon, telescope, pulse, pennant, reverb, octane.
---

# Laravel Developer Skill

Laravel 12+ için modern, güvenli ve performanslı uygulama geliştirme rehberi.

## 📚 Resmi Kaynaklar

Geliştirme sırasında güncel bilgi için bu kaynakları kullan:

| Kaynak | URL |
|--------|-----|
| Laravel Docs | https://laravel.com/docs |
| API Reference | https://laravel.com/api |
| Laravel News | https://laravel-news.com |
| Laracasts | https://laracasts.com |

**ÖNEMLİ:** Laravel sık güncellenir. Yeni özellikler veya değişiklikler için her zaman web_search ile güncel dokümantasyonu kontrol et.

## Laravel 12 Temel Özellikleri

### PHP Gereksinimleri
- PHP 8.2 - 8.4 arası destekleniyor
- Composer 2.x gerekli

### Yeni Starter Kit'ler (Laravel 12)
Laravel 12 ile Breeze ve Jetstream yerine yeni starter kit'ler geldi:

```bash
# React starter kit (Inertia 2, TypeScript, shadcn/ui, Tailwind)
laravel new myapp  # React seç

# Vue starter kit (Vue 3, TypeScript, shadcn-vue, Tailwind)
laravel new myapp  # Vue seç

# Livewire starter kit (Livewire 3, Volt, Flux UI, Tailwind)
laravel new myapp  # Livewire seç
```

## Proje Yapısı (Laravel 12)

```
app/
├── Console/Commands/       # Artisan komutları
├── Exceptions/             # Exception handler'lar
├── Http/
│   ├── Controllers/        # Controller'lar
│   ├── Middleware/         # Middleware'ler
│   └── Requests/           # Form Request validation
├── Models/                 # Eloquent modelleri
├── Policies/               # Authorization policy'leri
├── Providers/              # Service provider'lar
├── Services/               # Business logic servisleri
├── Actions/                # Single-action sınıfları
├── DTOs/                   # Data Transfer Objects
├── Enums/                  # PHP 8.1+ enum'lar
└── Events/, Listeners/, Jobs/, Mail/, Notifications/
bootstrap/
├── app.php                 # Application bootstrap
├── providers.php           # Service provider listesi
config/                     # Konfigürasyon dosyaları
database/
├── migrations/             # Veritabanı migration'ları
├── seeders/                # Data seeder'lar
└── factories/              # Model factory'leri
resources/
├── views/                  # Blade şablonları
├── css/ & js/              # Frontend assets
routes/
├── web.php                 # Web route'ları
├── api.php                 # API route'ları
├── console.php             # Artisan route'ları
└── channels.php            # Broadcast channel'ları
storage/ & public/
tests/
├── Feature/                # Feature testleri
└── Unit/                   # Unit testleri
```

## Eloquent Model Best Practices

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Builder;
use App\Enums\PostStatus;

class Post extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'title',
        'slug',
        'content',
        'status',
        'user_id',
        'published_at',
    ];

    protected function casts(): array
    {
        return [
            'published_at' => 'datetime',
            'status' => PostStatus::class,      // Enum casting
            'meta' => 'array',                   // JSON casting
            'is_featured' => 'boolean',
        ];
    }

    // Accessor (Laravel 9+ syntax)
    protected function title(): Attribute
    {
        return Attribute::make(
            get: fn (string $value) => ucfirst($value),
            set: fn (string $value) => strtolower($value),
        );
    }

    // Relationships
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function comments(): HasMany
    {
        return $this->hasMany(Comment::class);
    }

    // Scopes
    public function scopePublished(Builder $query): Builder
    {
        return $query->where('status', PostStatus::Published)
                     ->whereNotNull('published_at')
                     ->where('published_at', '<=', now());
    }

    public function scopeByAuthor(Builder $query, int $userId): Builder
    {
        return $query->where('user_id', $userId);
    }

    // Boot method for model events
    protected static function booted(): void
    {
        static::creating(function (Post $post) {
            $post->slug ??= Str::slug($post->title);
        });
    }
}
```

## Controller Best Practices

```php
<?php

namespace App\Http\Controllers;

use App\Http\Requests\StorePostRequest;
use App\Http\Requests\UpdatePostRequest;
use App\Http\Resources\PostResource;
use App\Models\Post;
use App\Services\PostService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;

class PostController extends Controller
{
    public function __construct(
        private readonly PostService $postService
    ) {}

    public function index(): AnonymousResourceCollection
    {
        $posts = Post::query()
            ->published()
            ->with(['user:id,name', 'comments'])
            ->withCount('comments')
            ->latest('published_at')
            ->paginate(15);

        return PostResource::collection($posts);
    }

    public function store(StorePostRequest $request): JsonResponse
    {
        $post = $this->postService->create($request->validated());

        return response()->json([
            'message' => __('Post created successfully'),
            'data' => new PostResource($post),
        ], 201);
    }

    public function show(Post $post): PostResource
    {
        $this->authorize('view', $post);
        
        return new PostResource($post->load(['user', 'comments.user']));
    }

    public function update(UpdatePostRequest $request, Post $post): PostResource
    {
        $post = $this->postService->update($post, $request->validated());

        return new PostResource($post);
    }

    public function destroy(Post $post): JsonResponse
    {
        $this->authorize('delete', $post);
        
        $post->delete();

        return response()->json(['message' => __('Post deleted')]);
    }
}
```

## Form Request Validation

```php
<?php

namespace App\Http\Requests;

use App\Enums\PostStatus;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;
use Illuminate\Validation\Rules\Enum;

class StorePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('create', Post::class);
    }

    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'slug' => ['nullable', 'string', 'max:255', 'unique:posts,slug'],
            'content' => ['required', 'string', 'min:100'],
            'status' => ['required', new Enum(PostStatus::class)],
            'category_id' => ['required', 'exists:categories,id'],
            'tags' => ['nullable', 'array'],
            'tags.*' => ['exists:tags,id'],
            'published_at' => ['nullable', 'date', 'after_or_equal:today'],
            'meta' => ['nullable', 'array'],
            'meta.seo_title' => ['nullable', 'string', 'max:60'],
            'meta.seo_description' => ['nullable', 'string', 'max:160'],
        ];
    }

    public function messages(): array
    {
        return [
            'title.required' => __('Başlık alanı zorunludur.'),
            'content.min' => __('İçerik en az :min karakter olmalıdır.'),
        ];
    }

    protected function prepareForValidation(): void
    {
        $this->merge([
            'slug' => $this->slug ?? Str::slug($this->title),
        ]);
    }
}
```

## API Resources

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class PostResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'slug' => $this->slug,
            'excerpt' => Str::limit($this->content, 150),
            'content' => $this->when($request->routeIs('posts.show'), $this->content),
            'status' => $this->status->value,
            'is_published' => $this->status->isPublished(),
            'published_at' => $this->published_at?->toIso8601String(),
            'created_at' => $this->created_at->toIso8601String(),
            
            // Conditional relationships
            'author' => new UserResource($this->whenLoaded('user')),
            'comments' => CommentResource::collection($this->whenLoaded('comments')),
            'comments_count' => $this->whenCounted('comments'),
            
            // Links
            'links' => [
                'self' => route('posts.show', $this->slug),
                'edit' => $this->when(
                    $request->user()?->can('update', $this->resource),
                    route('posts.edit', $this->slug)
                ),
            ],
        ];
    }
}
```

## Güvenlik Kontrol Listesi

### 1. Mass Assignment Koruması
```php
// ❌ YANLIŞ - Tüm input'u kabul etme
$post = Post::create($request->all());

// ✅ DOĞRU - Sadece izin verilenleri kabul et
$post = Post::create($request->validated());
$post = Post::create($request->only(['title', 'content']));
```

### 2. SQL Injection Koruması
```php
// ❌ YANLIŞ - Raw query ile user input
DB::select("SELECT * FROM users WHERE name = '$name'");

// ✅ DOĞRU - Prepared statements
DB::select('SELECT * FROM users WHERE name = ?', [$name]);
User::where('name', $name)->get();
```

### 3. XSS Koruması
```blade
{{-- ❌ YANLIŞ - Escape edilmemiş output --}}
{!! $userInput !!}

{{-- ✅ DOĞRU - Otomatik escape --}}
{{ $userInput }}

{{-- ✅ HTML gerekiyorsa sanitize et --}}
{!! clean($userInput) !!}
```

### 4. CSRF Koruması
```blade
{{-- Form'larda CSRF token kullan --}}
<form method="POST">
    @csrf
    ...
</form>
```

### 5. Authorization
```php
// Policy kullan
$this->authorize('update', $post);

// Gate kullan
Gate::authorize('edit-settings');

// Middleware ile
Route::middleware('can:manage-users')->group(...);
```

### 6. Rate Limiting
```php
// routes/api.php
Route::middleware('throttle:60,1')->group(function () {
    Route::apiResource('posts', PostController::class);
});

// Özel rate limiter
RateLimiter::for('api', function (Request $request) {
    return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
});
```

### 7. Güvenli Dosya Yükleme
```php
public function upload(Request $request)
{
    $request->validate([
        'file' => [
            'required',
            'file',
            'max:10240', // 10MB
            'mimes:pdf,doc,docx,jpg,png',
        ],
    ]);

    $path = $request->file('file')->store('uploads', 'private');
    
    return response()->json(['path' => $path]);
}
```

## Detaylı Rehberler

Daha fazla bilgi için references/ klasöründeki dosyalara bak:

| Dosya | İçerik |
|-------|--------|
| `references/eloquent-advanced.md` | İleri Eloquent teknikleri, ilişkiler, query optimization |
| `references/api-development.md` | REST API, Sanctum, versioning, documentation |
| `references/testing.md` | PHPUnit, Pest, feature/unit testing, mocking |
| `references/performance.md` | Cache, queue, eager loading, database optimization |
| `references/deployment.md` | Forge, Vapor, Cloud, CI/CD, production checklist |
| `references/packages.md` | Popüler paketler: Spatie, Filament, Livewire, Inertia |
| `references/security-checklist.md` | Kapsamlı güvenlik kontrol listesi |
