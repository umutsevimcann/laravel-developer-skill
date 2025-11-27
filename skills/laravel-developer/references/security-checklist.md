# Laravel Güvenlik Kontrol Listesi

## 1. Authentication & Authorization

### ✅ Güçlü Şifre Politikası
```php
// app/Providers/AppServiceProvider.php
use Illuminate\Validation\Rules\Password;

public function boot(): void
{
    Password::defaults(function () {
        return Password::min(8)
            ->letters()
            ->mixedCase()
            ->numbers()
            ->symbols()
            ->uncompromised(); // Breach database kontrolü
    });
}

// Validation'da kullan
'password' => ['required', 'confirmed', Password::defaults()],
```

### ✅ Rate Limiting (Brute Force Koruması)
```php
// routes/web.php
Route::post('/login', [AuthController::class, 'login'])
    ->middleware('throttle:5,1'); // 5 deneme/dakika

// Özel limiter
RateLimiter::for('login', function (Request $request) {
    return [
        Limit::perMinute(5)->by($request->input('email')),
        Limit::perMinute(10)->by($request->ip()),
    ];
});
```

### ✅ Session Güvenliği
```php
// config/session.php
'secure' => env('SESSION_SECURE_COOKIE', true),  // HTTPS only
'http_only' => true,                              // JavaScript erişimi yok
'same_site' => 'lax',                            // CSRF koruması
```

### ✅ Password Reset Token Expiry
```php
// config/auth.php
'passwords' => [
    'users' => [
        'provider' => 'users',
        'table' => 'password_reset_tokens',
        'expire' => 60,     // 60 dakika
        'throttle' => 60,   // Yeniden gönderme süresi
    ],
],
```

## 2. Input Validation & Sanitization

### ✅ Request Validation
```php
// Her zaman Form Request kullan
public function store(StorePostRequest $request): JsonResponse
{
    // $request->validated() kullan, $request->all() değil!
    $post = Post::create($request->validated());
}
```

### ✅ Mass Assignment Koruması
```php
// Model'de $fillable tanımla
protected $fillable = ['title', 'content', 'status'];

// veya $guarded kullan
protected $guarded = ['id', 'user_id'];

// ❌ ASLA bunları yapma:
protected $guarded = [];  // Tehlikeli!
$post = Post::create($request->all());  // Tehlikeli!
```

### ✅ File Upload Güvenliği
```php
public function upload(Request $request): JsonResponse
{
    $request->validate([
        'file' => [
            'required',
            'file',
            'max:10240',  // Max 10MB
            'mimes:pdf,doc,docx,jpg,png',  // İzin verilen tipler
        ],
    ]);

    // Orijinal dosya adını kullanma!
    $path = $request->file('file')->store('uploads', 'private');
    
    // Dosya tipi doğrulama (magic bytes)
    $mimeType = $request->file('file')->getMimeType();
}
```

## 3. SQL Injection Koruması

### ✅ Eloquent & Query Builder Kullan
```php
// ✅ Güvenli
User::where('email', $email)->first();
DB::table('users')->where('email', '=', $email)->first();

// ✅ Prepared statements
DB::select('SELECT * FROM users WHERE email = ?', [$email]);

// ❌ Tehlikeli - Asla yapma!
DB::select("SELECT * FROM users WHERE email = '$email'");
User::whereRaw("email = '$email'")->first();
```

### ✅ LIKE Sorguları
```php
// ✅ Güvenli LIKE
$search = '%' . $request->search . '%';
Post::where('title', 'like', $search)->get();

// ✅ Daha güvenli (escape)
$search = '%' . DB::raw($request->search) . '%';
```

## 4. XSS (Cross-Site Scripting) Koruması

### ✅ Blade Escape
```blade
{{-- ✅ Otomatik escape --}}
{{ $userInput }}
{{ $user->name }}

{{-- ❌ Escape edilmemiş - dikkatli kullan --}}
{!! $trustedHtml !!}

{{-- ✅ HTML gerekiyorsa sanitize et --}}
{!! clean($userInput) !!}  // HTMLPurifier
{!! Str::markdown($content) !!}  // Markdown safe
```

### ✅ JSON Output
```php
// Controller'da
return response()->json($data);  // Otomatik encode

// Blade'de
<script>
    const data = @json($data);  // Güvenli
</script>
```

### ✅ Content Security Policy
```php
// Middleware ile CSP header
public function handle($request, Closure $next)
{
    $response = $next($request);
    
    $response->headers->set(
        'Content-Security-Policy',
        "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'"
    );
    
    return $response;
}
```

## 5. CSRF Koruması

### ✅ Form'larda Token
```blade
<form method="POST" action="/posts">
    @csrf
    <!-- form fields -->
</form>
```

### ✅ AJAX İstekleri
```javascript
// Axios otomatik ekler (Laravel default setup)
axios.defaults.headers.common['X-CSRF-TOKEN'] = document.querySelector('meta[name="csrf-token"]').content;

// Manuel jQuery
$.ajaxSetup({
    headers: {
        'X-CSRF-TOKEN': $('meta[name="csrf-token"]').attr('content')
    }
});
```

### ✅ API Routes için Exclude
```php
// API routes zaten stateless, CSRF gerekmez
// routes/api.php middleware grubunda 'web' yok
```

## 6. Security Headers

```php
// app/Http/Middleware/SecurityHeaders.php
class SecurityHeaders
{
    public function handle($request, Closure $next)
    {
        $response = $next($request);
        
        $response->headers->set('X-Frame-Options', 'SAMEORIGIN');
        $response->headers->set('X-Content-Type-Options', 'nosniff');
        $response->headers->set('X-XSS-Protection', '1; mode=block');
        $response->headers->set('Referrer-Policy', 'strict-origin-when-cross-origin');
        $response->headers->set('Permissions-Policy', 'geolocation=(), microphone=()');
        
        if ($request->secure()) {
            $response->headers->set(
                'Strict-Transport-Security',
                'max-age=31536000; includeSubDomains'
            );
        }
        
        return $response;
    }
}
```

## 7. Sensitive Data Koruması

### ✅ Env Değişkenleri
```php
// .env dosyasını ASLA commit etme!
// .env.example'da gerçek değerler olmasın

// config dosyalarında env() kullan
'api_key' => env('API_KEY'),
```

### ✅ Log'larda Hassas Veri
```php
// config/logging.php - sensitive data maskeleme
'tap' => [App\Logging\SensitiveDataProcessor::class],

// veya model'de $hidden kullan
protected $hidden = ['password', 'remember_token', 'api_key'];

// Log'lamada dikkat
Log::info('User logged in', ['user_id' => $user->id]); // ✅
Log::info('User logged in', $user->toArray()); // ❌ password içerebilir
```

### ✅ Database Encryption
```php
// Hassas alanları encrypt et
protected function casts(): array
{
    return [
        'ssn' => 'encrypted',
        'api_token' => 'encrypted',
    ];
}
```

## 8. API Güvenliği

### ✅ Token Expiration
```php
// Sanctum token expiration
'expiration' => 60 * 24 * 7, // 7 gün

// Manuel token oluşturma
$token = $user->createToken(
    'api-token',
    ['*'],
    now()->addDays(7)  // Expiration
);
```

### ✅ API Rate Limiting
```php
RateLimiter::for('api', function (Request $request) {
    return Limit::perMinute(60)
        ->by($request->user()?->id ?: $request->ip())
        ->response(function () {
            return response()->json([
                'message' => 'Too many requests',
            ], 429);
        });
});
```

### ✅ CORS Konfigürasyonu
```php
// config/cors.php
'paths' => ['api/*'],
'allowed_origins' => ['https://trusted-domain.com'],
'allowed_methods' => ['GET', 'POST', 'PUT', 'DELETE'],
'allowed_headers' => ['Content-Type', 'Authorization'],
'supports_credentials' => true,
```

## 9. Production Güvenliği

### ✅ Debug Mode Kapalı
```env
APP_DEBUG=false
APP_ENV=production
```

### ✅ Error Handling
```php
// Handler.php - Production'da detay gösterme
public function register(): void
{
    $this->reportable(function (Throwable $e) {
        // Sentry/Bugsnag'a raporla
    });
}

// API response'larda detay verme
if (app()->environment('production')) {
    return response()->json(['message' => 'Server error'], 500);
}
```

### ✅ HTTPS Zorunlu
```php
// AppServiceProvider.php
public function boot(): void
{
    if (app()->environment('production')) {
        URL::forceScheme('https');
    }
}

// .htaccess veya nginx ile redirect
```

## 10. Dependency Güvenliği

### ✅ Güvenlik Güncellemeleri
```bash
# Güvenlik açıkları kontrolü
composer audit

# Güncellemeleri kontrol et
composer outdated

# Güvenlik güncellemeleri
composer update --prefer-stable
```

### ✅ Lock Dosyası
```bash
# composer.lock'u commit et
# Exact versions kullan
```

## Güvenlik Checklist Özet

```markdown
[ ] Strong password policy
[ ] Rate limiting on auth routes
[ ] Secure session configuration
[ ] Input validation with Form Requests
[ ] Mass assignment protection
[ ] Secure file uploads
[ ] Eloquent/Query Builder (no raw SQL with user input)
[ ] Blade escape ({{ }} not {!! !!})
[ ] CSRF tokens on forms
[ ] Security headers
[ ] Sensitive data hidden in logs
[ ] API token expiration
[ ] Rate limiting on API
[ ] CORS properly configured
[ ] APP_DEBUG=false in production
[ ] HTTPS enforced
[ ] Dependencies audited
[ ] .env not in version control
```
