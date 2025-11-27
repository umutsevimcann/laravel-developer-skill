# Laravel Performans Optimizasyonu

## Database Optimization

### N+1 Problem Çözümü
```php
// ❌ N+1 Problem
$posts = Post::all();
foreach ($posts as $post) {
    echo $post->user->name; // Her post için ayrı query
}

// ✅ Eager Loading
$posts = Post::with('user')->get();

// ✅ Nested
$posts = Post::with(['user', 'comments.user'])->get();

// ✅ Selective columns
$posts = Post::with('user:id,name,email')->get();
```

### Query Optimization
```php
// ✅ Select sadece gerekli kolonları
$posts = Post::select(['id', 'title', 'user_id'])
    ->with('user:id,name')
    ->get();

// ✅ Chunk processing
Post::chunk(1000, function ($posts) {
    foreach ($posts as $post) {
        // Process
    }
});

// ✅ Lazy collection (memory efficient)
Post::lazy()->each(function ($post) {
    // Process
});

// ✅ Cursor (en memory efficient)
foreach (Post::cursor() as $post) {
    // Process
}

// ✅ Aggregation işlemleri DB'de yap
$count = Post::count();
$sum = Order::sum('total');
$avg = Product::avg('price');

// ❌ PHP'de aggregation yapma
$posts = Post::all();
$count = count($posts); // Tüm veriyi çekiyor
```

### Index Optimization
```php
// Migration'da index ekle
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->string('slug')->unique();
    $table->string('status');
    $table->timestamp('published_at')->nullable();
    $table->timestamps();
    
    // Composite index
    $table->index(['status', 'published_at']);
    
    // Full-text index
    $table->fullText(['title', 'content']);
});

// Explain query
DB::enableQueryLog();
$posts = Post::where('status', 'published')->get();
dd(DB::getQueryLog());
```

## Caching

### Query Caching
```php
// Basic cache
$posts = Cache::remember('posts.published', 3600, function () {
    return Post::published()->with('user')->get();
});

// Cache tags
$posts = Cache::tags(['posts'])->remember('posts.all', 3600, function () {
    return Post::all();
});

// Invalidation
Cache::tags(['posts'])->flush();

// Cache forever (manuel invalidate gerekir)
$settings = Cache::rememberForever('settings', function () {
    return Setting::all()->pluck('value', 'key');
});
```

### Model Caching
```php
// config/cache.php ile
public function getPostsCached()
{
    return Cache::remember(
        "user.{$this->id}.posts",
        now()->addHours(1),
        fn () => $this->posts()->published()->get()
    );
}

// Observer ile cache invalidation
class PostObserver
{
    public function saved(Post $post): void
    {
        Cache::forget("user.{$post->user_id}.posts");
        Cache::tags(['posts'])->flush();
    }
}
```

### Route Caching
```bash
# Production'da route'ları cache'le
php artisan route:cache

# Route cache temizle
php artisan route:clear
```

### Config Caching
```bash
# Production'da config'i cache'le
php artisan config:cache

# Config cache temizle
php artisan config:clear
```

### View Caching
```bash
# Blade view'ları compile et
php artisan view:cache

# View cache temizle
php artisan view:clear
```

## Queue System

### Job Oluşturma
```php
// php artisan make:job ProcessPost

class ProcessPost implements ShouldQueue
{
    use Queueable;

    public int $tries = 3;
    public int $timeout = 120;
    public int $backoff = 60;

    public function __construct(
        public readonly Post $post
    ) {}

    public function handle(): void
    {
        // Heavy processing
        $this->post->processContent();
        $this->post->generateThumbnails();
        $this->post->notifySubscribers();
    }

    public function failed(\Throwable $exception): void
    {
        Log::error('Post processing failed', [
            'post_id' => $this->post->id,
            'error' => $exception->getMessage(),
        ]);
    }
}
```

### Job Dispatch
```php
// Immediate dispatch
ProcessPost::dispatch($post);

// Delayed dispatch
ProcessPost::dispatch($post)->delay(now()->addMinutes(10));

// Specific queue
ProcessPost::dispatch($post)->onQueue('processing');

// Chain jobs
Bus::chain([
    new ProcessPost($post),
    new GenerateThumbnails($post),
    new NotifySubscribers($post),
])->dispatch();

// Batch jobs
Bus::batch([
    new ProcessPost($post1),
    new ProcessPost($post2),
    new ProcessPost($post3),
])->then(function (Batch $batch) {
    // All completed
})->catch(function (Batch $batch, \Throwable $e) {
    // First failure
})->finally(function (Batch $batch) {
    // Batch finished
})->dispatch();
```

### Queue Worker
```bash
# Worker başlat
php artisan queue:work

# Specific queue
php artisan queue:work --queue=high,default,low

# Memory limit
php artisan queue:work --memory=128

# Supervisor config (production)
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/app/artisan queue:work --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
numprocs=8
redirect_stderr=true
stdout_logfile=/var/www/app/storage/logs/worker.log
```

### Horizon (Redis queue yönetimi)
```bash
composer require laravel/horizon
php artisan horizon:install
php artisan horizon
```

## Laravel Octane

### Kurulum
```bash
composer require laravel/octane
php artisan octane:install
```

### Çalıştırma
```bash
# FrankenPHP ile
php artisan octane:start --server=frankenphp

# Swoole ile
php artisan octane:start --server=swoole

# RoadRunner ile
php artisan octane:start --server=roadrunner
```

### Octane Dikkat Edilecekler
```php
// ❌ Statik property'ler request'ler arasında paylaşılır
class Service
{
    public static array $data = []; // Tehlikeli!
}

// ✅ Container'dan resolve et
$service = app(Service::class);

// ❌ Global state kullanma
$_SESSION['user'] = $user; // Çalışmaz

// ✅ Request helper kullan
request()->user();
```

## Response Optimization

### Compression
```php
// .htaccess veya nginx config'de gzip aktif et

// Response headers
return response($content)
    ->header('Content-Encoding', 'gzip')
    ->header('Cache-Control', 'max-age=3600');
```

### Lazy Loading Prevention
```php
// AppServiceProvider.php
public function boot(): void
{
    Model::preventLazyLoading(!app()->isProduction());
    Model::preventSilentlyDiscardingAttributes(!app()->isProduction());
}
```

### Debugbar Devre Dışı
```php
// .env (production)
DEBUGBAR_ENABLED=false
APP_DEBUG=false
```

## Production Checklist

```bash
# 1. Optimize autoloader
composer install --optimize-autoloader --no-dev

# 2. Cache configuration
php artisan config:cache

# 3. Cache routes
php artisan route:cache

# 4. Cache views
php artisan view:cache

# 5. Cache events
php artisan event:cache

# 6. Optimize
php artisan optimize

# Tümünü temizle
php artisan optimize:clear
```

## Monitoring

### Laravel Telescope (Development)
```bash
composer require laravel/telescope --dev
php artisan telescope:install
php artisan migrate
```

### Laravel Pulse (Production)
```bash
composer require laravel/pulse
php artisan vendor:publish --provider="Laravel\Pulse\PulseServiceProvider"
php artisan migrate
```

```php
// Dashboard: /pulse
// Slow queries, exceptions, queue jobs izleme
```
