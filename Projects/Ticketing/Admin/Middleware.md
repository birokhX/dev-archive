برای ادمین میخوام middleware استفاده کنم و یاد بگیرم چون باز خوب درست بلد نیستم.

# آموزش Middleware در Laravel 13

## نقشه راه آموزشی

این آموزش را به ۷ مرحله تقسیم می‌کنیم:

|مرحله|موضوع|خروجی|
|---|---|---|
|۱|مفهوم Middleware و چرخه درخواست|درک معماری|
|۲|ساختار و محل تعریف Middleware|شناخت `bootstrap/app.php`|
|۳|Middleware احراز هویت Admin|محافظت از پنل ادمین|
|۴|Middleware وضعیت حساب|جلوگیری از ورود حساب غیرفعال|
|۵|Middleware احراز هویت دومرحله‌ای|کنترل وضعیت MFA|
|۶|Middleware تغییر اجباری رمز|کنترل دسترسی حساب‌های دعوت‌شده|
|۷|گروه‌بندی و معماری نهایی Routeها|ساختار Production|

فعلاً فقط مرحله اول و دوم را یاد می‌گیریم. بعد از بررسی ساختار فعلی پروژه، وارد پیاده‌سازی می‌شویم.

# مرحله اول: Middleware چیست و چرا به آن نیاز داریم؟

فرض کن یک ادمین می‌خواهد وارد این آدرس شود:

```
/admin/dashboard
```

قبل از اینکه Laravel کامپوننت داشبورد را اجرا کند، باید چند موضوع بررسی شود:

1. آیا کاربر وارد سیستم شده است؟
    
2. آیا حساب او فعال است؟
    
3. آیا احراز هویت دومرحله‌ای را تکمیل کرده است؟
    
4. آیا مجبور به تغییر رمز عبور است؟
    
5. آیا اجازه دسترسی به این بخش را دارد؟
اگر تمام این بررسی‌ها را داخل کامپوننت داشبورد بنویسی، چه اتفاقی می‌افتد؟

```php
public function mount()
{
    // Check authentication
    // Check account status
    // Check MFA
    // Check password expiration
    // Check permissions
}
```

این طراحی مناسب نیست؛ چون منطق امنیتی به صفحه داشبورد وابسته شده است. اگر فردا صفحه دیگری مثل گزارش‌ها یا مدیریت ادمین‌ها بسازی، باید دوباره همین بررسی‌ها را تکرار کنی.

## Middleware در چرخه درخواست

![[Screenshot 2026-11-04 234620.png]]

Middleware می‌تواند قبل از اجرای منطق اصلی درخواست، آن را بررسی کند و یکی از این کارها را انجام دهد:

- اجازه ادامه درخواست بدهد.
    
- کاربر را Redirect کند.
    
- پاسخ خطای HTTP بدهد.
    
- درخواست را متوقف کند.
    
- بعد از اجرای درخواست نیز پاسخ را بررسی یا تغییر دهد.

ساختار پایه یک Middleware:

```php
public function handle(Request $request, Closure $next): Response
{
    // Before request

    $response = $next($request);

    // After request

    return $response;
}
```

نکته مهم: `$next($request)` یعنی درخواست را به مرحله بعدی چرخه بسپار.
اگر قبل از آن `return` کنی، درخواست دیگر به مراحل بعدی نمی‌رسد.

# مرحله دوم: Middlewareها در Laravel 13 کجا تعریف می‌شوند؟

در Laravel 13، تنظیمات اصلی Middleware در فایل زیر قرار دارد:

```bash
bootstrap/app.php
```

در نسخه‌های جدید Laravel، برخلاف ساختار قدیمی، لازم نیست حتماً فایل `app/Http/Kernel.php` داشته باشی.

سه بخش مهم در `bootstrap/app.php` وجود دارد:

### ۱. Global Middleware

Middlewareهایی که روی تمام درخواست‌های برنامه اجرا می‌شوند.

مثلاً:

- مدیریت Trusted Proxies     
- مدیریت CORS
    
- برخی بررسی‌های عمومی HTTP
    
این قسمت برای Middleware اختصاصی پنل ادمین نیست.

### ۲. Middleware Groups

برای دسته‌بندی Middlewareها استفاده می‌شوند.

دو گروه مهم Laravel:

- `web`: برای درخواست‌های وب، Session، Cookie و CSRF
    
- `api`: برای درخواست‌های API
    

### ۳. Middleware Aliases

برای تعریف نام کوتاه برای Middlewareهای اختصاصی استفاده می‌شوند.

مثلاً:
```php
'admin.active' => EnsureAdminIsActive::class,
```

بعد می‌توانی در Route بنویسی:
```php
Route::middleware('admin.active')->group(function () {
    // ...
});
```

این نام‌گذاری باعث می‌شود Routeها خواناتر شوند.

## ساختار واقعی `bootstrap/app.php`

یک نمونه ساده از تنظیمات Middleware در Laravel 13:

```php
<?php

use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware): void {

        // Middleware aliases
        $middleware->alias([
            'admin.active' => \App\Http\Middleware\EnsureAdminIsActive::class,
        ]);

    })
    ->withExceptions(function (Exceptions $exceptions): void {
        //
    })
    ->create();
```

در این مثال، `admin.active` یک Alias است؛ یعنی نام کوتاهی برای کلاس Middleware.

این دو معادل یکدیگر هستند:
```php
Route::middleware(
    \App\Http\Middleware\EnsureAdminIsActive::class
);
```

و:

```php
Route::middleware('admin.active');
```

## چه چیزهایی را Middleware کنیم و چه چیزهایی را نه؟

این تفکیک در پروژه‌های حرفه‌ای بسیار مهم است:

| نیاز                  | راهکار مناسب                      |
| --------------------- | --------------------------------- |
| ورودکردن کاربر        | Authentication Middleware         |
| فعال‌بودن حساب        | Custom Middleware                 |
| تکمیل MFA             | Custom Middleware                 |
| اجبار به تغییر رمز    | Custom Middleware                 |
| مجوز مشاهده یک تیکت   | Policy                            |
| مجوز حذف دسته‌بندی    | Policy                            |
| اعتبارسنجی فرم        | Form Request / Livewire Form      |
| محدودیت تعداد درخواست | Rate Limiter                      |
| ثبت فعالیت ادمین      | Event / Listener یا Audit Service |

سرفصل‌هایی که در این سند تکمیل خواهیم کرد:

1. Middleware چیست؟
    
2. Request Lifecycle
    
3. Global Middleware
    
4. Middleware Groups
    
5. Middleware Aliases
    
6. ساخت Custom Middleware
    
7. Authentication Middleware
    
8. Admin Account Status Middleware
    
9. MFA Middleware
    
10. Force Password Change Middleware
    
11. Route Group Architecture
    
12. Middleware Testing
13. Production Security Considerations


# پیاده سازی Middleware های ادمین

پیشنهاد من این است که Middlewareهای ادمین را به چهار مسئولیت محدود کنیم:

1. احراز هویت ادمین: `auth:admin` — خود Laravel دارد؛ نیازی به ساخت Middleware اختصاصی نیست.
    
2. بررسی فعال‌بودن حساب: `EnsureAdminIsActive`
    
3. بررسی تکمیل احراز هویت دومرحله‌ای: `EnsureAdminHasCompletedTwoFactor`
    
4. اجبار به تغییر رمز عبور: `EnsureAdminHasChangedPassword`

نکته معماری: مجوز دسترسی به صفحات و عملیات خاص مثل حذف ادمین یا مدیریت تیکت‌ها را با Policy کنترل می‌کنیم، نه Middleware.

## ۱. ساختار پوشه‌ها

```
app/
└── Http/
    └── Middleware/
        └── Admin/
            ├── EnsureAdminIsActive.php
            ├── EnsureAdminHasCompletedTwoFactor.php
            └── EnsureAdminHasChangedPassword.php
```

برای هر Middleware یک کلاس مستقل می‌سازیم.

```bash
php artisan make:middleware Admin/EnsureAdminIsActive
php artisan make:middleware Admin/EnsureAdminHasCompletedTwoFactor
php artisan make:middleware Admin/EnsureAdminHasChangedPassword
```

مسئولیت هر کلاس:

|Middleware|مسئولیت|
|---|---|
|`auth:admin`|بررسی ورود ادمین|
|`EnsureAdminIsActive`|جلوگیری از دسترسی حساب غیرفعال|
|`EnsureAdminHasCompletedTwoFactor`|الزام به تکمیل MFA|
|`EnsureAdminHasUpdatedPassword`|الزام به تغییر رمز اولیه


## ۲. بررسی فعال‌بودن حساب ادمین

فایل:

`app/Http/Middleware/Admin/EnsureAdminIsActive.php`

```php

<?php

namespace App\Http\Middleware\Admin;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Symfony\Component\HttpFoundation\Response;

final class EnsureAdminIsActive
{
    public function handle(Request $request, Closure $next): Response
    {
        $admin = $request->user('admin');

        if (! $admin || ! $admin->is_active) {
            Auth::guard('admin')->logout();

            $request->session()->regenerate();
            $request->session()->regenerateToken();

            return redirect()
                ->route('admin.login')
                ->with('error', 'حساب کاربری شما غیرفعال است.');
        }

        return $next($request);
    }
}

```
این Middleware باید بعد از `auth:admin` اجرا شود.

## ۳. بررسی تکمیل احراز هویت دومرحله‌ای

در جدول `admins` این فیلدها را داریم:
```php
$table->text('two_factor_secret')->nullable();
$table->text('two_factor_recovery_codes')->nullable();
$table->timestamp('two_factor_confirmed_at')->nullable();
```

از این فیلد استفاده می‌کنیم:

- `two_factor_confirmed_at = null`: ادمین هنوز MFA را فعال نکرده است.
    
- مقدار غیر null: MFA قبلاً فعال و تأیید شده است.
    

اما تأیید قبلی کافی نیست؛ باید بررسی کنیم که ادمین در نشست فعلی هم کد دومرحله‌ای را با موفقیت وارد کرده باشد.

برای این کار یک مقدار در Session نگه می‌داریم:
```php
admin.two_factor_verified_admin_id
```

این مقدار فقط باید بعد از تأیید موفق کد TOTP یا Recovery Code ثبت شود.

`app/Http/Middleware/Admin/EnsureAdminHasCompletedTwoFactor.php`

```php

<?php

namespace App\Http\Middleware\Admin;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

final class EnsureAdminHasCompletedTwoFactor
{
    public function handle(Request $request, Closure $next): Response
    {
        $admin = $request->user('admin');

        if (! $admin) {
            return redirect()->route('admin.login');
        }

        // MFA has not been configured yet.
        if (! $admin->two_factor_confirmed_at) {
            return redirect()->route('admin.two-factor.setup');
        }

        // MFA must be verified in the current session.
        $verifiedAdminId = $request->session()->get(
            'admin.two_factor_verified_admin_id'
        );

        if ((string) $verifiedAdminId !== (string) $admin->getAuthIdentifier()) {
            return redirect()->route('admin.two-factor.challenge');
        }

        return $next($request);
    }
}

```

نکته بسیار مهم: این Middleware خودش کد OTP را بررسی نمی‌کند. این کار وظیفه Action مربوط به تأیید MFA است. آن Action بعد از تأیید واقعی کد باید این مقدار را ثبت کند:

```php
$request->session()->put(
    'admin.two_factor_verified_admin_id',
    $admin->getAuthIdentifier()
);
```

هرگز این مقدار را در Login Action بلافاصله بعد از بررسی رمز عبور ثبت نکن؛ چون در آن صورت MFA عملاً بی‌اثر می‌شود.

همچنین هنگام ورود جدید، خروج، تغییر حساب و تغییر تنظیمات MFA باید این مقدار پاک شود.

## ۴. اجبار به تغییر رمز عبور

در جدول ادمین‌ها فیلد زیر را داریم:

```php
$table->boolean('must_change_password')->default(false);
```

برای مثال، وقتی ادمین جدید با دعوت‌نامه وارد سیستم می‌شود، ممکن است رمز اولیه داشته باشد و مجبور باشد رمز اختصاصی خودش را انتخاب کند.

فایل:

`app/Http/Middleware/Admin/EnsureAdminHasUpdatedPassword.php`

```php

<?php

namespace App\Http\Middleware\Admin;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

final class EnsureAdminHasUpdatedPassword
{
    public function handle(Request $request, Closure $next): Response
    {
        $admin = $request->user('admin');

        if (! $admin) {
            return redirect()->route('admin.login');
        }

        if ($admin->must_change_password) {
            return redirect()->route('admin.password.change');
        }

        return $next($request);
    }
}
```

در این طراحی، صفحه تغییر رمز نباید خودش زیر همین Middleware قرار بگیرد؛ وگرنه کاربر وارد یک چرخه Redirect بی‌نهایت می‌شود.

پس Route تغییر رمز را جداگانه تعریف می‌کنیم.

## ۵. ثبت Aliasها در bootstrap/app.php

حالا Middlewareها را به Laravel معرفی می‌کنیم.

در بالای فایل:

```bash
use App\Http\Middleware\Admin\EnsureAdminIsActive;
use App\Http\Middleware\Admin\EnsureAdminHasCompletedTwoFactor;
use App\Http\Middleware\Admin\EnsureAdminHasUpdatedPassword;
```

داخل `withMiddleware`:

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->alias([
        'admin.active' => EnsureAdminIsActive::class,
        'admin.two-factor' => EnsureAdminHasCompletedTwoFactor::class,
        'admin.password' => EnsureAdminHasUpdatedPassword::class,
    ]);
})
```

از این به بعد می‌توانی این Aliasها را در Routeها استفاده کنی.

## ۶. ساختار Routeهای Admin

حالا مهم‌ترین بخش: استفاده درست از Middlewareها.

پیشنهاد من این است که Routeهای Admin را بر اساس وضعیت امنیتی‌شان گروه‌بندی کنیم.

فایل `routes/web.php`:

```php

<?php

use Illuminate\Support\Facades\Route;

Route::prefix('admin')
    ->name('admin.')
    ->group(function () {

        /*
        |--------------------------------------------------------------------------
        | Guest Routes
        |--------------------------------------------------------------------------
        */

        Route::middleware('guest:admin')->group(function () {

            Route::livewire('/login', 'pages::admin.auth.login')
                ->name('login');

        });


        /*
        |--------------------------------------------------------------------------
        | Authenticated Admin Routes
        |--------------------------------------------------------------------------
        */

        Route::middleware('auth:admin')->group(function () {

            // Logout: available even if account becomes inactive.
            Route::post('/logout', [
                \App\Http\Controllers\Admin\Auth\LogoutController::class,
                'logout',
            ])->name('logout');


            /*
            |--------------------------------------------------------------------------
            | Active Admin Routes
            |--------------------------------------------------------------------------
            */

            Route::middleware('admin.active')->group(function () {

                // MFA setup and challenge must not use admin.two-factor.
                Route::livewire(
                    '/two-factor/setup',
                    'pages::admin.auth.two-factor.setup'
                )->name('two-factor.setup');

                Route::livewire(
                    '/two-factor/challenge',
                    'pages::admin.auth.two-factor.challenge'
                )->name('two-factor.challenge');


                /*
                |--------------------------------------------------------------------------
                | MFA Verified Routes
                |--------------------------------------------------------------------------
                */

                Route::middleware('admin.two-factor')->group(function () {

                    // Password change must be accessible even when required.
                    Route::livewire(
                        '/password/change',
                        'pages::admin.auth.password.change'
                    )->name('password.change');


                    /*
                    |--------------------------------------------------------------------------
                    | Fully Authorized Admin Area
                    |--------------------------------------------------------------------------
                    */

                    Route::middleware('admin.password')->group(function () {

                        Route::livewire(
                            '/dashboard',
                            'pages::admin.dashboard.index'
                        )->name('dashboard');

                        Route::livewire(
                            '/admins',
                            'pages::admin.admins.index'
                        )->name('admins.index');

                        Route::livewire(
                            '/support-agents',
                            'pages::admin.support-agents.index'
                        )->name('support-agents.index');

                        Route::livewire(
                            '/categories',
                            'pages::admin.categories.index'
                        )->name('categories.index');

                    });

                });

            });

        });

    });

```

در این ساختار:

ترتیب عبور درخواست از امنیت Admin

1. `auth:admin`
    
    آیا ادمین وارد شده است؟
    
2. `admin.active`
    
    آیا حساب فعال است؟
    
3. `admin.two-factor`
    
    آیا MFA در این نشست تأیید شده است؟
    
4. `admin.password`
    
    آیا تغییر رمز اجباری انجام شده است؟
    
5. اجرای صفحه
    
    دسترسی به داشبورد یا بخش موردنظر
## ۷. یک نکته مهم درباره Logout

چون قبلاً برای Tixo یک `LogoutController` طراحی کرده بودی، آن را داخل گروه `auth:admin` قرار بده، نه گروه `admin.active` یا `admin.two-factor`.

به این ترتیب حتی اگر حساب غیرفعال شود یا وضعیت MFA مشکل داشته باشد، ادمین می‌تواند از نشست خود خارج شود.

در Controller:

```php
public function logout(Request $request): RedirectResponse
{
    Auth::guard('admin')->logout();

    $request->session()->invalidate();
    $request->session()->regenerateToken();

    return redirect()->route('admin.login');
}
```

در اینجا `invalidate()` کل Session مرورگر را باطل می‌کند؛ اگر در یک Session هم‌زمان از Guard کاربر و ادمین استفاده می‌کنی، این رفتار هر دو را خارج می‌کند. اگر این موضوع در Tixo مطلوب نیست، باید سیاست Logout را جداگانه طراحی کنیم.

## ۸. قبل از استفاده در Production این موارد را بررسی کن

- Routeهای `admin.two-factor.setup` و `admin.two-factor.challenge` باید واقعاً ساخته شوند.
    
- Action تأیید MFA باید فقط پس از بررسی موفق OTP یا Recovery Code، شناسه ادمین را در Session ثبت کند.
    
- در Login جدید، مقدار قبلی `admin.two_factor_verified_admin_id` باید پاک شود.
    
- پس از تغییر رمز، `must_change_password` باید `false` شود.
    
- برای تغییر رمز و تنظیم MFA، اعتبارسنجی و Rate Limit مستقل لازم است.
    
- دسترسی به عملیات مدیریتی همچنان باید با Policy کنترل شود.