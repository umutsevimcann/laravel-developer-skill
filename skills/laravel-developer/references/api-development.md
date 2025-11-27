# API Geliştirme Rehberi

## Route Tanımlama

### API Routes (routes/api.php)
```php
use App\Http\Controllers\Api\V1;

// Versiyonlu API
Route::prefix('v1')->name('api.v1.')->group(function () {
    // Public routes
    Route::get('posts', [V1\PostController::class, 'index']);
    Route::get('posts/{post:slug}', [V1\PostController::class, 'show']);
    
    // Auth routes
    Route::post('auth/login', [V1\AuthController::class, 'login']);
    Route::post('auth/register', [V1\AuthController::class, 'register']);
    
    // Protected routes
    Route::middleware('auth:sanctum')->group(function () {
        Route::post('auth/logout', [V1\AuthController::class, 'logout']);
        Route::get('auth/user', [V1\AuthController::class, 'user']);
        
        Route::apiResource('posts', V1\PostController::class)->except(['index', 'show']);
        Route::apiResource('comments', V1\CommentController::class);
        
        // Nested resources
        Route::apiResource('posts.comments', V1\PostCommentController::class)->shallow();
    });
});

// Rate limiting
Route::middleware(['auth:sanctum', 'throttle:api'])->group(function () {
    Route::post('upload', [UploadController::class, 'store']);
});
```

## Laravel Sanctum Authentication

### Kurulum
```bash
composer require laravel/sanctum
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"
php artisan migrate
```

### SPA Authentication
```php
// config/cors.php
'supports_credentials' => true,

// config/sanctum.php
'stateful' => explode(',', env('SANCTUM_STATEFUL_DOMAINS', 'localhost,localhost:3000')),

// routes/api.php
Route::middleware('auth:sanctum')->get('/user', function (Request $request) {
    return $request->user();
});
```

### API Token Authentication
```php
// AuthController.php
public function login(LoginRequest $request): JsonResponse
{
    $user = User::where('email', $request->email)->first();

    if (!$user || !Hash::check($request->password, $user->password)) {
        throw ValidationException::withMessages([
            'email' => ['The provided credentials are incorrect.'],
        ]);
    }

    // Token oluştur
    $token = $user->createToken(
        $request->device_name ?? 'api-token',
        ['*'], // abilities
        now()->addDays(7) // expiration
    );

    return response()->json([
        'user' => new UserResource($user),
        'token' => $token->plainTextToken,
        'expires_at' => $token->accessToken->expires_at,
    ]);
}

public function logout(Request $request): JsonResponse
{
    // Mevcut token'ı sil
    $request->user()->currentAccessToken()->delete();
    
    // Tüm token'ları sil
    // $request->user()->tokens()->delete();

    return response()->json(['message' => 'Logged out']);
}
```

### Token Abilities
```php
// Token oluşturma
$token = $user->createToken('api-token', ['posts:read', 'posts:write']);

// Route'da ability kontrolü
Route::middleware(['auth:sanctum', 'ability:posts:write'])->group(function () {
    Route::post('posts', [PostController::class, 'store']);
});

// Controller'da kontrol
if ($request->user()->tokenCan('posts:write')) {
    // ...
}
```

## API Resources & Collections

### Resource
```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class PostResource extends JsonResource
{
    // Wrap key'i değiştir
    public static $wrap = 'post';

    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'type' => 'post',
            'attributes' => [
                'title' => $this->title,
                'slug' => $this->slug,
                'content' => $this->when(
                    $request->routeIs('*.show'),
                    $this->content
                ),
                'excerpt' => $this->whenNotNull($this->excerpt),
                'status' => $this->status,
                'created_at' => $this->created_at->toIso8601String(),
                'updated_at' => $this->updated_at->toIso8601String(),
            ],
            'relationships' => [
                'author' => new UserResource($this->whenLoaded('user')),
                'tags' => TagResource::collection($this->whenLoaded('tags')),
                'comments_count' => $this->whenCounted('comments'),
            ],
            'links' => [
                'self' => route('api.v1.posts.show', $this->slug),
            ],
            'meta' => $this->when($request->user()?->can('update', $this->resource), [
                'can_edit' => true,
                'edit_url' => route('api.v1.posts.update', $this->slug),
            ]),
        ];
    }

    public function with(Request $request): array
    {
        return [
            'meta' => [
                'api_version' => 'v1',
            ],
        ];
    }
}
```

### Resource Collection
```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\ResourceCollection;

class PostCollection extends ResourceCollection
{
    public $collects = PostResource::class;

    public function toArray(Request $request): array
    {
        return [
            'data' => $this->collection,
        ];
    }

    public function with(Request $request): array
    {
        return [
            'meta' => [
                'total' => $this->total(),
                'per_page' => $this->perPage(),
                'current_page' => $this->currentPage(),
                'last_page' => $this->lastPage(),
            ],
            'links' => [
                'first' => $this->url(1),
                'last' => $this->url($this->lastPage()),
                'prev' => $this->previousPageUrl(),
                'next' => $this->nextPageUrl(),
            ],
        ];
    }
}
```

## Error Handling

### API Exception Handler
```php
// app/Exceptions/Handler.php
use Illuminate\Http\JsonResponse;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;

public function register(): void
{
    $this->renderable(function (NotFoundHttpException $e, Request $request) {
        if ($request->expectsJson()) {
            return response()->json([
                'error' => [
                    'code' => 'NOT_FOUND',
                    'message' => 'Resource not found',
                ],
            ], 404);
        }
    });

    $this->renderable(function (ValidationException $e, Request $request) {
        if ($request->expectsJson()) {
            return response()->json([
                'error' => [
                    'code' => 'VALIDATION_ERROR',
                    'message' => 'The given data was invalid.',
                    'errors' => $e->errors(),
                ],
            ], 422);
        }
    });

    $this->renderable(function (AuthenticationException $e, Request $request) {
        if ($request->expectsJson()) {
            return response()->json([
                'error' => [
                    'code' => 'UNAUTHENTICATED',
                    'message' => 'Unauthenticated',
                ],
            ], 401);
        }
    });
}
```

### Custom API Exceptions
```php
<?php

namespace App\Exceptions;

use Exception;
use Illuminate\Http\JsonResponse;

class ApiException extends Exception
{
    public function __construct(
        public string $errorCode,
        string $message,
        public int $statusCode = 400,
        public array $errors = []
    ) {
        parent::__construct($message);
    }

    public function render(): JsonResponse
    {
        return response()->json([
            'error' => [
                'code' => $this->errorCode,
                'message' => $this->getMessage(),
                'errors' => $this->errors ?: null,
            ],
        ], $this->statusCode);
    }
}

// Kullanım
throw new ApiException('INSUFFICIENT_BALANCE', 'Yetersiz bakiye', 400);
```

## Rate Limiting

```php
// app/Providers/AppServiceProvider.php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

public function boot(): void
{
    // Standart API limiti
    RateLimiter::for('api', function (Request $request) {
        return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
    });

    // Premium kullanıcılar için farklı limit
    RateLimiter::for('api', function (Request $request) {
        return $request->user()?->isPremium()
            ? Limit::perMinute(120)->by($request->user()->id)
            : Limit::perMinute(60)->by($request->ip());
    });

    // Endpoint bazlı limit
    RateLimiter::for('uploads', function (Request $request) {
        return Limit::perHour(10)->by($request->user()->id);
    });

    // Response ile
    RateLimiter::for('api', function (Request $request) {
        return Limit::perMinute(60)
            ->by($request->user()?->id ?: $request->ip())
            ->response(function () {
                return response()->json([
                    'error' => [
                        'code' => 'RATE_LIMIT_EXCEEDED',
                        'message' => 'Too many requests',
                    ],
                ], 429);
            });
    });
}
```

## API Versioning

### URL Prefix Versioning
```php
// routes/api.php
Route::prefix('v1')->name('v1.')->group(base_path('routes/api/v1.php'));
Route::prefix('v2')->name('v2.')->group(base_path('routes/api/v2.php'));
```

### Header Versioning
```php
// app/Http/Middleware/ApiVersion.php
class ApiVersion
{
    public function handle(Request $request, Closure $next): Response
    {
        $version = $request->header('Api-Version', 'v1');
        
        config(['app.api_version' => $version]);
        
        return $next($request);
    }
}
```

## Pagination

```php
// Controller
public function index(Request $request)
{
    $posts = Post::query()
        ->when($request->search, fn($q, $s) => $q->where('title', 'like', "%{$s}%"))
        ->when($request->status, fn($q, $s) => $q->where('status', $s))
        ->with(['user:id,name', 'tags:id,name'])
        ->withCount('comments')
        ->latest()
        ->paginate($request->input('per_page', 15))
        ->withQueryString();

    return new PostCollection($posts);
}

// Cursor pagination (performanslı)
$posts = Post::cursorPaginate(15);
```

## API Documentation

### OpenAPI/Swagger (L5-Swagger)
```bash
composer require darkaonline/l5-swagger
php artisan vendor:publish --provider="L5Swagger\L5SwaggerServiceProvider"
```

```php
/**
 * @OA\Get(
 *     path="/api/v1/posts",
 *     summary="List all posts",
 *     tags={"Posts"},
 *     @OA\Parameter(
 *         name="page",
 *         in="query",
 *         description="Page number",
 *         required=false,
 *         @OA\Schema(type="integer")
 *     ),
 *     @OA\Response(
 *         response=200,
 *         description="Successful operation",
 *         @OA\JsonContent(
 *             type="object",
 *             @OA\Property(property="data", type="array", @OA\Items(ref="#/components/schemas/Post"))
 *         )
 *     ),
 *     security={{"sanctum": {}}}
 * )
 */
public function index(): PostCollection
{
    // ...
}
```
