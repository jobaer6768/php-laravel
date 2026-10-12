# Lesson 39 — File Storage and S3

> **Phase 5: Production** | You'll be here: ~30 min

## The Senior's Question

Students have profile photos. Invoices are PDFs. Receipts need to be downloadable. The junior saves files to `public/uploads`. The senior uses **Laravel Storage with S3** — durable, scalable, served from a CDN. Let me show you how.

---

## Theory: Why Local Storage Is a Trap

Storing files in `public/uploads` or `storage/app`:

```
Local Storage Problems:
- Files lost when container restarts
- Can't scale horizontally (files on one server, request on another)
- No CDN delivery (slow for users in another country)
- Backups are hard
- Permissions are a nightmare
```

> **Senior rule:** in production, files live in object storage (S3, DigitalOcean Spaces, Cloudflare R2). Your app server just stores URLs.

---

## Theory: Laravel's Filesystem Abstraction

Laravel's `Storage` facade abstracts the disk:

```php
use Illuminate\Support\Facades\Storage;

Storage::disk('local')->put('file.txt', 'contents');
Storage::disk('s3')->put('file.txt', 'contents');
```

> **The same code works with any backend.** Swap `local` → `s3` in config, no code changes.

Disks are defined in `config/filesystems.php`:

```php
'disks' => [
    'local' => [
        'driver' => 'local',
        'root' => storage_path('app'),
    ],
    'public' => [
        'driver' => 'local',
        'root' => storage_path('app/public'),
        'url' => env('APP_URL') . '/storage',
        'visibility' => 'public',
    ],
    's3' => [
        'driver' => 's3',
        'key' => env('AWS_ACCESS_KEY_ID'),
        'secret' => env('AWS_SECRET_ACCESS_KEY'),
        'region' => env('AWS_DEFAULT_REGION'),
        'bucket' => env('AWS_BUCKET'),
        'url' => env('AWS_URL'),
    ],
],
```

The default disk is set with `FILESYSTEM_DISK` in `.env`:

```env
FILESYSTEM_DISK=s3
```

---

## Theory: Basic File Operations

### Write

```php
Storage::put('avatars/aisha.jpg', $fileContents);
Storage::putFile('avatars', $request->file('avatar'));  // generates unique name
Storage::putFileAs('avatars', $request->file('avatar'), 'aisha.jpg');  // specific name
```

### Read

```php
$contents = Storage::get('file.txt');
$exists = Storage::exists('file.txt');
$size = Storage::size('file.txt');
$time = Storage::lastModified('file.txt');
```

### Delete

```php
Storage::delete('file.txt');
Storage::delete(['file1.txt', 'file2.txt']);
Storage::deleteDirectory('avatars/old');
```

### Copy / Move

```php
Storage::copy('old/file.txt', 'new/file.txt');
Storage::move('old/file.txt', 'new/file.txt');
```

### URLs

```php
$url = Storage::url('avatars/aisha.jpg');  // public URL
$url = Storage::temporaryUrl(
    'invoices/private.pdf',
    now()->addMinutes(5)
);  // S3 signed URL
```

---

## Theory: Uploads from Forms

### Basic upload

```php
public function store(Request $request)
{
    $request->validate([
        'avatar' => 'required|image|max:2048',  // 2MB max
    ]);
    
    $path = $request->file('avatar')->store('avatars', 's3');
    
    $user->update(['avatar_path' => $path]);
    
    return back();
}
```

### Customizing the filename

```php
$path = $request->file('avatar')->storeAs(
    'avatars',
    Str::uuid() . '.' . $request->file('avatar')->extension(),
    's3'
);
```

> **`storeAs()`** lets you set the filename. Useful for predictable paths (`invoice-123.pdf`).

### Multiple uploads

```php
foreach ($request->file('documents') as $file) {
    $path = $file->store('documents', 's3');
    Document::create(['path' => $path, 'name' => $file->getClientOriginalName()]);
}
```

---

## Theory: File Visibility

S3 has the concept of public vs private:

```php
// Public — anyone with the link can access
Storage::putFileAs('public/avatars', $file, 'aisha.jpg');

// Private — needs a signed URL
Storage::putFileAs('private/invoices', $file, 'invoice-123.pdf');
```

In the model:

```php
$student->avatar_url = Storage::disk('s3')->url($student->avatar_path);
```

For private files:

```php
return Storage::disk('s3')->temporaryUrl(
    $invoice->path,
    now()->addMinutes(5)
);
```

> **Senior rule:** default to private. Public files are public forever (and indexed by search engines). Use signed URLs for everything sensitive.

---

## Theory: Signed URLs (The Right Way)

A signed URL grants temporary access. Perfect for:
- Invoice PDFs (parents download their own)
- Reports (admin generates link, sends to boss)
- Receipts

```php
$url = Storage::disk('s3')->temporaryUrl(
    'invoices/invoice-123.pdf',
    now()->addMinutes(30)
);

return $url;
```

> **The URL expires.** Even if leaked, it works for 30 minutes only.

You can also generate signed URLs from S3 console directly.

---

## Theory: Serving Files (Download vs Display)

### Direct download (force browser to save)

```php
return Storage::disk('s3')->download('invoices/invoice-123.pdf');
```

Or with a custom filename:

```php
return Storage::disk('s3')->download(
    'invoices/invoice-123.pdf',
    "Invoice-{$invoice->id}.pdf",
    ['Content-Type' => 'application/pdf']
);
```

### Display inline (browser shows the PDF)

```php
return response()->file(
    Storage::disk('s3')->path('invoices/invoice-123.pdf'),
    ['Content-Disposition' => 'inline; filename="invoice.pdf"']
);
```

Or just embed the signed URL in an `<iframe>` or `<embed>`.

---

## Theory: S3 Setup

### Install the S3 driver

```bash
composer require league/flysystem-aws-s3-v3
```

### Get credentials

In AWS Console:
1. Create an IAM user with `AmazonS3FullAccess`
2. Get Access Key ID and Secret
3. Create an S3 bucket

In `.env`:

```env
FILESYSTEM_DISK=s3
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=coaching-saas-prod
AWS_URL=https://coaching-saas-prod.s3.amazonaws.com
```

> **Senior rule:** use IAM roles, not access keys, when running on AWS. The credentials are automatic.

---

## Theory: Alternative Providers

### DigitalOcean Spaces (S3-compatible, cheaper)

```env
FILESYSTEM_DISK=s3
AWS_ENDPOINT=https://nyc3.digitaloceanspaces.com
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_BUCKET=coaching
AWS_DEFAULT_REGION=us-east-1
```

### Cloudflare R2 (free egress)

```env
FILESYSTEM_DISK=s3
AWS_ENDPOINT=https://<accountid>.r2.cloudflarestorage.com
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_BUCKET=coaching
AWS_DEFAULT_REGION=auto
```

### MinIO (self-hosted S3)

```env
AWS_ENDPOINT=http://minio:9000
AWS_ACCESS_KEY_ID=minio
AWS_SECRET_ACCESS_KEY=minio_password
AWS_BUCKET=coaching
```

> **R2 is the senior's choice for new projects.** Zero egress fees. Same S3 API.

---

## Theory: Multi-Tenant File Isolation

Each tenant's files must be isolated:

```php
public function uploadAvatar(Request $request)
{
    $tenant = app('current_tenant');
    
    $path = $request->file('avatar')->storeAs(
        "tenants/{$tenant->id}/avatars",
        $request->file('avatar')->hashName(),
        's3'
    );
    
    $request->user()->update(['avatar_path' => $path]);
}
```

> **Path-prefixed tenant isolation.** `tenants/1/avatars/...`, `tenants/2/avatars/...`. Easy to give a tenant a "delete all my data" feature.

For very paranoid isolation, use **separate S3 buckets per tenant** (expensive, complex).

---

## Theory: Direct Browser Upload (Bypass Your Server)

For large files (videos, 50MB PDFs), don't upload through your Laravel server:

```
1. Browser → asks your server for a presigned URL
2. Your server → returns a presigned S3 PUT URL
3. Browser → uploads directly to S3
4. Browser → tells your server it's done
```

```php
public function getUploadUrl(Request $request)
{
    $tenant = app('current_tenant');
    
    $filename = $request->input('filename');
    $path = "tenants/{$tenant->id}/uploads/{$filename}";
    
    return response()->json([
        'url' => Storage::disk('s3')->temporaryUploadUrl($path, now()->addMinutes(5)),
        'path' => $path,
    ]);
}
```

Frontend uploads directly to the `url`. Then notifies your API.

> **Senior rule:** for files over 10MB, use presigned URLs. Saves your server bandwidth.

---

## Theory: CDN in Front of S3

S3 is fast but CloudFront (or Cloudflare) is faster:

```php
'cloudfront' => [
    'driver' => 's3',
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION'),
    'bucket' => env('AWS_BUCKET'),
    'url' => env('AWS_CLOUDFRONT_URL'),  // https://d123.cloudfront.net
    'cloudfront_signed_url' => true,     // for private files
],
```

> **CloudFront** caches files at edge locations. Users in Tokyo get the file from a Tokyo server, not Virginia.

---

## Theory: Image transformations (thumbnails)

Don't resize images in PHP. Use S3 + Lambda, or Cloudflare Images, or a service like imgix.

Simple option: **intervention/image** for occasional use:

```bash
composer require intervention/image
```

```php
use Intervention\Image\Facades\Image;

$img = Image::make($request->file('avatar'))
    ->fit(200, 200)
    ->encode('jpg', 80);

Storage::disk('s3')->put('avatars/thumb.jpg', $img);
```

> **Senior rule:** generate thumbnails on upload, not on request. Store both.

---

## Theory: Backup Files

S3 itself is durable (99.999999999% — eleven nines). But:

- Enable **versioning** (rollback accidental deletes)
- Enable **cross-region replication** (DR)
- Lifecycle policy: move old files to Glacier (cheaper)

In S3 Console → Bucket → Management → Lifecycle rule:

```
Rule: Move to Glacier after 90 days
Rule: Delete incomplete multipart uploads after 7 days
```

---

## Theory: Local Development

For dev, use the `local` disk:

```env
FILESYSTEM_DISK=local
```

And link `storage/app/public` to `public/storage`:

```bash
php artisan storage:link
```

> **`storage:link`** creates a symlink. Files in `storage/app/public` are served at `/storage/`.

---

## Why It Matters

Files are user data. Get it wrong:
- Photos get lost (forever)
- Invoices inaccessible to parents
- Storage costs balloon
- GDPR/data residency issues (files in wrong region)

Get it right:
- Files in the cloud with versioning
- Signed URLs for private files
- CDN for speed
- Tenant isolation built in

> **A senior thinks about files as data, not as part of the app.** Cloud, durable, served from the edge.

---

## Lab: Add File Uploads to Your SaaS

### Step 1 — Install S3 driver

```bash
composer require league/flysystem-aws-s3-v3
```

For dev, use the `local` disk. For prod, use S3 or R2.

### Step 2 — Add the avatar column

```bash
php artisan make:migration add_avatar_to_users_table --table=users
```

```php
Schema::table('users', function (Blueprint $table) {
    $table->string('avatar_path')->nullable();
});
```

```bash
php artisan migrate
```

### Step 3 — Create the upload form

`resources/views/profile/edit.blade.php`:

```blade
<form method="POST" action="{{ route('profile.avatar') }}" enctype="multipart/form-data">
    @csrf
    
    <div class="form-group">
        <label>Profile Photo</label>
        <input type="file" name="avatar" accept="image/*" required>
        @error('avatar') <span class="text-danger">{{ $message }}</span> @enderror
    </div>
    
    <button type="submit" class="btn btn-primary">Upload</button>
</form>

@if(auth()->user()->avatar_path)
    <img src="{{ auth()->user()->avatar_url }}" class="avatar">
@endif
```

### Step 4 — Add the controller

`app/Http/Controllers/ProfileController.php`:

```php
public function updateAvatar(Request $request)
{
    $request->validate([
        'avatar' => 'required|image|max:2048|dimensions:min_width=100,min_height=100',
    ]);
    
    $tenant = app('current_tenant');
    $user = $request->user();
    
    // Delete old avatar
    if ($user->avatar_path) {
        Storage::disk(config('filesystems.default'))->delete($user->avatar_path);
    }
    
    // Store new
    $path = $request->file('avatar')->storeAs(
        "tenants/{$tenant->id}/avatars",
        Str::uuid() . '.' . $request->file('avatar')->extension(),
        config('filesystems.default')
    );
    
    $user->update(['avatar_path' => $path]);
    
    return back()->with('success', 'Avatar updated');
}
```

### Step 5 — Add an accessor for the URL

`app/Models/User.php`:

```php
protected $appends = ['avatar_url'];

public function getAvatarUrlAttribute(): ?string
{
    if (!$this->avatar_path) {
        return null;
    }
    
    return Storage::disk(config('filesystems.default'))->url($this->avatar_path);
}
```

### Step 6 — Generate signed URLs for invoices

```php
public function downloadInvoice(FeeInvoice $invoice)
{
    $this->authorize('view', $invoice);
    
    if (!$invoice->pdf_path) {
        abort(404);
    }
    
    return Storage::disk('s3')->temporaryUrl(
        $invoice->pdf_path,
        now()->addMinutes(5)
    );
}
```

### Step 7 — Add invoice PDF upload

In the fee controller:

```php
public function uploadInvoicePdf(Request $request, FeeInvoice $invoice)
{
    $request->validate([
        'pdf' => 'required|file|max:10240',  // 10MB
    ]);
    
    $tenant = app('current_tenant');
    $path = $request->file('pdf')->storeAs(
        "tenants/{$tenant->id}/invoices",
        "invoice-{$invoice->id}.pdf",
        's3'
    );
    
    $invoice->update(['pdf_path' => $path]);
    
    return back()->with('success', 'Invoice uploaded');
}
```

### Step 8 — Use storage:link for dev

In dev (when using local disk):

```bash
php artisan storage:link
```

Now `/storage/avatars/xxx.jpg` serves the files.

### Step 9 — Configure for production

In production `.env`:

```env
FILESYSTEM_DISK=s3
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_BUCKET=coaching-saas-prod
AWS_DEFAULT_REGION=us-east-1
```

### Step 10 — Test uploads

```bash
php artisan tinker
>>> Storage::disk('s3')->put('test.txt', 'hello');
>>> Storage::disk('s3')->exists('test.txt');
=> true
>>> Storage::disk('s3')->url('test.txt');
=> "https://coaching-saas-prod.s3.amazonaws.com/test.txt"
```

### Step 11 — Generate thumbnails on upload

```bash
composer require intervention/image
```

In `ProfileController`:

```php
public function updateAvatar(Request $request)
{
    $request->validate(['avatar' => 'required|image|max:2048']);
    
    $tenant = app('current_tenant');
    $user = $request->user();
    $file = $request->file('avatar');
    
    $uuid = Str::uuid();
    $disk = config('filesystems.default');
    
    // Original
    Storage::disk($disk)->putFileAs(
        "tenants/{$tenant->id}/avatars",
        $file,
        "{$uuid}.{$file->extension()}"
    );
    
    // Thumbnail
    $thumb = Image::make($file)->fit(200, 200)->encode('jpg', 80);
    Storage::disk($disk)->put(
        "tenants/{$tenant->id}/avatars/thumbs/{$uuid}.jpg",
        $thumb
    );
    
    $user->update(['avatar_path' => "tenants/{$tenant->id}/avatars/{$uuid}.{$file->extension()}"]);
}
```

### Step 12 — Clean up on user delete

`app/Observers/UserObserver.php`:

```php
public function deleted(User $user): void
{
    if ($user->avatar_path) {
        Storage::disk(config('filesystems.default'))->delete($user->avatar_path);
        // Also delete thumb
    }
}
```

---

## Quiz

**Q1.** Why not store uploaded files in `public/uploads`? List 2 specific production problems.

**Q2.** A signed URL expires in 30 minutes. Why? What if they need access for a week?

**Q3.** When would you use a presigned upload URL instead of uploading through Laravel?

**Q4.** You have 1000 students, each with 5 photos. Storage is getting expensive. What lifecycle policy would you apply?

**Q5.** (Stretch) Your app supports 5,000 tenants. Each tenant can upload 10GB. Some tenants leave. How do you handle "delete all data for tenant X" cleanly?

---

## What You Now Have

✅ Storage facade (local + S3)
✅ File uploads with validation
✅ Signed URLs for private files
✅ Tenant-isolated paths
✅ Thumbnail generation
✅ Presigned direct uploads
✅ CloudFront / CDN setup

## What's Next

Lesson 40 covers **logging, monitoring, and debugging** — when something breaks in production, how do you know? How do you find the cause? Structured logs, error tracking, performance monitoring.

📖 **[Lesson 40 — Logging, Monitoring, and Debugging](40-logging-monitoring-and-debugging.md)**