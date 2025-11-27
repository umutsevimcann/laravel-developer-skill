# Eloquent İleri Seviye Rehberi

## İlişki Türleri

### One-to-One
```php
// User.php
public function profile(): HasOne
{
    return $this->hasOne(Profile::class);
}

// Profile.php
public function user(): BelongsTo
{
    return $this->belongsTo(User::class);
}
```

### One-to-Many
```php
// User.php
public function posts(): HasMany
{
    return $this->hasMany(Post::class);
}

// Post.php
public function user(): BelongsTo
{
    return $this->belongsTo(User::class);
}
```

### Many-to-Many
```php
// Post.php
public function tags(): BelongsToMany
{
    return $this->belongsToMany(Tag::class)
        ->withPivot(['order', 'created_at'])
        ->withTimestamps()
        ->orderByPivot('order');
}

// Tag.php
public function posts(): BelongsToMany
{
    return $this->belongsToMany(Post::class);
}

// Kullanım
$post->tags()->attach($tagId);
$post->tags()->attach($tagId, ['order' => 1]);
$post->tags()->detach($tagId);
$post->tags()->sync([1, 2, 3]);
$post->tags()->syncWithoutDetaching([1, 2]);
$post->tags()->toggle([1, 2, 3]);
```

### Has-Many-Through
```php
// Country.php - Country -> Users -> Posts
public function posts(): HasManyThrough
{
    return $this->hasManyThrough(
        Post::class,      // Final model
        User::class,      // Intermediate model
        'country_id',     // Foreign key on users table
        'user_id',        // Foreign key on posts table
        'id',             // Local key on countries table
        'id'              // Local key on users table
    );
}
```

### Polymorphic Relations
```php
// Comment.php - comments tablosu: commentable_type, commentable_id
public function commentable(): MorphTo
{
    return $this->morphTo();
}

// Post.php
public function comments(): MorphMany
{
    return $this->morphMany(Comment::class, 'commentable');
}

// Video.php
public function comments(): MorphMany
{
    return $this->morphMany(Comment::class, 'commentable');
}
```

### Many-to-Many Polymorphic
```php
// Tag.php
public function posts(): MorphToMany
{
    return $this->morphedByMany(Post::class, 'taggable');
}

public function videos(): MorphToMany
{
    return $this->morphedByMany(Video::class, 'taggable');
}

// Post.php
public function tags(): MorphToMany
{
    return $this->morphToMany(Tag::class, 'taggable');
}
```

## Query Optimization

### Eager Loading
```php
// N+1 problemi - YANLIŞ
$posts = Post::all();
foreach ($posts as $post) {
    echo $post->user->name; // Her iterasyonda query
}

// Eager loading - DOĞRU
$posts = Post::with('user')->get();
$posts = Post::with(['user', 'comments', 'tags'])->get();

// Nested eager loading
$posts = Post::with('comments.user')->get();

// Conditional eager loading
$posts = Post::with(['comments' => function ($query) {
    $query->where('approved', true)
          ->orderBy('created_at', 'desc');
}])->get();

// Eager load count
$posts = Post::withCount('comments')->get();
// $post->comments_count

// Conditional count
$posts = Post::withCount(['comments', 'comments as approved_comments_count' => function ($query) {
    $query->where('approved', true);
}])->get();
```

### Lazy Eager Loading
```php
$posts = Post::all();

if ($needComments) {
    $posts->load('comments');
}

// Missing ile sadece yüklenmemişleri yükle
$posts->loadMissing('comments');
```

### Chunking
```php
// Büyük veri setleri için
Post::chunk(100, function ($posts) {
    foreach ($posts as $post) {
        // Process
    }
});

// Lazy collection
Post::lazy()->each(function ($post) {
    // Process
});

// Cursor (memory efficient)
foreach (Post::cursor() as $post) {
    // Process
}
```

### Query Caching
```php
// Cache query sonuçları
$posts = Cache::remember('posts.published', 3600, function () {
    return Post::published()->with('user')->get();
});

// Cache tags ile
$posts = Cache::tags(['posts'])->remember('posts.all', 3600, function () {
    return Post::all();
});

// Cache invalidation
Cache::tags(['posts'])->flush();
```

## Advanced Query Building

### Subqueries
```php
// Subquery select
$users = User::addSelect([
    'last_post_at' => Post::select('created_at')
        ->whereColumn('user_id', 'users.id')
        ->latest()
        ->limit(1)
])->get();

// Subquery orderBy
$users = User::orderByDesc(
    Post::select('created_at')
        ->whereColumn('user_id', 'users.id')
        ->latest()
        ->limit(1)
)->get();
```

### When Conditional
```php
$posts = Post::query()
    ->when($request->search, function ($query, $search) {
        $query->where('title', 'like', "%{$search}%");
    })
    ->when($request->status, function ($query, $status) {
        $query->where('status', $status);
    })
    ->when($request->sort === 'oldest', function ($query) {
        $query->oldest();
    }, function ($query) {
        $query->latest();
    })
    ->paginate();
```

### Raw Expressions
```php
// Raw select
$users = User::select(DB::raw('count(*) as post_count, user_id'))
    ->groupBy('user_id')
    ->get();

// selectRaw
$orders = Order::selectRaw('price * quantity as total')->get();

// whereRaw
$orders = Order::whereRaw('price > IF(state = "TX", ?, 100)', [200])->get();

// orderByRaw
$orders = Order::orderByRaw('updated_at - created_at DESC')->get();
```

## Model Events & Observers

### Observer
```php
// php artisan make:observer PostObserver --model=Post

class PostObserver
{
    public function creating(Post $post): void
    {
        $post->slug ??= Str::slug($post->title);
        $post->user_id ??= auth()->id();
    }

    public function created(Post $post): void
    {
        event(new PostCreated($post));
    }

    public function updating(Post $post): void
    {
        if ($post->isDirty('title')) {
            $post->slug = Str::slug($post->title);
        }
    }

    public function deleted(Post $post): void
    {
        Storage::delete($post->image);
    }

    public function forceDeleted(Post $post): void
    {
        // Permanent delete actions
    }
}

// AppServiceProvider.php
public function boot(): void
{
    Post::observe(PostObserver::class);
}
```

### Inline Events (booted method)
```php
protected static function booted(): void
{
    static::creating(function (Post $post) {
        $post->uuid = Str::uuid();
    });

    static::deleting(function (Post $post) {
        $post->comments()->delete();
    });
}
```

## Query Scopes

### Local Scopes
```php
// Model'de
public function scopePopular(Builder $query, int $minViews = 1000): Builder
{
    return $query->where('views', '>=', $minViews);
}

public function scopePublishedBetween(Builder $query, Carbon $start, Carbon $end): Builder
{
    return $query->whereBetween('published_at', [$start, $end]);
}

// Kullanım
Post::popular()->publishedBetween(now()->subMonth(), now())->get();
```

### Global Scopes
```php
// Scope class
class ActiveScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        $builder->where('is_active', true);
    }
}

// Model'de uygula
protected static function booted(): void
{
    static::addGlobalScope(new ActiveScope);
}

// Anonymous global scope
static::addGlobalScope('active', function (Builder $builder) {
    $builder->where('is_active', true);
});

// Global scope'u devre dışı bırak
Post::withoutGlobalScope(ActiveScope::class)->get();
Post::withoutGlobalScopes()->get();
```

## Custom Casts

```php
// php artisan make:cast Json

class Json implements CastsAttributes
{
    public function get(Model $model, string $key, mixed $value, array $attributes): mixed
    {
        return json_decode($value, true);
    }

    public function set(Model $model, string $key, mixed $value, array $attributes): mixed
    {
        return json_encode($value);
    }
}

// Kullanım
protected function casts(): array
{
    return [
        'options' => Json::class,
        'address' => AddressCast::class,
    ];
}
```

### Value Object Cast
```php
class Address implements Castable
{
    public static function castUsing(array $arguments): CastsAttributes
    {
        return new class implements CastsAttributes
        {
            public function get($model, $key, $value, $attributes): Address
            {
                return new Address(
                    $attributes['address_line_1'],
                    $attributes['address_line_2'],
                    $attributes['city'],
                    $attributes['country'],
                );
            }

            public function set($model, $key, $value, $attributes): array
            {
                return [
                    'address_line_1' => $value->lineOne,
                    'address_line_2' => $value->lineTwo,
                    'city' => $value->city,
                    'country' => $value->country,
                ];
            }
        };
    }
}
```
