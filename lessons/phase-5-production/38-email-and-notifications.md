# Lesson 38 — Email and Notifications

> **Phase 5: Production** | You'll be here: ~30 min

## The Senior's Question

Your SaaS needs to send emails: payment receipts, password resets, attendance digests, marketing. The junior uses `mail()` and the page hangs for 8 seconds. The senior uses **queued notifications with Markdown templates and a mail service like Postmark or SES.** Let me show you how.

---

## Theory: Why Email Is Hard

Email is deceptively simple:

```php
mail($to, $subject, $body);  // ← never do this
```

Real email needs:

- **HTML + plaintext** versions
- **Spam-friendly headers** (SPF, DKIM, DMARC)
- **Deliverability** (don't get blacklisted)
- **Templates** that designers can edit
- **Tracking** (opens, clicks, bounces)
- **Bounces handling** (remove dead addresses)
- **Unsubscribe links** (CAN-SPAM, GDPR)
- **Queue integration** (don't block the request)
- **Localization** (different languages)

> **Senior rule:** never send email from your app server. Use a transactional email service (Postmark, SES, Mailgun, SendGrid). They handle deliverability, bounces, and complaints.

---

## Theory: Choosing a Mail Provider

| Provider | Pros | Cons | Price |
|---|---|---|---|
| **Postmark** | Best deliverability, great DX | More expensive | $10/mo for 10k |
| **Amazon SES** | Cheap, integrates with AWS | Setup is more work | $0.10 per 1k |
| **Mailgun** | Good APIs, EU routes | Deliverability varies | $35/mo for 50k |
| **SendGrid** | Big player, marketing features | Older API, less reliable | $20/mo for 50k |

> **Senior default:** Postmark for transactional, SES for high volume. **Don't use Gmail SMTP** — limits and blocks.

In `.env`:

```env
MAIL_MAILER=postmark
MAIL_FROM_ADDRESS="hello@coaching-saas.com"
MAIL_FROM_NAME="${APP_NAME}"
POSTMARK_TOKEN=xxx
```

---

## Theory: Laravel's Mail vs Notifications

Laravel has two ways to send:

### Mail — for emails specifically

```php
Mail::to($user)->send(new WelcomeEmail($user));
```

### Notifications — for any channel (mail, SMS, Slack, database)

```php
$user->notify(new PaymentReceived($payment));
```

A notification can be sent via:
- `mail` — email
- `database` — stored in DB, shown in bell icon
- `broadcast` — WebSocket
- `sms` — via Vonage / Twilio
- `slack` — via webhook

> **Senior rule:** use Notifications. They abstract the channel. Same code, swap from mail to SMS by changing the class config.

---

## Theory: Creating a Notification

```bash
php artisan make:notification PaymentReceivedNotification
```

`app/Notifications/PaymentReceivedNotification.php`:

```php
<?php

namespace App\Notifications;

use App\Models\Payment;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Messages\MailMessage;
use Illuminate\Notifications\Notification;

class PaymentReceivedNotification extends Notification implements ShouldQueue
{
    use Queueable;
    
    public function __construct(public Payment $payment) {}
    
    public function via($notifiable): array
    {
        return ['mail', 'database'];
    }
    
    public function toMail($notifiable): MailMessage
    {
        return (new MailMessage)
            ->subject("Payment Received: {$this->payment->amount} {$this->payment->currency}")
            ->greeting("Hi {$notifiable->name}!")
            ->line("We received your payment of {$this->payment->amount}.")
            ->action('View Invoice', route('fees.show', $this->payment->invoice_id))
            ->line('Thank you for your business!');
    }
    
    public function toArray($notifiable): array
    {
        return [
            'payment_id' => $this->payment->id,
            'amount' => $this->payment->amount,
            'message' => "Payment of {$this->payment->amount} received",
        ];
    }
}
```

> **`implements ShouldQueue`** sends the email asynchronously via the queue. The request returns immediately. Email is sent in the background.

Send it:

```php
$user->notify(new PaymentReceivedNotification($payment));
```

---

## Theory: Markdown Mail Templates

Laravel has built-in Markdown templates with pre-styled components:

```php
public function toMail($notifiable): MailMessage
{
    return (new MailMessage)
        ->subject('Welcome!')
        ->markdown('mail.welcome', ['user' => $notifiable]);
}
```

`resources/views/mail/welcome.blade.php`:

```blade
@component('mail::message')
# Welcome, {{ $user->name }}!

Thanks for joining {{ config('app.name') }}. We're excited to have you.

@component('mail::button', ['url' => route('dashboard')])
Go to Dashboard
@endcomponent

Thanks,<br>
{{ config('app.name') }}
@endcomponent
```

> **Markdown templates** are easy for non-developers to edit. Designers can tweak CSS in `resources/views/vendor/mail/html/themes/default.css`.

---

## Theory: Custom Mail Templates

For branded emails, create a custom theme:

```bash
php artisan vendor:publish --tag=laravel-mail
```

Edit `resources/views/vendor/mail/html/themes/coaching.blade.php`:

```blade
<style>
    .header { background: #4F46E5; color: white; }
    .button { background: #4F46E5; }
</style>
```

Use it:

```php
->theme('coaching')
```

---

## Theory: Queued Notifications

By default, notifications are sent immediately. For real apps, queue them:

```php
use Illuminate\Contracts\Queue\ShouldQueue;

class PaymentReceivedNotification extends Notification implements ShouldQueue
{
    use Queueable;
    
    public $tries = 3;
    public $backoff = [60, 180, 600];  // 1 min, 3 min, 10 min
    public $timeout = 30;
}
```

> **If the email service is down, the job retries.** After 3 fails, it goes to the failed jobs table. You can inspect and replay.

Run the worker:

```bash
php artisan queue:work
```

In production, use Supervisor (covered in a later lesson).

---

## Theory: Database Notifications (The Bell Icon)

Store notifications in the database for the in-app bell:

Migration:

```bash
php artisan notifications:table
php artisan migrate
```

```php
public function via($notifiable): array
{
    return ['mail', 'database'];
}
```

Show in bell:

```php
$user->unreadNotifications;  // Collection
$user->readNotifications;
$user->unreadNotifications->markAsRead();
```

Display:

```blade
<li class="nav-item">
    <a href="{{ route('notifications.index') }}">
        <i class="bell-icon"></i>
        @if(auth()->user()->unreadNotifications->count() > 0)
            <span class="badge">
                {{ auth()->user()->unreadNotifications->count() }}
            </span>
        @endif
    </a>
</li>
```

> **Database notifications** give the user a history. They can also act as a "did the email work?" backup.

---

## Theory: Notification Routing (Multi-Tenancy)

In a multi-tenant app, notifications must respect the tenant:

```php
class PaymentReceivedNotification extends Notification
{
    public function routeNotificationForMail($notifiable)
    {
        // Use the tenant's from address
        return $notifiable->tenant->email ?? $notifiable->email;
    }
}
```

Or use the `NotificationSender` trait:

```php
trait TenantScopedNotifications
{
    public function routeNotificationForMail($notification)
    {
        // For tenant admins, send to the tenant's primary email
        if ($this instanceof Tenant) {
            return $this->billing_email;
        }
        
        return $this->email;
    }
}
```

---

## Theory: Bounce and Complaint Handling

When an email bounces, your provider sends a webhook:

```php
// routes/web.php
Route::post('webhooks/postmark', [PostmarkWebhookController::class, 'handle']);
```

`PostmarkWebhookController`:

```php
public function handle(Request $request)
{
    $event = $request->input('RecordType');
    
    if ($event === 'Bounce') {
        $email = $request->input('Email');
        User::where('email', $email)->update(['email_bounced' => true]);
    }
    
    if ($event === 'SpamComplaint') {
        $email = $request->input('Email');
        User::where('email', $email)->update(['email_unsubscribed' => true]);
    }
    
    return response('ok', 200);
}
```

> **Senior rule:** honor bounces and complaints. If you keep emailing dead addresses, your sender reputation tanks.

---

## Theory: Unsubscribe Links (CAN-SPAM, GDPR)

Every marketing email must have an unsubscribe link:

```php
public function toMail($notifiable): MailMessage
{
    return (new MailMessage)
        ->subject('...')
        ->view('emails.weekly-digest')
        ->with(['unsubscribeUrl' => route('unsubscribe', $notifiable->id)]);
}
```

In template:

```blade
@include('emails.partials.unsubscribe', ['url' => $unsubscribeUrl])
```

> **Transactional emails** (receipts, password resets) don't need unsubscribe. But **marketing** does. **Legal requirement.**

---

## Theory: Local Development with Mailpit

Don't send real emails in dev. Use **Mailpit** (or `log` driver):

```bash
# Install Mailpit
# macOS
brew install mailpit
mailpit

# Windows
# Download from https://github.com/axllent/mailpit/releases
```

In `.env`:

```env
MAIL_MAILER=smtp
MAIL_HOST=localhost
MAIL_PORT=1025
MAIL_USERNAME=null
MAIL_PASSWORD=null
```

View emails at `http://localhost:8025`. **No emails leave your machine.**

> **Senior rule:** `log` driver in tests, `smtp` to Mailpit in dev, real provider in staging/prod.

---

## Theory: Testing Notifications

```php
use Illuminate\Support\Facades\Notification;

test('payment triggers notification', function () {
    Notification::fake();
    
    $student = Student::factory()->for($this->tenant)->create();
    $payment = Payment::factory()->for($student)->create();
    
    $student->user->notify(new PaymentReceivedNotification($payment));
    
    Notification::assertSentTo(
        $student->user,
        PaymentReceivedNotification::class
    );
});
```

For the actual mail content:

```php
use Illuminate\Support\Facades\Mail;

test('payment email has correct content', function () {
    Mail::fake();
    
    $user = User::factory()->create();
    $payment = Payment::factory()->create();
    
    $user->notify(new PaymentReceivedNotification($payment));
    
    Mail::assertSent(PaymentReceivedNotification::class, function ($mail) use ($payment) {
        return $mail->hasTo($user->email)
            && $mail->subject === "Payment Received: {$payment->amount}";
    });
});
```

---

## Why It Matters

Email is the **voice of your SaaS.** Get it right:
- Receipts land in inbox (not spam)
- Password resets arrive in 5 seconds
- Attendance digests reach parents on time
- Failed notifications get retried
- Bounces don't kill your sender score

> **A senior's email system is queued, branded, tracked, and respects the user's inbox.** A junior's `mail()` calls go to spam and break when the SMTP server hiccups.

---

## Lab: Add Notifications to Your SaaS

### Step 1 — Configure mail in dev

Install Mailpit:

```bash
brew install mailpit  # or download for Windows
mailpit
```

`.env`:

```env
MAIL_MAILER=smtp
MAIL_HOST=localhost
MAIL_PORT=1025
```

### Step 2 — Create the notifications table

```bash
php artisan notifications:table
php artisan migrate
```

### Step 3 — Build a notification

```bash
php artisan make:notification PaymentReceivedNotification
```

`app/Notifications/PaymentReceivedNotification.php`:

```php
<?php

namespace App\Notifications;

use App\Models\Payment;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Messages\MailMessage;
use Illuminate\Notifications\Notification;

class PaymentReceivedNotification extends Notification implements ShouldQueue
{
    use Queueable;
    
    public function __construct(public Payment $payment) {}
    
    public function via($notifiable): array
    {
        return ['mail', 'database'];
    }
    
    public function toMail($notifiable): MailMessage
    {
        $invoice = $this->payment->invoice;
        
        return (new MailMessage)
            ->subject("Payment Receipt: {$this->payment->amount} {$this->payment->currency}")
            ->greeting("Hi {$notifiable->name}!")
            ->line("Thank you for your payment of {$this->payment->amount} {$this->payment->currency}.")
            ->line("Invoice: #{$invoice->id}")
            ->line("Status: {$invoice->status}")
            ->action('View Invoice', route('fees.invoice', $invoice))
            ->line('Thank you for choosing ' . config('app.name') . '!');
    }
    
    public function toArray($notifiable): array
    {
        return [
            'payment_id' => $this->payment->id,
            'invoice_id' => $this->payment->invoice_id,
            'amount' => $this->payment->amount,
            'message' => "Payment of {$this->payment->amount} received",
        ];
    }
}
```

### Step 4 — Trigger the notification from the action

`app/Actions/RecordPayment.php`:

```php
namespace App\Actions;

use App\Models\Payment;
use App\Models\FeeInvoice;
use App\Notifications\PaymentReceivedNotification;
use Illuminate\Support\Facades\DB;

class RecordPayment
{
    public function execute(FeeInvoice $invoice, float $amount, string $method): Payment
    {
        return DB::transaction(function () use ($invoice, $amount, $method) {
            $payment = Payment::create([
                'invoice_id' => $invoice->id,
                'student_id' => $invoice->student_id,
                'tenant_id' => $invoice->tenant_id,
                'amount' => $amount,
                'method' => $method,
                'paid_at' => now(),
            ]);
            
            $invoice->paid += $amount;
            $invoice->due -= $amount;
            $invoice->status = $invoice->due <= 0 ? 'paid' : 'partial';
            $invoice->save();
            
            // Notify the parent
            if ($invoice->student->guardian) {
                $invoice->student->guardian->notify(
                    new PaymentReceivedNotification($payment)
                );
            }
            
            return $payment;
        });
    }
}
```

### Step 5 — Create the mail template

`resources/views/emails/payment-receipt.blade.php`:

```blade
@component('mail::message')
# Payment Receipt

Hi {{ $payment->invoice->student->guardian?->name ?? 'Parent' }},

We have received your payment.

**Amount:** {{ number_format($payment->amount, 2) }}
**Method:** {{ ucfirst($payment->method) }}
**Date:** {{ $payment->paid_at->format('F j, Y') }}
**Invoice:** #{{ $payment->invoice->id }}
**Status:** {{ ucfirst($payment->invoice->status) }}

@component('mail::button', ['url' => route('fees.invoice', $payment->invoice)])
View Invoice
@endcomponent

Thank you,<br>
{{ config('app.name') }}
@endcomponent
```

### Step 6 — Show the bell icon

`resources/views/layouts/app.blade.php` (in the nav):

```blade
<li class="nav-item dropdown">
    <a href="#" data-toggle="dropdown">
        <i class="fas fa-bell"></i>
        @if(auth()->user()->unreadNotifications->count() > 0)
            <span class="badge badge-danger">
                {{ auth()->user()->unreadNotifications->count() }}
            </span>
        @endif
    </a>
    <div class="dropdown-menu">
        @forelse(auth()->user()->unreadNotifications->take(5) as $notification)
            <a href="{{ route('notifications.show', $notification->id) }}" class="dropdown-item">
                {{ $notification->data['message'] ?? 'Notification' }}
                <small class="text-muted">{{ $notification->created_at->diffForHumans() }}</small>
            </a>
        @empty
            <span class="dropdown-item text-muted">No new notifications</span>
        @endforelse
    </div>
</li>
```

### Step 7 — Mark as read

`app/Http/Controllers/NotificationController.php`:

```php
public function show(string $id)
{
    $notification = auth()->user()->notifications()->findOrFail($id);
    $notification->markAsRead();
    
    return redirect($notification->data['url'] ?? '/');
}

public function markAllRead()
{
    auth()->user()->unreadNotifications->markAsRead();
    return back();
}
```

### Step 8 — Queue the work

Make sure your queue runs:

```bash
php artisan queue:work
```

> **The notification sends in the background.** The user gets the success page immediately.

### Step 9 — Configure for production

In production `.env`:

```env
MAIL_MAILER=postmark
POSTMARK_TOKEN=your_real_token
```

Test the connection:

```bash
php artisan tinker
>>> Mail::raw('test', fn($m) => $m->to('you@example.com')->subject('test'));
```

### Step 10 — Add unsubscribe for marketing

`resources/views/emails/partials/footer.blade.php`:

```blade
<small>
    You're receiving this because you're a customer of {{ config('app.name') }}.
    <a href="{{ route('unsubscribe', $user->unsubscribe_token) }}">Unsubscribe</a>
</small>
```

In the User model:

```php
public function getUnsubscribeTokenAttribute(): string
{
    return hash('sha256', $this->id . $this->email . config('app.key'));
}
```

---

## Quiz

**Q1.** Why use a notification class instead of `Mail::send()` directly? List 2 specific advantages.

**Q2.** A junior says "I sent the email synchronously, the request took 3 seconds." What's the senior's fix? What 2-line change makes it 200ms?

**Q3.** What's the difference between `mail` and `database` channels in a notification? When would you use each?

**Q4.** Your app sends 10,000 marketing emails a week. After 2 months, Gmail sends them to spam. What likely went wrong? How do you prevent it?

**Q5.** (Stretch) You have a `PaymentReceivedNotification` for parents. The same event also needs to notify the school admin. How do you structure this so both get notified without duplicating logic?

---

## What You Now Have

✅ Notification fluency (mail + database)
✅ Queued notifications with retries
✅ Markdown templates
✅ Multi-channel delivery (mail, DB, future SMS/Slack)
✅ Bounce/complaint handling
✅ Test patterns

## What's Next

Lesson 39 covers **file storage and S3** — uploading student photos, invoices, signed URLs, the patterns for handling files in production.

📖 **[Lesson 39 — File Storage and S3](39-file-storage-and-s3.md)**
