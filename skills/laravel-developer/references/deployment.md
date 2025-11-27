# Laravel Deployment Rehberi

## Deployment Platformları

### Laravel Cloud (2025 - En Yeni)
Laravel'in resmi tam yönetimli platformu. Sunucu yönetimi gerektirmez.

```yaml
# cloud.yaml (root dizinde)
id: my-app
name: My Laravel App

environments:
  production:
    build:
      - composer install --no-dev --optimize-autoloader
      - npm ci && npm run build
    workers:
      - php artisan queue:work
    schedule: true
    database: true
    cache: true
```

**Özellikler:**
- Push-to-deploy
- Auto-scaling
- Serverless PostgreSQL
- Yerleşik güvenlik
- Zero-downtime deployment

### Laravel Forge
Server provisioning ve management tool.

```bash
# Forge deployment script örneği
cd /home/forge/example.com
git pull origin $FORGE_SITE_BRANCH

$FORGE_COMPOSER install --no-dev --no-interaction --prefer-dist --optimize-autoloader

( flock -w 10 9 || exit 1
    echo 'Restarting FPM...'; sudo -S service $FORGE_PHP_FPM reload ) 9>/tmp/fpmlock

if [ -f artisan ]; then
    $FORGE_PHP artisan migrate --force
    $FORGE_PHP artisan config:cache
    $FORGE_PHP artisan route:cache
    $FORGE_PHP artisan view:cache
    $FORGE_PHP artisan queue:restart
fi

npm ci
npm run build
```

### Laravel Vapor (Serverless)
AWS Lambda üzerinde serverless deployment.

```yaml
# vapor.yml
id: my-app
name: my-app
environments:
  production:
    memory: 1024
    cli-memory: 512
    runtime: php-8.3:al2
    build:
      - 'composer install --no-dev'
      - 'npm ci && npm run build'
    database: my-database
    cache: my-cache
    queue: my-queue
    storage: my-bucket
    
  staging:
    memory: 512
    build:
      - 'composer install'
      - 'npm ci && npm run build'
```

```bash
# Deploy
vapor deploy production

# Logs
vapor logs production

# Tail
vapor logs production --tail
```

## CI/CD Pipeline

### GitHub Actions
```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  tests:
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_DATABASE: testing
          MYSQL_ROOT_PASSWORD: password
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping" --health-interval=10s --health-timeout=5s --health-retries=3

    steps:
      - uses: actions/checkout@v4
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          extensions: mbstring, pdo, pdo_mysql
          coverage: xdebug

      - name: Install Composer dependencies
        run: composer install --no-progress --prefer-dist --optimize-autoloader

      - name: Copy .env
        run: cp .env.example .env

      - name: Generate key
        run: php artisan key:generate

      - name: Run tests
        env:
          DB_CONNECTION: mysql
          DB_HOST: 127.0.0.1
          DB_PORT: 3306
          DB_DATABASE: testing
          DB_USERNAME: root
          DB_PASSWORD: password
        run: php artisan test --coverage --min=80

      - name: Run Pint
        run: ./vendor/bin/pint --test

  deploy:
    needs: tests
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      # Laravel Forge deployment
      - name: Deploy to Forge
        run: |
          curl -X POST ${{ secrets.FORGE_DEPLOY_URL }}
      
      # veya Laravel Vapor
      # - name: Deploy to Vapor
      #   run: |
      #     composer global require laravel/vapor-cli
      #     vapor deploy production --commit="${GITHUB_SHA}"
      #   env:
      #     VAPOR_API_TOKEN: ${{ secrets.VAPOR_API_TOKEN }}
```

### GitLab CI
```yaml
# .gitlab-ci.yml
stages:
  - test
  - deploy

variables:
  MYSQL_DATABASE: testing
  MYSQL_ROOT_PASSWORD: secret

test:
  stage: test
  image: php:8.3
  services:
    - mysql:8.0
  before_script:
    - apt-get update && apt-get install -y git unzip
    - docker-php-ext-install pdo pdo_mysql
    - curl -sS https://getcomposer.org/installer | php
    - php composer.phar install
    - cp .env.example .env
    - php artisan key:generate
  script:
    - php artisan test
  only:
    - merge_requests
    - main

deploy:
  stage: deploy
  image: alpine
  script:
    - apk add --no-cache curl
    - curl -X POST $FORGE_DEPLOY_URL
  only:
    - main
  when: on_success
```

## Environment Configuration

### Production .env
```env
APP_NAME="My App"
APP_ENV=production
APP_KEY=base64:...
APP_DEBUG=false
APP_URL=https://myapp.com

LOG_CHANNEL=stack
LOG_LEVEL=error

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=myapp
DB_USERNAME=myapp
DB_PASSWORD=secure_password

CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

MAIL_MAILER=smtp
MAIL_HOST=smtp.mailgun.org
MAIL_PORT=587
MAIL_USERNAME=your-username
MAIL_PASSWORD=your-password
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS="hello@myapp.com"
MAIL_FROM_NAME="${APP_NAME}"
```

## Server Requirements

### PHP Extensions
```
- BCMath
- Ctype
- cURL
- DOM
- Fileinfo
- JSON
- Mbstring
- OpenSSL
- PCRE
- PDO
- Tokenizer
- XML
```

### Nginx Configuration
```nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com;
    root /var/www/example.com/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_hide_header X-Powered-By;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }

    # Gzip
    gzip on;
    gzip_comp_level 5;
    gzip_min_length 256;
    gzip_proxied any;
    gzip_vary on;
    gzip_types
        application/javascript
        application/json
        application/xml
        text/css
        text/plain;
}
```

## Production Checklist

### Before Deployment
```bash
# 1. Tests pass
php artisan test

# 2. Code style
./vendor/bin/pint

# 3. Static analysis
./vendor/bin/phpstan analyse

# 4. No debug code
grep -r "dd(" app/ --include="*.php"
grep -r "dump(" app/ --include="*.php"
```

### Deployment Commands
```bash
# 1. Maintenance mode
php artisan down --refresh=15 --secret="secret-token"

# 2. Pull latest code
git pull origin main

# 3. Install dependencies
composer install --no-dev --optimize-autoloader

# 4. Run migrations
php artisan migrate --force

# 5. Clear and cache
php artisan optimize

# 6. Restart queue workers
php artisan queue:restart

# 7. Up
php artisan up
```

### Post-Deployment
```bash
# Check logs
tail -f storage/logs/laravel.log

# Monitor queues
php artisan queue:monitor redis:default,redis:high --max=100

# Health check
curl -I https://myapp.com/health
```

## Zero-Downtime Deployment

### Envoyer Script
```bash
cd {{ release }}

composer install --no-dev --no-interaction --prefer-dist --optimize-autoloader

php artisan migrate --force
php artisan config:cache
php artisan route:cache
php artisan view:cache

npm ci
npm run build
```

### Symlink Strategy
```
/var/www/myapp/
├── releases/
│   ├── 20240101120000/
│   ├── 20240102120000/
│   └── 20240103120000/  # en yeni
├── current -> releases/20240103120000
├── storage/
│   ├── app/
│   ├── framework/
│   └── logs/
└── .env
```

## Monitoring & Logging

### Sentry Integration
```bash
composer require sentry/sentry-laravel
php artisan sentry:publish --dsn=https://xxx@sentry.io/xxx
```

### Health Check Endpoint
```php
// routes/web.php
Route::get('/health', function () {
    try {
        DB::connection()->getPdo();
        Cache::get('health-check');
        
        return response()->json([
            'status' => 'healthy',
            'timestamp' => now()->toIso8601String(),
        ]);
    } catch (\Exception $e) {
        return response()->json([
            'status' => 'unhealthy',
            'error' => $e->getMessage(),
        ], 503);
    }
});
```
