# Lesson 50 — CDN, Static Assets, Edge Caching

> **Phase 6: Deploy & Scale** | You'll be here: ~25 min

## The Senior's Question

Your app loads in 200ms from Virginia. From Singapore, it takes 2 seconds. The junior blames the user's internet. The senior puts a **CDN in front** — and the page loads in 200ms globally. Let me show you how.

---

## Theory: What a CDN Does

```
Without CDN:
  User in Tokyo → Your server in Virginia (2s round trip)
  
With CDN:
  User in Tokyo → Tokyo edge server (20ms) → Your server in Virginia (only for cache misses)
```

A CDN (Content Delivery Network) is a **globally distributed cache.** It has edge servers in 200+ cities. When a user requests a file, the CDN serves it from the nearest edge.

> **Senior rule:** static assets (CSS, JS, images, fonts) MUST be on a CDN. Always. Non-negotiable.**

---

## Theory: What to Put on a CDN

✅ **Always on CDN:**
- CSS files
- JavaScript files
- Images (logo, photos, icons)
- Fonts
- Videos
- PDFs (invoices, reports)
- Public files

❌ **Don't put on CDN:**
- API responses with sensitive data
- HTML pages (or with care — see below)
- Authenticated content

> **The senior's default: ALL static assets on CDN. HTML pages rendered on origin, then served from CDN edge with cache.**

---

## Theory: CDN Providers

| Provider | Price | Egress | Best for |
|---|---|---|---|
| **Cloudflare** | Free - $200/mo | Free (unlimited) | Most apps |
| **CloudFront** (AWS) | $0.085/GB | Pay per use | AWS deployments |
| **Fastly** | $50/mo+ | Pay per use | Real-time purging |
| **BunnyCDN** | $0.01/GB | Cheap | Budget |
| **KeyCDN** | $0.04/GB | Pay per use | Cheap, simple |
| **Akamai** | Enterprise | Enterprise | Massive scale |

> **Senior default:** Cloudflare. **Free tier is excellent. Unlimited egress. Free SSL. Free DDoS protection. Global edge network.**

---

## Theory: Setting Up Cloudflare

### Step 1 — Sign up

https://cloudflare.com — free plan.

### Step 2 — Add your domain

Cloudflare scans your DNS. Replace nameservers at your registrar.

### Step 3 — Configure DNS

Point your domain at your server:

```
A    coaching-saas.com    → 192.0.2.1
A    www                  → 192.0.2.1
```

> **Cloudflare now proxies your traffic.** Visitors hit Cloudflare first, then your server.

### Step 4 — Enable caching

Cloudflare → Caching → Configuration:

- **Caching Level:** Standard
- **Browser Cache TTL:** 4 hours (or respect existing headers)

### Step 5 — Page rules

Cloudflare → Rules → Page Rules:

```
URL: *coaching-saas.com/static/*
  Cache Level: Cache Everything
  Edge Cache TTL: 1 month
  Browser Cache TTL: 1 month

URL: *coaching-saas.com/api/*
  Cache Level: Bypass
```

> **Static assets: cache for a month. API: never cache.**

---

## Theory: Asset Bundling in Laravel

Laravel's `Vite` (default) bundles your assets:

```javascript
// vite.config.js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            input: ['resources/css/app.css', 'resources/js/app.js'],
            refresh: true,
        }),
    ],
});
```

Build:

```bash
npm run build
```

Output in `public/build/`:

```
public/build/
├── assets/
│   ├── app-abc123.css
│   └── app-def456.js
└── manifest.json
```

In Blade:

```blade
@vite(['resources/css/app.css', 'resources/js/app.js'])
```

> **Vite** generates a manifest. The actual filenames are hashed (`app-abc123.css`). When the file changes, the hash changes. **Cache forever.**

---

## Theory: Long-Term Browser Caching

In `public/.htaccess` (or Nginx config):

```nginx
location ~* \.(js|css|png|jpg|jpeg|gif|svg|woff|woff2|ttf|eot)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}
```

> **`immutable`** tells the browser: "this will never change, don't even revalidate." **Most efficient cache.**

The trick: hash the filename when content changes. So `app.css` becomes `app.abc123.css`. The hash is in the URL; if the URL is the same, the content is the same.

Laravel's Vite does this automatically.

---

## Theory: Image Optimization

Images are often 80% of page weight. Optimize them.

### On upload

```php
public function uploadAvatar(Request $request)
{
    $image = Image::make($request->file('avatar'))
        ->fit(800, 800)
        ->encode('webp', 85);  // WebP is 30% smaller than JPEG
    
    Storage::disk('s3')->put($path, $image);
}
```

> **WebP** is 30% smaller than JPEG with same quality. Supported by all modern browsers.

### On delivery

Use `srcset` for responsive images:

```html
<img srcset="
    avatar-400.webp 400w,
    avatar-800.webp 800w,
    avatar-1200.webp 1200w"
    sizes="(max-width: 600px) 400px, 800px"
    src="avatar-800.webp"
    alt="...">
```

Or use a service:

- **Cloudflare Images** — automatic resize, format conversion, on-demand
- **imgix** — paid but excellent
- **Cloudinary** — generous free tier

> **Senior rule:** serve images at the right size.** A 50KB thumbnail is better than a 5MB full-size image.

---

## Theory: Frontend Performance

### Minify CSS/JS

Vite does this in production. Minified = smaller files = faster loads.

### Code splitting

```javascript
// Only load Vue on pages that need it
const Vue = await import('vue');
```

### Defer non-critical JS

```html
<script src="app.js" defer></script>
```

### Critical CSS inline

```html
<style>
    /* Above-the-fold CSS only */
</style>
```

### Preconnect to external resources

```html
<link rel="preconnect" href="https://cdn.example.com">
<link rel="dns-prefetch" href="https://api.stripe.com">
```

> **Senior rule:** measure first.** Use Lighthouse, WebPageTest, or Chrome DevTools. **Optimize what's actually slow.**

---

## Theory: Edge Caching HTML Pages

The hardest part: caching HTML. You need to be careful about authenticated content.

### Cloudflare Cache Everything for public pages

```html
<!-- Marketing pages, blog, docs -->
URL: *coaching-saas.com/about
  Cache Level: Cache Everything
  Edge Cache TTL: 1 day
```

### Bypass for authenticated

```html
URL: *coaching-saas.com/dashboard*
  Cache Level: Bypass
  Cookie: logged_in
```

### Vary by cookie (multi-tenancy)

For multi-tenant SaaS, you can cache HTML per tenant:

```
URL: *coaching-saas.com/dashboard
  Cache Key: include tenant_id_cookie
```

> **HTML caching is advanced.** Start without it, optimize the rest first. **Don't cache dynamic, user-specific HTML unless you know what you're doing.**

---

## Theory: Brotli Compression

Brotli compresses text files 20-30% better than gzip.

Cloudflare: Brotli on by default.
CloudFront: enable in behavior.

Result: smaller files, faster loads.

> **Senior rule:** use Brotli.** It's free and faster. No reason not to.**

---

## Theory: HTTP/2 and HTTP/3

HTTP/2 multiplexes requests (one connection, many requests). HTTP/3 (QUIC) is even faster.

Cloudflare and most CDNs support both automatically.

> **Make sure your origin server supports HTTP/2.** Nginx 1.25+ and most modern setups do.

---

## Theory: The Cache Hierarchy

```
Browser cache (fastest, smallest)
   ↓ miss
CDN edge (fast, regional)
   ↓ miss
Origin server (slowest, biggest)
```

A request to a cached static asset:

- Browser has it: 0ms
- Browser doesn't, CDN has it: 20ms
- Neither has it: 200ms (CDN fetches from origin, caches, returns)

> **The senior's goal: serve from the browser cache.** That's 0ms. CDN is the second-best.

Set long cache headers + content-hashed filenames = browser caches forever.

---

## Theory: Cloudflare for Security

Beyond speed, Cloudflare provides:

- ✅ **DDoS protection** — absorbs attacks
- ✅ **WAF** (Web Application Firewall) — block common attacks
- ✅ **Bot protection** — block bad bots
- ✅ **SSL** — free, auto-renewing
- ✅ **Rate limiting** — prevent abuse
- ✅ **Analytics** — see traffic, threats, performance

> **Cloudflare's free tier is essentially required for any production app.** Not just for speed — for security.

---

## Theory: The Senior's Caching Setup

For a coaching SaaS:

```
DNS: Cloudflare (free)
SSL: Cloudflare (free, auto)
Static assets: Cloudflare cache
Images: S3 + Cloudflare cache
HTML: Origin server, no edge cache (yet)
API: Origin server, no edge cache
DDoS: Cloudflare (free)
WAF: Cloudflare (free tier basics)
Rate limiting: Cloudflare (free)
```

> **This is free or near-free.** The senior uses these defaults.

---

## Theory: Performance Budget

Set targets:

```
Page load: < 2s (3G), < 500ms (broadband)
Time to First Byte (TTFB): < 200ms
Largest Contentful Paint (LCP): < 2.5s
First Input Delay (FID): < 100ms
Cumulative Layout Shift (CLS): < 0.1
Total page size: < 1MB
```

Lighthouse measures these. Set as goals in CI.

> **Senior rule:** measure in CI, not just locally.** Slow page on a fast laptop isn't a real measurement. **Use Lighthouse CI.**

---

## Theory: When CDN Doesn't Help

- **Authenticated dashboards** — already bypassed
- **Real-time data** — chat, live sports scores
- **Personalized content** — different per user
- **API responses** — need to be fresh

> **Senior rule:** CDN is for static or cacheable content. Dynamic content stays on origin.**

For "real-time" UX, use WebSockets or SSE (Server-Sent Events).

---

## Why It Matters

CDN + asset optimization is the difference between:
- Page load 2s vs 200ms
- $0 bandwidth costs vs $500/mo
- Site dies under attack vs absorbs it
- Global users happy vs users in Asia quit

> **A senior's app is fast everywhere, secure by default, and bandwidth-cheap.** A junior's app is fast in their country, vulnerable, and expensive.

---

## Lab: Set Up Cloudflare and Optimize Assets

### Step 1 — Sign up for Cloudflare

https://cloudflare.com — free plan.

### Step 2 — Add your domain

Follow the wizard. Cloudflare scans DNS records.

### Step 3 — Update nameservers

At your domain registrar, point to Cloudflare's nameservers. Wait 24-48h for propagation.

### Step 4 — Configure DNS

Cloudflare → DNS → Records:

```
A    coaching-saas.com    → your server IP    Proxied
A    www                  → your server IP    Proxied
```

> **"Proxied" = traffic goes through Cloudflare.** Orange cloud icon.

### Step 5 — Configure SSL/TLS

Cloudflare → SSL/TLS:

- **Encryption mode:** Full (Strict)
- **Always Use HTTPS:** On
- **Minimum TLS Version:** 1.2
- **Automatic HTTPS Rewrites:** On

### Step 6 — Set up caching

Cloudflare → Caching → Configuration:

- **Caching Level:** Standard
- **Browser Cache TTL:** Respect Existing Headers

### Step 7 — Add page rules for static assets

Cloudflare → Rules → Page Rules:

```
URL: *coaching-saas.com/build/*
  Cache Level: Cache Everything
  Edge Cache TTL: 1 month
  Browser Cache TTL: 1 month

URL: *coaching-saas.com/storage/*
  Cache Level: Cache Everything
  Edge Cache TTL: 1 week
  Browser Cache TTL: 1 week
```

### Step 8 — Configure Nginx to send long cache headers

In `/etc/nginx/sites-available/coaching-saas.com`:

```nginx
location ~* \.(js|css|png|jpg|jpeg|gif|svg|webp|woff|woff2|ttf|eot)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}

location /build/ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}

location /storage/ {
    expires 1w;
    add_header Cache-Control "public";
}
```

Reload Nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### Step 9 — Enable Brotli

Cloudflare → Speed → Optimization:

- **Brotli:** On
- **Auto Minify:** HTML, CSS, JS (all on)
- **Rocket Loader:** Off (can break things)

### Step 10 — Optimize images

In Laravel, when uploading:

```bash
composer require intervention/image
```

```php
public function upload(Request $request)
{
    $image = Image::make($request->file('image'))
        ->orientate()
        ->resize(1200, null, fn($c) => $c->aspectRatio())
        ->encode('webp', 85);
    
    Storage::disk('s3')->put($path, $image);
}
```

### Step 11 — Use Vite for assets

Verify `vite.config.js`:

```javascript
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            input: ['resources/css/app.css', 'resources/js/app.js'],
            refresh: true,
        }),
    ],
});
```

Build:

```bash
npm run build
```

### Step 12 — Use lazy loading for images

```html
<img src="photo.webp" loading="lazy" alt="...">
```

> **`loading="lazy"`** defers off-screen images. Faster initial load.

### Step 13 — Add preconnect hints

In `resources/views/layouts/app.blade.php`:

```html
<head>
    <link rel="preconnect" href="https://coaching-saas-prod.s3.amazonaws.com" crossorigin>
    <link rel="dns-prefetch" href="https://hooks.stripe.com">
    
    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
```

### Step 14 — Test with Lighthouse

Open Chrome DevTools → Lighthouse → Analyze page load.

Targets:
- Performance: 90+
- Accessibility: 90+
- Best Practices: 90+
- SEO: 90+

Fix what it flags. **Senior iterates until 90+.**

### Step 15 — Add security headers

In `app/Http/Middleware/SecurityHeaders.php`:

```php
public function handle($request, Closure $next)
{
    $response = $next($request);
    
    $response->headers->set('X-Content-Type-Options', 'nosniff');
    $response->headers->set('X-Frame-Options', 'SAMEORIGIN');
    $response->headers->set('X-XSS-Protection', '1; mode=block');
    $response->headers->set('Referrer-Policy', 'strict-origin-when-cross-origin');
    $response->headers->set('Permissions-Policy', 'geolocation=(), microphone=()');
    
    return $response;
}
```

Or use **Cloudflare's** Transform Rules for global application.

### Step 16 — Set up Cloudflare Analytics

Cloudflare → Analytics:

- See traffic
- See threats blocked
- See cache hit rate
- See bandwidth saved

> **Cache hit rate > 80% is the senior's goal.** Means 80% of requests never hit your origin.

---

## Quiz

**Q1.** You put your CSS on a CDN. When you change the CSS, users see the old version for a week. Why? How do you fix it?

**Q2.** Your Cloudflare cache hit rate is 30%. The senior says "should be 80%." What might be wrong?

**Q3.** A user says "the page is slow on mobile." You check Lighthouse — page is 4MB. Walk through your fixes.

**Q4.** You have a marketing page that doesn't change often. Should you cache it on CDN? What's the trade-off?

**Q5.** (Stretch) Your SaaS is global. Customers in Tokyo see 1.5s page loads. Walk through your CDN + asset optimization plan.

---

## What You Now Have

✅ Cloudflare (free) configured
✅ Static asset long-term caching
✅ Vite + content-hashed assets
✅ Image optimization (WebP, lazy loading)
✅ Brotli compression
✅ Security headers
✅ Performance measurement

## What's Next

Lesson 51 covers **backups, disaster recovery, and RPO/RTO** — what happens when your server dies, your database corrupts, or you get ransomware. The senior's plan for the worst day.

📖 **[Lesson 51 — Backups, Disaster Recovery, RPO/RTO](51-backups-disaster-recovery-rpo-rto.md)**
