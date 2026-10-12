# Lesson 46 — Docker for PHP Applications

> **Phase 6: Deploy & Scale** | You'll be here: ~40 min

## The Senior's Question

"It works on my machine." The junior has a different PHP version, different MySQL, different Redis. **The senior containers everything.** Docker makes "works on my machine" and "works on prod" the same. Let me show you how.

---

## Theory: What Docker Solves

Without Docker:
```
Dev A: macOS, PHP 8.3, MySQL 8, Redis 7
Dev B: Windows, PHP 8.2, MySQL 5.7, Redis 6
Prod:  Ubuntu 22.04, PHP 8.3, MySQL 8, Redis 7

"Works on mine" = doesn't work on prod
```

With Docker:
```
Dev A: Docker container, PHP 8.3, MySQL 8, Redis 7
Dev B: Docker container, PHP 8.3, MySQL 8, Redis 7
Prod:  Docker container, PHP 8.3, MySQL 8, Redis 7

Same. Everywhere.
```

> **Senior rule:** Docker is not about "containers are cool." It's about **environment reproducibility.** A new dev clones, runs `docker-compose up`, has the same env as prod. Period.**

---

## Theory: Docker Concepts

### Image

A snapshot. Read-only. Like a `.iso` for a VM but lighter.

Examples: `php:8.3-cli`, `mysql:8.0`, `redis:7-alpine`.

### Container

A running instance of an image. You can have many containers from one image.

### Dockerfile

A recipe to build an image. Instructions for what to install, what to copy, what to run.

### docker-compose

A way to run multiple containers together. Your app + MySQL + Redis + Meilisearch, all in one file.

### Volume

Persistent storage. Containers are ephemeral; volumes keep data.

### Network

Containers can talk to each other. The `app` container reaches `mysql` by name.

---

## Theory: The Dockerfile for Laravel

`Dockerfile`:

```dockerfile
# Use official PHP image
FROM php:8.3-fpm-alpine

# Install system dependencies
RUN apk add --no-cache \
    git \
    curl \
    libpng-dev \
    libzip-dev \
    zip \
    unzip \
    nginx \
    supervisor

# Install PHP extensions
RUN docker-php-ext-install pdo_mysql gd zip bcmath opcache

# Install Composer
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# Set working directory
WORKDIR /var/www/html

# Copy composer files first (better caching)
COPY composer.json composer.lock ./
RUN composer install --no-dev --no-scripts --no-autoloader --prefer-dist

# Copy application code
COPY . .

# Generate autoloader
RUN composer dump-autoload --optimize

# Set permissions
RUN chown -R www-data:www-data /var/www/html/storage /var/www/html/bootstrap/cache

EXPOSE 9000
CMD ["php-fpm"]
```

> **Multi-stage builds** let you keep the final image small. Build with one image, run with another.

### Smaller production image

```dockerfile
# Build stage
FROM php:8.3-cli AS builder
# ... build stuff

# Production stage
FROM php:8.3-fpm-alpine
COPY --from=builder /app /app
# ...
```

---

## Theory: Nginx + PHP-FPM

In production, you run two containers:

- `app` — PHP-FPM, processes Laravel
- `nginx` — serves static files, proxies to PHP-FPM

`docker/nginx/default.conf`:

```nginx
server {
    listen 80;
    server_name _;
    root /var/www/html/public;
    
    index index.php;
    
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    
    location ~ \.php$ {
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
    
    location ~ /\.ht {
        deny all;
    }
}
```

> **`fastcgi_pass app:9000`** — `app` is the service name. Docker DNS resolves it.

---

## Theory: docker-compose.yml

This is the senior's "everything in one file":

`docker-compose.yml`:

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    image: coaching-saas-app
    container_name: coaching-app
    restart: unless-stopped
    working_dir: /var/www/html
    volumes:
      - ./:/var/www/html
      - ./docker/php/local.ini:/usr/local/etc/php/conf.d/local.ini
    networks:
      - coaching
    depends_on:
      - mysql
      - redis

  nginx:
    image: nginx:alpine
    container_name: coaching-nginx
    restart: unless-stopped
    ports:
      - "8000:80"
    volumes:
      - ./:/var/www/html
      - ./docker/nginx:/etc/nginx/conf.d
    networks:
      - coaching
    depends_on:
      - app

  mysql:
    image: mysql:8.0
    container_name: coaching-mysql
    restart: unless-stopped
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: coaching
      MYSQL_USER: coaching
      MYSQL_PASSWORD: secret
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - coaching

  redis:
    image: redis:7-alpine
    container_name: coaching-redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - coaching

  meilisearch:
    image: getmeili/meilisearch:v1.6
    container_name: coaching-meilisearch
    restart: unless-stopped
    ports:
      - "7700:7700"
    environment:
      MEILI_MASTER_KEY: masterKey
    volumes:
      - meili_data:/meili_data
    networks:
      - coaching

  queue:
    image: coaching-saas-app
    container_name: coaching-queue
    restart: unless-stopped
    command: php artisan queue:work --verbose --tries=3 --timeout=90
    volumes:
      - ./:/var/www/html
    networks:
      - coaching
    depends_on:
      - app
      - redis

  scheduler:
    image: coaching-saas-app
    container_name: coaching-scheduler
    restart: unless-stopped
    command: sh -c "while true; do php artisan schedule:run; sleep 60; done"
    volumes:
      - ./:/var/www/html
    networks:
      - coaching
    depends_on:
      - app

volumes:
  mysql_data:
  redis_data:
  meili_data:

networks:
  coaching:
    driver: bridge
```

> **This file replaces 5 separate install steps.** A new dev runs `docker-compose up`, gets everything.

---

## Theory: PHP Configuration

`docker/php/local.ini`:

```ini
upload_max_filesize = 40M
post_max_size = 40M
memory_limit = 512M
max_execution_time = 60
```

> **Mounted into the container at `/usr/local/etc/php/conf.d/local.ini`.** Override defaults without rebuilding the image.

---

## Theory: Running It

### Start

```bash
docker-compose up -d
```

`-d` = detached (background). `docker-compose ps` to see status.

### Stop

```bash
docker-compose down
```

### Logs

```bash
docker-compose logs -f app
docker-compose logs -f queue
```

### Shell into a container

```bash
docker-compose exec app bash
docker-compose exec app php artisan migrate
docker-compose exec app php artisan tinker
```

### Run a one-off command

```bash
docker-compose run --rm app php artisan scout:import "App\Models\Student"
```

### Rebuild after Dockerfile changes

```bash
docker-compose up -d --build
```

---

## Theory: Multi-Environment Compose Files

### Base (docker-compose.yml)

Shared by all environments.

### Override (docker-compose.override.yml)

Local dev only. Auto-loaded.

```yaml
# docker-compose.override.yml
services:
  app:
    environment:
      APP_ENV: local
      APP_DEBUG: "true"
    volumes:
      - ./:/var/www/html  # live code reload
```

### Production (docker-compose.prod.yml)

```yaml
services:
  app:
    environment:
      APP_ENV: production
      APP_DEBUG: "false"
    volumes:
      - .:/var/www/html:ro  # read-only
  
  nginx:
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./docker/nginx/prod.conf:/etc/nginx/conf.d/default.conf
      - ./docker/certs:/etc/nginx/certs
```

Run with:

```bash
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

> **Senior rule:** the same `docker-compose.yml` works in dev and prod.** Override for the differences. **Same images, different config.**

---

## Theory: Persistent Data

Containers are ephemeral. **Volumes persist.**

```yaml
volumes:
  mysql_data:
    driver: local
```

This creates a named volume. Data survives `docker-compose down`. To wipe data: `docker-compose down -v`.

> **Senior rule:** always use named volumes for databases.** Don't bind-mount `/var/lib/mysql` to the host — it's slow and has permission issues.

---

## Theory: The `.dockerignore` File

`.dockerignore`:

```
.git
node_modules
vendor
.env
.env.*
storage/logs/*
storage/framework/cache/*
storage/framework/sessions/*
storage/framework/views/*
tests
.phpunit.result.cache
README.md
```

> **Speeds up builds.** Docker doesn't copy what it doesn't need. Smaller context = faster builds.

---

## Theory: Layer Caching

Docker caches each layer. Order matters:

```dockerfile
# BAD: invalidates cache on every code change
COPY . .
RUN composer install

# GOOD: cache deps layer, only changes invalidate when composer.json changes
COPY composer.json composer.lock ./
RUN composer install
COPY . .
```

> **Senior rule:** put the stuff that changes least at the top.** System deps, then PHP deps, then your code. **Code changes don't bust the cache for `composer install`.**

---

## Theory: Production Builds

In CI/CD, you build the image once and push to a registry:

```bash
# Build
docker build -t yourname/coaching-saas:abc123 .

# Push
docker push yourname/coaching-saas:abc123
```

In production, you pull and run:

```bash
docker pull yourname/coaching-saas:abc123
docker-compose up -d
```

> **Senior rule:** images are immutable.** Tag with git SHA, not `latest`. **Reproducible, auditable, rollbackable.**

---

## Theory: Docker in CI/CD

GitHub Actions example:

```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          push: true
          tags: yourname/coaching-saas:${{ github.sha }}
```

> **Image is built once, deployed everywhere.** Dev, staging, prod all use the same artifact.

---

## Theory: When to Use Docker vs Forge

| Use case | Recommendation |
|---|---|
| Solo dev, simple SaaS | Forge (no Docker) |
| Small team, same env | Docker + Forge or Docker alone |
| Microservices | Docker (mandatory) |
| Multiple envs (dev/staging/prod) | Docker (easier parity) |
| Want easy onboarding | Docker (clone, up, done) |
| Want to learn k8s later | Docker (foundation) |

> **Senior rule:** Docker is the lingua franca.** Even if you use Forge today, knowing Docker means you can switch to ECS, GKE, Fargate, or any other platform.

---

## Theory: Common Docker Commands

```bash
# List running containers
docker ps

# List images
docker images

# Stop a container
docker stop coaching-app

# Remove a container
docker rm coaching-app

# Remove an image
docker rmi coaching-saas-app

# View logs
docker logs -f coaching-app

# Stats (CPU, memory)
docker stats

# Disk usage
docker system df

# Clean up unused
docker system prune
```

> **Senior tip:** `docker system df` shows what's using disk.** Old images, dangling volumes. Clean up monthly.

---

## Theory: Security in Docker

### Run as non-root user

```dockerfile
RUN addgroup -g 1000 -S app && adduser -u 1000 -S app -G app
USER app
```

### Use official images

```dockerfile
FROM php:8.3-fpm-alpine  # official, patched
```

Don't use random Docker Hub images for production.

### Scan for vulnerabilities

```bash
docker scout cves coaching-saas-app
# or use Snyk, Trivy
```

### Don't put secrets in the image

Use `docker-compose` env vars or Docker secrets. **Never `ENV STRIPE_SECRET=xxx` in the Dockerfile.**

---

## Why It Matters

Docker gives you:
- **Reproducibility** — dev, staging, prod are identical
- **Portability** — run anywhere Docker runs
- **Isolation** — apps don't conflict
- **Onboarding speed** — new dev is productive in 5 minutes
- **Deployment confidence** — same artifact everywhere

> **A senior's Docker setup is boring, reproducible, and well-cached.** A junior's "Docker is too complex" is "I'll debug prod at 3 AM."

---

## Lab: Dockerize Your Coaching SaaS

### Step 1 — Install Docker

- **macOS:** [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- **Windows:** [Docker Desktop with WSL2](https://www.docker.com/products/docker-desktop/)
- **Linux:** `curl -fsSL https://get.docker.com | sh`

Verify:

```bash
docker --version
docker-compose --version
```

### Step 2 — Create the Dockerfile

`G:\php-laravel\coaching-saas\Dockerfile`:

```dockerfile
FROM php:8.3-fpm-alpine

RUN apk add --no-cache \
    git curl libpng-dev libzip-dev zip unzip nginx supervisor \
    && docker-php-ext-install pdo_mysql gd zip bcmath opcache

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html

COPY composer.json composer.lock ./
RUN composer install --no-dev --no-scripts --no-autoloader --prefer-dist

COPY . .

RUN composer dump-autoload --optimize \
    && chown -R www-data:www-data storage bootstrap/cache

EXPOSE 9000
CMD ["php-fpm"]
```

### Step 3 — Create the nginx config

`G:\php-laravel\coaching-saas\docker\nginx\default.conf`:

```nginx
server {
    listen 80;
    server_name _;
    root /var/www/html/public;
    
    index index.php;
    charset utf-8;
    
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    
    location ~ \.php$ {
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
    
    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

### Step 4 — Create docker-compose.yml

`G:\php-laravel\coaching-saas\docker-compose.yml`:

```yaml
version: '3.8'

services:
  app:
    build: .
    image: coaching-saas-app
    container_name: coaching-app
    working_dir: /var/www/html
    volumes:
      - ./:/var/www/html
      - ./docker/php/local.ini:/usr/local/etc/php/conf.d/local.ini
    networks:
      - coaching
    depends_on:
      - mysql
      - redis

  nginx:
    image: nginx:alpine
    container_name: coaching-nginx
    ports:
      - "8000:80"
    volumes:
      - ./:/var/www/html
      - ./docker/nginx:/etc/nginx/conf.d
    networks:
      - coaching
    depends_on:
      - app

  mysql:
    image: mysql:8.0
    container_name: coaching-mysql
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: coaching
      MYSQL_USER: coaching
      MYSQL_PASSWORD: secret
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - coaching

  redis:
    image: redis:7-alpine
    container_name: coaching-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - coaching

  queue:
    image: coaching-saas-app
    container_name: coaching-queue
    command: php artisan queue:work --verbose --tries=3 --timeout=90
    volumes:
      - ./:/var/www/html
    networks:
      - coaching
    depends_on:
      - app
      - redis

volumes:
  mysql_data:
  redis_data:

networks:
  coaching:
    driver: bridge
```

### Step 5 — Create PHP config override

`G:\php-laravel\coaching-saas\docker\php\local.ini`:

```ini
upload_max_filesize = 40M
post_max_size = 40M
memory_limit = 512M
```

### Step 6 — Create .dockerignore

`G:\php-laravel\coaching-saas\.dockerignore`:

```
.git
node_modules
vendor
.env
.env.*
storage/logs/*
storage/framework/cache/*
storage/framework/sessions/*
storage/framework/views/*
tests
.phpunit.result.cache
README.md
```

### Step 7 — Update `.env`

```env
APP_ENV=local
APP_DEBUG=true
APP_URL=http://localhost:8000

DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=coaching
DB_USERNAME=coaching
DB_PASSWORD=secret

REDIS_HOST=redis
REDIS_PASSWORD=null
REDIS_PORT=6379

CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis

MAIL_MAILER=log
```

> **`DB_HOST=mysql`**, `REDIS_HOST=redis` — these are the Docker service names. Docker DNS resolves them.

### Step 8 — Start the stack

```bash
cd G:\php-laravel\coaching-saas
docker-compose up -d --build
```

Wait for build to complete (5-10 minutes first time).

### Step 9 — Run migrations

```bash
docker-compose exec app php artisan migrate
```

### Step 10 — Open the app

Visit `http://localhost:8000`. You should see Laravel's welcome page.

### Step 11 — Test artisan commands

```bash
docker-compose exec app php artisan tinker
>>> User::count()
=> 0

docker-compose exec app php artisan scout:import "App\Models\Student"
```

### Step 12 — Check the queue

```bash
docker-compose logs -f queue
```

### Step 13 — Add a production override

`docker-compose.prod.yml`:

```yaml
services:
  app:
    environment:
      APP_ENV: production
      APP_DEBUG: "false"
    volumes:
      - .:/var/www/html:ro
  
  nginx:
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./docker/nginx/prod.conf:/etc/nginx/conf.d/default.conf
```

### Step 14 — Stop the stack

```bash
docker-compose down
```

Or to wipe data:

```bash
docker-compose down -v
```

---

## Quiz

**Q1.** Why put `COPY composer.json composer.lock ./` BEFORE `COPY . .` in the Dockerfile? What does it optimize?

**Q2.** You have `APP_DEBUG=true` in dev and want to ensure it stays `false` in production. How does Docker help enforce this?

**Q3.** A junior bind-mounts MySQL data to `./data/mysql` on the host. Why is this a bad idea?

**Q4.** Your Dockerfile runs as root. A security audit flags this. Walk through the fix.

**Q5.** (Stretch) You have 3 services: `app`, `queue`, `scheduler`. They all need the same code. How does Docker keep them in sync? How do you update them all on deploy?

---

## What You Now Have

✅ Dockerfile for Laravel
✅ Multi-container docker-compose
✅ Nginx + PHP-FPM setup
✅ Persistent volumes
✅ Local dev parity with prod
✅ Production override pattern
✅ Layer caching knowledge

## What's Next

Lesson 47 covers **queues and workers in production** — running them reliably, monitoring them, scaling them. The senior's queue ops.

📖 **[Lesson 47 — Queues and Workers in Production](47-queues-and-workers-in-production.md)**
