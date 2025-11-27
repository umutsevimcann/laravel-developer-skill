# Laravel Popüler Paketler

## Spatie Paketleri

### Laravel Permission
Rol ve izin yönetimi için en popüler paket.

```bash
composer require spatie/laravel-permission
php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider"
php artisan migrate
```

```php
// User.php
use Spatie\Permission\Traits\HasRoles;

class User extends Authenticatable
{
    use HasRoles;
}

// Kullanım
$user->assignRole('admin');
$user->givePermissionTo('edit articles');
$user->hasRole('admin'); // true
$user->can('edit articles'); // true

// Middleware
Route::group(['middleware' => ['role:admin']], function () {
    //
});

Route::group(['middleware' => ['permission:publish articles']], function () {
    //
});

// Blade
@role('admin')
    // Admin content
@endrole

@can('edit articles')
    // Edit button
@endcan
```

### Laravel Media Library
Dosya ve medya yönetimi.

```bash
composer require spatie/laravel-medialibrary
php artisan vendor:publish --provider="Spatie\MediaLibrary\MediaLibraryServiceProvider" --tag="medialibrary-migrations"
php artisan migrate
```

```php
// Post.php
use Spatie\MediaLibrary\HasMedia;
use Spatie\MediaLibrary\InteractsWithMedia;

class Post extends Model implements HasMedia
{
    use InteractsWithMedia;

    public function registerMediaCollections(): void
    {
        $this->addMediaCollection('images')
            ->useDisk('public')
            ->acceptsMimeTypes(['image/jpeg', 'image/png']);

        $this->addMediaCollection('documents')
            ->singleFile();
    }

    public function registerMediaConversions(Media $media = null): void
    {
        $this->addMediaConversion('thumb')
            ->width(200)
            ->height(200)
            ->sharpen(10);

        $this->addMediaConversion('preview')
            ->width(800)
            ->withResponsiveImages();
    }
}

// Kullanım
$post->addMedia($request->file('image'))->toMediaCollection('images');
$post->getFirstMediaUrl('images', 'thumb');
$post->getMedia('images');
```

### Laravel Backup
Veritabanı ve dosya yedekleme.

```bash
composer require spatie/laravel-backup
php artisan vendor:publish --provider="Spatie\Backup\BackupServiceProvider"
```

```php
// Backup çalıştır
php artisan backup:run

// Sadece db
php artisan backup:run --only-db

// Schedule
$schedule->command('backup:run')->daily()->at('01:00');
$schedule->command('backup:clean')->daily()->at('02:00');
```

### Laravel Activitylog
Kullanıcı aktivite loglama.

```bash
composer require spatie/laravel-activitylog
php artisan vendor:publish --provider="Spatie\Activitylog\ActivitylogServiceProvider" --tag="activitylog-migrations"
php artisan migrate
```

```php
// Model'de
use Spatie\Activitylog\Traits\LogsActivity;
use Spatie\Activitylog\LogOptions;

class Post extends Model
{
    use LogsActivity;

    public function getActivitylogOptions(): LogOptions
    {
        return LogOptions::defaults()
            ->logOnly(['title', 'content', 'status'])
            ->logOnlyDirty()
            ->dontSubmitEmptyLogs();
    }
}

// Manuel log
activity()
    ->causedBy($user)
    ->performedOn($post)
    ->withProperties(['key' => 'value'])
    ->log('Post güncellendi');

// Logları getir
$post->activities;
Activity::causedBy($user)->get();
```

### Laravel Data
DTO (Data Transfer Object) oluşturma.

```bash
composer require spatie/laravel-data
```

```php
use Spatie\LaravelData\Data;

class PostData extends Data
{
    public function __construct(
        public string $title,
        public string $content,
        public ?string $slug,
        public PostStatus $status,
        #[WithCast(DateTimeInterfaceCast::class)]
        public ?CarbonImmutable $published_at,
    ) {}

    public static function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'content' => ['required', 'string'],
        ];
    }
}

// Request'ten oluştur
$data = PostData::from($request);

// Model'den oluştur
$data = PostData::from($post);

// Array'e çevir
$data->toArray();
```

## Filament

Admin panel ve CRUD builder.

```bash
composer require filament/filament
php artisan filament:install --panels
php artisan make:filament-user
```

### Resource Oluşturma
```bash
php artisan make:filament-resource Post --generate
```

```php
// app/Filament/Resources/PostResource.php
class PostResource extends Resource
{
    protected static ?string $model = Post::class;
    protected static ?string $navigationIcon = 'heroicon-o-document-text';
    protected static ?string $navigationGroup = 'Content';

    public static function form(Form $form): Form
    {
        return $form->schema([
            Forms\Components\TextInput::make('title')
                ->required()
                ->maxLength(255)
                ->live(onBlur: true)
                ->afterStateUpdated(fn ($state, $set) => 
                    $set('slug', Str::slug($state))),
            
            Forms\Components\TextInput::make('slug')
                ->required()
                ->unique(ignoreRecord: true),
            
            Forms\Components\RichEditor::make('content')
                ->required()
                ->columnSpanFull(),
            
            Forms\Components\Select::make('status')
                ->options(PostStatus::class)
                ->required(),
            
            Forms\Components\DateTimePicker::make('published_at'),
            
            Forms\Components\Select::make('user_id')
                ->relationship('user', 'name')
                ->required(),
        ]);
    }

    public static function table(Table $table): Table
    {
        return $table
            ->columns([
                Tables\Columns\TextColumn::make('title')
                    ->searchable()
                    ->sortable(),
                Tables\Columns\TextColumn::make('user.name')
                    ->label('Author'),
                Tables\Columns\BadgeColumn::make('status')
                    ->colors([
                        'warning' => 'draft',
                        'success' => 'published',
                    ]),
                Tables\Columns\TextColumn::make('published_at')
                    ->dateTime()
                    ->sortable(),
            ])
            ->filters([
                Tables\Filters\SelectFilter::make('status')
                    ->options(PostStatus::class),
            ])
            ->actions([
                Tables\Actions\EditAction::make(),
                Tables\Actions\DeleteAction::make(),
            ]);
    }
}
```

## Livewire

PHP ile reaktif UI bileşenleri.

```bash
composer require livewire/livewire
```

### Component Oluşturma
```bash
php artisan make:livewire PostList
```

```php
// app/Livewire/PostList.php
class PostList extends Component
{
    public string $search = '';
    public string $status = '';
    
    protected $queryString = ['search', 'status'];

    public function updatedSearch(): void
    {
        $this->resetPage();
    }

    public function render(): View
    {
        $posts = Post::query()
            ->when($this->search, fn($q) => 
                $q->where('title', 'like', "%{$this->search}%"))
            ->when($this->status, fn($q) => 
                $q->where('status', $this->status))
            ->latest()
            ->paginate(10);

        return view('livewire.post-list', compact('posts'));
    }

    public function delete(Post $post): void
    {
        $this->authorize('delete', $post);
        $post->delete();
        
        $this->dispatch('post-deleted');
    }
}
```

```blade
{{-- resources/views/livewire/post-list.blade.php --}}
<div>
    <div class="mb-4 flex gap-4">
        <input 
            type="text" 
            wire:model.live.debounce.300ms="search" 
            placeholder="Ara..."
            class="border rounded px-4 py-2"
        >
        
        <select wire:model.live="status" class="border rounded px-4 py-2">
            <option value="">Tüm Durumlar</option>
            @foreach(PostStatus::cases() as $status)
                <option value="{{ $status->value }}">{{ $status->label() }}</option>
            @endforeach
        </select>
    </div>

    <div class="space-y-4">
        @foreach($posts as $post)
            <div wire:key="{{ $post->id }}" class="p-4 border rounded">
                <h3>{{ $post->title }}</h3>
                <button 
                    wire:click="delete({{ $post->id }})"
                    wire:confirm="Silmek istediğinize emin misiniz?"
                    class="text-red-600"
                >
                    Sil
                </button>
            </div>
        @endforeach
    </div>

    {{ $posts->links() }}
</div>
```

## Inertia.js

Laravel + Vue/React SPA.

```bash
composer require inertiajs/inertia-laravel
npm install @inertiajs/vue3
```

### Controller
```php
use Inertia\Inertia;
use Inertia\Response;

class PostController extends Controller
{
    public function index(): Response
    {
        return Inertia::render('Posts/Index', [
            'posts' => PostResource::collection(
                Post::with('user')->latest()->paginate(10)
            ),
            'filters' => request()->only(['search', 'status']),
        ]);
    }

    public function create(): Response
    {
        return Inertia::render('Posts/Create', [
            'statuses' => PostStatus::cases(),
        ]);
    }

    public function store(StorePostRequest $request): RedirectResponse
    {
        Post::create($request->validated());

        return redirect()->route('posts.index')
            ->with('success', 'Post oluşturuldu.');
    }
}
```

### Vue Component
```vue
<!-- resources/js/Pages/Posts/Index.vue -->
<script setup>
import { Head, Link, router } from '@inertiajs/vue3'
import { ref, watch } from 'vue'
import debounce from 'lodash/debounce'

const props = defineProps({
    posts: Object,
    filters: Object,
})

const search = ref(props.filters.search ?? '')

watch(search, debounce((value) => {
    router.get('/posts', { search: value }, { preserveState: true })
}, 300))
</script>

<template>
    <Head title="Posts" />
    
    <div class="container mx-auto p-4">
        <input 
            v-model="search" 
            type="text" 
            placeholder="Ara..."
            class="border rounded px-4 py-2 mb-4"
        >
        
        <div v-for="post in posts.data" :key="post.id" class="p-4 border rounded mb-4">
            <Link :href="`/posts/${post.slug}`" class="text-xl font-bold">
                {{ post.title }}
            </Link>
            <p class="text-gray-600">{{ post.author.name }}</p>
        </div>
        
        <!-- Pagination -->
    </div>
</template>
```

## Diğer Popüler Paketler

### Laravel Debugbar
```bash
composer require barryvdh/laravel-debugbar --dev
```

### Laravel IDE Helper
```bash
composer require barryvdh/laravel-ide-helper --dev
php artisan ide-helper:generate
php artisan ide-helper:models
```

### Laravel Excel
```bash
composer require maatwebsite/excel
```

### Laravel Socialite
```bash
composer require laravel/socialite
```

### Laravel Scout (Search)
```bash
composer require laravel/scout
composer require meilisearch/meilisearch-php
```
