**ساختار ایجاد ادمین در پروژه** 
- جدول
- مدل
- نوع  Guard
- نحوه احراز هویت اختصاصی از Auth
- مجوزها و دسترسی‌ها(Policies)
- مبحث Middleware

## ۱. ساختار پیشنهادی جدول `admins`
### الف) اطلاعات هویتی و پروفایل

| فیلد          | نوع               | توضیح                        |
| ------------- | ----------------- | ---------------------------- |
| `id`          | BIGINT            | شناسه داخلی                  |
| `first_name`  | VARCHAR(100)      | نام                          |
| `last_name`   | VARCHAR(100)      | نام خانوادگی                 |
| `email`       | VARCHAR(255)      | ایمیل مدیر                   |
| `phone`       | VARCHAR(20) NULL  | شماره تماس                   |
| `avatar_path` | VARCHAR(255) NULL | مسیر تصویر پروفایل           |
| `job_title`   | VARCHAR(100) NULL | عنوان شغلی، مثلاً مدیر سیستم |

### ب) اطلاعات احراز هویت

| فیلد                | نوع               | توضیح            |
| ------------------- | ----------------- | ---------------- |
| `password`          | VARCHAR(255)      | رمز عبور هش‌شده  |
| `email_verified_at` | TIMESTAMP NULL    | زمان تأیید ایمیل |
| `remember_token`    | VARCHAR(100) NULL | توکن Remember Me |

اگر ورود با ایمیل انجام می‌شود، نیازی به `username` جداگانه نداری. اگر واقعاً ورود با نام کاربری هم می‌خواهی، آن را اضافه کن.

### ج) وضعیت و مدیریت حساب

|فیلد|نوع|توضیح|
|---|---|---|
|`is_active`|BOOLEAN|فعال یا غیرفعال|
|`last_login_at`|TIMESTAMP NULL|آخرین ورود موفق|
|`password_changed_at`|TIMESTAMP NULL|آخرین تغییر رمز|
|`must_change_password`|BOOLEAN|الزام تغییر رمز در ورود بعدی|
|`created_at`|TIMESTAMP|زمان ایجاد|
|`updated_at`|TIMESTAMP|زمان ویرایش|

### د) امنیت حساب مدیر

|فیلد|نوع|توضیح|
|---|---|---|
|`two_factor_secret`|TEXT NULL|اطلاعات محرمانه MFA|
|`two_factor_recovery_codes`|TEXT NULL|کدهای بازیابی MFA|
|`two_factor_confirmed_at`|TIMESTAMP NULL|زمان فعال‌سازی تأییدشده MFA|

```php
Schema::create('admins', function (Blueprint $table) {
    $table->id();

    // Identity
    $table->string('first_name', 100);
    $table->string('last_name', 100);
    $table->string('email')->unique();
    $table->string('phone', 20)->nullable();
    $table->string('avatar_path')->nullable();
    $table->string('job_title', 100)->nullable();

    // Authentication
    $table->string('password');
    $table->timestamp('email_verified_at')->nullable();
    $table->rememberToken();

    // Account status
    $table->boolean('is_active')->default(true);
    $table->timestamp('last_login_at')->nullable();
    $table->timestamp('password_changed_at')->nullable();
    $table->boolean('must_change_password')->default(false);

    // MFA
    $table->text('two_factor_secret')->nullable();
    $table->text('two_factor_recovery_codes')->nullable();
    $table->timestamp('two_factor_confirmed_at')->nullable();

    $table->timestamps();
});
```

## 2. تاریخچه امنیتی را داخل جدول admins نریز

یک اشتباه رایج این است که تمام اطلاعات امنیتی را به جدول حساب کاربری اضافه می‌کنند.

برای سیستم حرفه‌ای، بهتر است تاریخچه فعالیت‌ها جدا باشد:

جدول پیشنهادی `admin_activity_logs`:

| فیلد           | نوع               | توضیح            |
| -------------- | ----------------- | ---------------- |
| `id`           | BIGINT            | شناسه            |
| `admin_id`     | BIGINT NULL       | مدیر انجام‌دهنده |
| `event`        | VARCHAR(100)      | نوع فعالیت       |
| `subject_type` | VARCHAR(100) NULL | نوع رکورد هدف    |
| `subject_id`   | BIGINT NULL       | شناسه رکورد هدف  |
| `ip_address`   | VARCHAR(45) NULL  | IP               |
| `user_agent`   | TEXT NULL         | مشخصات مرورگر    |
| `properties`   | JSON NULL         | جزئیات تغییرات   |
| `created_at`   | TIMESTAMP         | زمان رخداد       |
```bash
admin.login
admin.logout
admin.password_changed
admin.email_changed
admin.mfa_enabled
admin.support_agent_created
admin.support_agent_disabled
admin.ticket_assigned
admin.ticket_status_changed
```

`Deepseek`
## 3. جدول لاگ (اصلی — اینجاست که باید IP ذخیره شود)
```php
Schema::create('admin_login_logs', function (Blueprint $table) {
    $table->id();
    $table->foreignId('admin_id')->nullable()->constrained('admins')->nullOnDelete();

    // Identity attempt
    $table->string('email_attempted')->nullable();

    // Network info
    $table->string('ip_address', 45);            // 45 = برای IPv6
    $table->text('user_agent')->nullable();
    $table->string('device_type', 50)->nullable();   // desktop / mobile / tablet
    $table->string('platform', 50)->nullable();      // windows / android / ios
    $table->string('browser', 50)->nullable();

    // Geo (اختیاری، اگر GeoIP داری)
    $table->string('country_code', 5)->nullable();
    $table->string('city', 100)->nullable();

    // Result
    $table->string('status', 20);                // success / failed / locked / mfa_failed
    $table->string('failure_reason')->nullable(); // wrong_password / mfa_invalid / inactive

    // Session
    $table->string('session_id', 100)->nullable();
    $table->timestamp('logged_in_at')->useCurrent();
    $table->timestamp('logged_out_at')->nullable();

    $table->timestamps();

    // Indexes برای جستجوی سریع
    $table->index(['admin_id', 'logged_in_at']);
    $table->index(['ip_address', 'logged_in_at']);
    $table->index('status');
});
```


# نقشه راه احراز هویت Admin در Tixo

این ترتیب را پیشنهاد می‌کنم:

| مرحله | موضوع                   | خروجی                                      |
| ----- | ----------------------- | ------------------------------------------ |
| ۱     | Guard و Provider        | جداسازی احراز هویت Admin از User و Support |
| ۲     | مدل و تنظیمات Admin     | اتصال صحیح مدل به سیستم Auth               |
| ۳     | ورود مدیر               | Login، اعتبارسنجی و Rate Limiting          |
| ۴     | خروج و نشست‌ها          | Logout، Session و امنیت نشست               |
| ۵     | Middleware              | محافظت از مسیرهای Admin                    |
| ۶     | بازیابی رمز عبور        | Password Reset مستقل                       |
| ۷     | تأیید ایمیل و تغییر رمز | محافظت از حساب مدیر                        |
| ۸     | MFA                     | احراز هویت چندمرحله‌ای                     |
| ۹     | Authorization           | Policy و نقش‌ها و مجوزها                   |
| ۱۰    | ثبت رویدادهای امنیتی    | Audit Log و پایش فعالیت مدیر               |

این ترتیب با مفاهیم Guard و Provider در Laravel و تفکیک Authentication از Authorization سازگار است.

**مرحله اول: Guard و Provider چیست و چرا به آن نیاز داریم؟**
در پروژه تو سه نوع حساب داریم:
- `users`: مشتریان
- `support_agents`: پشتیبان‌ها
- `admins`: مدیران
این سه گروه نباید صرفاً به خاطر اینکه همگی حساب کاربری هستند، از یک مدل و یک Provider استفاده کنند.
### ۱. Guard چیست؟
Guard مشخص می‌کند Laravel چگونه کاربر احراز هویت‌شده را در هر درخواست تشخیص دهد.
مثلاً:
```php
Auth::guard('admin')->user();
```
یعنی کاربر احراز هویت‌شده از Guard مدیر را دریافت کن.

یا:
```php
Auth::guard('web')->user();
```
یعنی کاربر احراز هویت‌شده از Guard پیش‌فرض وب را دریافت کن.

### ۲. Provider چیست؟
Provider مشخص می‌کند Laravel اطلاعات حساب را از کجا دریافت کند.
برای مثال:
```bash
admin provider
    ↓
Admin model
    ↓
admins table
```

بنابراین:
- Guard مسئول روش مدیریت احراز هویت است.
- Provider مسئول بازیابی حساب کاربری است.
این دو را با هم اشتباه نگیر؛ هر دو لازم‌اند، اما وظیفه متفاوتی دارند.

## ۳. تنظیم Guard و Provider مدیر
```bash
config/auth.php
```

الف) بخش `guards`
داخل آرایه `guards` این مورد را اضافه کن:
```php
'guards' => [
    'web' => [
        'driver' => 'session',
        'provider' => 'users',
    ],
    'admin' => [
        'driver' => 'session',
        'provider' => 'admins',
    ],
],
```

ب) بخش `providers`
داخل آرایه `providers` این مورد را اضافه کن:
```php
'providers' => [
    'users' => [
        'driver' => 'eloquent',
        'model' => App\Models\User::class,
    ],
    'admins' => [
        'driver' => 'eloquent',
        'model' => App\Models\Admin::class,
    ],
],
```

```php

<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;

class Admin extends Authenticatable
{
    protected $table = 'admins';

    protected $hidden = [
        'password',
        'remember_token',
    ];
}
```

```bash
php artisan config:clear
```

```shell
php artisan tinker
```
این دستورات را در Tinker اجرا کن:
```shell
config('auth.guards.admin');
```

خروجی مورد انتظار:
```php
[
    "driver" => "session",
    "provider" => "admins",
]
```