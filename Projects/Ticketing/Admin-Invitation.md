جدولی که ادمین برای ایجاد ادمین دیگری کاربرد دارد یعنی ادمین یک دعوت نامه میفرستد از طریق ایمیل برای شخص مورد نظر دیگر ما به سراغ ساخت حالت ورود برای عموم نرفتیم و سعی کردیم از این روش استفاده کنیم که ادمین اصلی دیگ ادمین هارو با ایجاد اولیه و فرستادن لینک دعوت به فرم ورود هدایت کنه.
**در این روش هیچ‌کس نمی‌تواند خودش حساب ادمین ایجاد کند.**

**فرایند:**
1. ادمین اصلی وارد پنل می‌شود.
    از قسمت مدیریت ادمین‌ها، ایمیل فرد جدید را وارد می‌کند.
2. سیستم دعوت‌نامه ایجاد می‌کند.
    یک لینک تصادفی، امن و دارای تاریخ انقضا تولید می‌شود.
3. فرد دعوت‌شده لینک را دریافت می‌کند.
    مثلاً از طریق ایمیل سازمانی.
4. صفحه فعال‌سازی حساب باز می‌شود.
    فرد ایمیل تأییدشده خود را می‌بیند و رمز عبور تعیین می‌کند.
5. حساب فعال می‌شود.
    دعوت‌نامه مصرف‌شده و غیرقابل استفاده مجدد می‌شود.
6. ورودهای بعدی از طریق احراز هویت معمول انجام می‌شوند.
    رمز عبور + عامل دوم احراز هویت.

## 1. نکات امنیتی که باید حتماً رعایت کنی

این بخش از طراحی مهم‌تر از ظاهر فرم ورود است.

|کنترل امنیتی|پیشنهاد برای Tixo|
|---|---|
|طول توکن|حداقل ۳۲ بایت تصادفی امن|
|ذخیره توکن|فقط Hash در دیتابیس|
|انقضای دعوت‌نامه|مثلاً ۲۴ ساعت|
|استفاده مجدد|ممنوع|
|لغو دعوت‌نامه|توسط ادمین اصلی امکان‌پذیر|
|ارسال مجدد|تولید توکن جدید و باطل کردن قبلی|
|رمز عبور|استفاده از Hash امن Laravel|
|MFA|اجباری برای ادمین‌ها|
|Rate Limiting|روی ورود، بررسی دعوت‌نامه و MFA|
|Session|تعویض شناسه نشست پس از ورود موفق|
|Audit Log|ثبت ورودها، شکست‌ها، دعوت‌ها و تغییر دسترسی|
سیاست دسترسی مدیران

**پیشنهاد نهایی**
- ایجاد حساب فقط با دعوت ادمین مجاز.
- دعوت‌نامه با اعتبار ۲۴ ساعته و یک‌بارمصرف.
- تعیین رمز عبور هنگام فعال‌سازی.
- MFA اجباری برای تمام ادمین‌ها.
- عدم وجود ثبت‌نام عمومی ادمین.
- ورودهای بعدی با رمز عبور و MFA.
- دسترسی به پنل فقط از طریق Guard مستقل `admin`.
- ثبت تمام رویدادهای امنیتی.

من یک مخالفت فنی هم با یک بخش از ایده اولیه‌ات دارم: اینکه فرم ورود فقط با یک لینک مخفی شود، امنیت واقعی ایجاد نمی‌کند. بهتر است فرم ورود عمومیِ قابل استفاده برای افراد غیرمجاز نداشته باشی، اما برای محافظت از حساب‌ها به مخفی بودن فرم وابسته نشوی. اگر واقعاً بخواهی خود فرم هم فقط پس از دریافت لینک نمایش داده شود، باید یک جریان Magic Link جداگانه طراحی کنیم.

## ۲. ساختار پیشنهادی جدول admin_invitations

من این جدول را به‌صورت یک جدول مستقل و قابل توسعه طراحی می‌کنم.

پیشنهاد نهایی ساختار

|ستون|نوع|کاربرد|
|---|---|---|
|`id`|BIGINT|شناسه دعوت‌نامه|
|`email`|VARCHAR(255)|ایمیل فرد دعوت‌شده|
|`token_hash`|CHAR(64)|هش توکن دعوت‌نامه|
|`invited_by`|BIGINT NULL|ادمین ارسال‌کننده دعوت|
|`accepted_admin_id`|BIGINT NULL|ادمینی که از این دعوت ساخته شده|
|`expires_at`|TIMESTAMP|تاریخ انقضای دعوت|
|`accepted_at`|TIMESTAMP NULL|زمان پذیرش|
|`revoked_at`|TIMESTAMP NULL|زمان لغو دعوت|
|`created_at`|TIMESTAMP|زمان ایجاد|
|`updated_at`|TIMESTAMP|زمان آخرین تغییر|
۱. `email`
ایمیل شخصی است که قرار است حساب ادمین برای او ایجاد شود.

۲. `token_hash`
توکن اصلی داخل لینک ایمیل قرار می‌گیرد، اما در دیتابیس فقط Hash آن ذخیره می‌شود.
```Plain text
Invitation URL:
https://tixo.test/admin/invitations/AbCd123...

Database:
token_hash = SHA256(AbCd123...)
```
در نتیجه اگر دیتابیس افشا شود، مهاجم مستقیماً به لینک معتبر دعوت‌نامه دسترسی ندارد.

۳. `invited_by`
مشخص می‌کند کدام ادمین این دعوت را ارسال کرده است.

۴. `accepted_admin_id`
مشخص می‌کند این دعوت‌نامه در نهایت به کدام حساب ادمین تبدیل شده است.
این ارتباط برای Audit Log و بررسی سوابق مدیریتی بسیار مفید است.

۵. `expires_at`
دعوت‌نامه بعد از زمان مشخصی دیگر معتبر نیست.

۶. `accepted_at`
مشخص می‌کند دعوت‌نامه پذیرفته شده است یا خیر.

۷. `revoked_at`
اگر ادمین دعوت‌کننده بخواهد قبل از پذیرش، دعوت را لغو کند، این فیلد مقدار می‌گیرد.
نکته مهم: من عمداً فیلد `status` اضافه نکردم؛ چون وضعیت را می‌توان از روی این سه فیلد محاسبه کرد:
- `accepted_at`
- `revoked_at`
- `expires_at`

وجود هم‌زمان `status` و این فیلدها باعث تکرار اطلاعات و احتمال ناسازگاری می‌شود.

`migration`
```php
public function up(): void
{
    Schema::create('admin_invitations', function (Blueprint $table) {
        $table->id();

        $table->string('email')->index();

        $table->char('token_hash', 64)->unique();

        // Invitation Relations
        $table->foreignId('invited_by')
            ->nullable()
            ->constrained('admins')
            ->nullOnDelete();

        $table->foreignId('accepted_admin_id')
            ->nullable()
            ->unique()
            ->constrained('admins')
            ->nullOnDelete();

        // Invitation Lifecycle
        $table->timestamp('expires_at');
        $table->timestamp('accepted_at')->nullable();
        $table->timestamp('revoked_at')->nullable();

        $table->timestamps();
    });
}
```

`accepted_admin_id` به دلیل `unique()` تضمین می‌کند هر حساب ادمین حداکثر به یک دعوت‌نامه پذیرفته‌شده متصل شود.

برای ایمیل عمداً `unique()` نگذاشتم؛ چون باید تاریخچه دعوت‌های قبلی حفظ شود و ممکن است برای یک ایمیل چند دعوت‌نامه در زمان‌های مختلف ایجاد شود.

**نحوه تولید توکن**
در Laravel می‌توانی از `Str::random()` استفاده کنی:
```php
use Illuminate\Support\Str;

$token = Str::random(64);
$tokenHash = hash('sha256', $token);
```

در مدل `Admin`:
```php
public function sentInvitations()
{
    return $this->hasMany(
        AdminInvitation::class,
        'invited_by'
    );
}
```

مدل`AdminInvitation`
```php
public function inviter()
{
    return $this->belongsTo(
        Admin::class,
        'invited_by'
    );
}

public function acceptedAdmin()
{
    return $this->belongsTo(
        Admin::class,
        'accepted_admin_id'
    );
}
```



# نقشه راه پیاده‌سازی دعوت ادمین

1. ساختار و آماده‌سازی
    بررسی Modelها، روابط، مسیرها و معماری فایل‌ها.
    
    مرحله فعلی

1. ساخت DTO
    تعریف داده‌هایی که برای ایجاد دعوت‌نامه نیاز داریم.
    
2. ساخت Action
    پیاده‌سازی `CreateAdminInvitationAction`، تولید توکن، ذخیره Hash و Transaction.
    
3. ساخت Livewire Form
    اعتبارسنجی ایمیل و آماده‌سازی داده‌های فرم.

4. ساخت کامپوننت Livewire MFC
    دریافت اطلاعات، اجرای Action و نمایش نتیجه.
    
5. ارسال ایمیل دعوت‌نامه
    ساخت Mailable و ارسال لینک امن.
    
6. مدیریت دعوت‌نامه‌ها
    لغو دعوت، ارسال مجدد، بررسی دعوت‌های فعال و نمایش وضعیت‌ها.

## مرحله اول: معماری عملیات دعوت

قبل از کدنویسی باید جریان عملیات را مشخص کنیم.
فرض کن مدیر در پنل Tixo به قسمت مدیریت ادمین‌ها می‌رود و ایمیل فرد جدید را وارد می‌کند.

![[Screenshot 2026-11-03 033417 1.png]]

**یک تصمیم مهم:** ارسال ایمیل را مستقیماً داخل Action ایجاد دعوت‌نامه قرار نمی‌دهیم. ابتدا دعوت‌نامه در دیتابیس با موفقیت ساخته می‌شود و سپس ایمیل ارسال می‌شود. در ادامه برای ارسال قابل اطمینان‌تر می‌توانیم از Queue و الگوی Outbox استفاده کنیم.

## مرحله اول: ساخت DTO

در این مرحله فقط DTO را می‌سازیم. هنوز سراغ Action و Livewire نمی‌رویم.

### ۱. DTO ما چه اطلاعاتی دریافت می‌کند؟
برای ایجاد دعوت‌نامه فعلاً فقط به یک داده از فرم نیاز داریم:
```bash
email
```

**اما یک نکته معماری مهم وجود دارد:**
شناسه ادمینی که دعوت‌نامه را ارسال می‌کند (`invited_by`) نباید از فرم دریافت شود؛ چون کاربر می‌تواند اطلاعات ارسالی را دست‌کاری کند.

این اطلاعات باید از نشست احراز هویت ادمین دریافت شود:
```php
auth('admin')->user()
```

بنابراین مسئولیت‌ها را این‌طور تقسیم می‌کنیم:

- **DTO:** نگهداری ایمیل دعوت‌شونده.
- **Livewire:** دریافت و اعتبارسنجی فرم.
- **Action:** اجرای عملیات دعوت.
- **Auth:** شناسایی ادمین دعوت‌کننده.

### ۲. ایجاد فایل DTO

ساختار پیشنهادی:
```plain text
app/
├── Actions/
│   └── AdminInvitations/
│       └── CreateAdminInvitationAction.php
│
├── DTOs/
│   └── AdminInvitations/
│       └── CreateAdminInvitationData.php
│
├── Models/
│   ├── Admin.php
│   └── AdminInvitation.php
│
└── Livewire/
```

در ترمینال پوشه را بساز:
```Bash
mkdir app/DTOs/AdminInvitations
```

سپس فایل زیر را ایجاد کن:

`app/DTOs/AdminInvitations/CreateAdminInvitationData.php`

کد:

```php

<?php

namespace App\DTOs\AdminInvitations;

readonly class CreateAdminInvitationData
{
    public function __construct(
        public string $email,
    ) {}
}

```

### ۳. چرا از readonly استفاده کردیم؟

در PHP، کلمه `readonly` باعث می‌شود بعد از مقداردهی اولیه، مقدار Property قابل تغییر نباشد.

مثلاً:

```php
$data = new CreateAdminInvitationData(
    email: 'admin@example.com'
);
```

می‌توانیم ایمیل را بخوانیم:

```php
$data->email;
```

اما نمی‌توانیم بعداً آن را تغییر دهیم:

```php
$data->email = 'another@example.com'; // Error
```

این رفتار برای DTO مناسب است؛ چون DTO قرار است یک بسته اطلاعاتی معتبر و مشخص را بین لایه‌های برنامه منتقل کند.

### ۴. فعلاً چه چیزی را پیاده‌سازی نکردیم؟

عمداً این موارد را وارد DTO نکردیم:

- `invited_by`
- `token_hash`
- `expires_at`
- `accepted_at`
- `revoked_at`

چرا؟
چون این‌ها اطلاعات داخلی عملیات دعوت هستند و توسط Action تولید یا مدیریت می‌شوند، نه توسط فرم کاربر.

# مرحله دوم: ساخت CreateAdminInvitationAction

در مرحله بعد از همین رابطه‌ای که یاد گرفتی استفاده خواهیم کرد:

```php
$inviter->sentInvitations()->create([...]);
```

## ۱. مسئولیت Action چیست؟

Action ما باید یک عملیات مشخص را انجام دهد:

`Create Admin Invitation`

وظایف آن:

1. دریافت اطلاعات از DTO.
    
2. دریافت ادمینی که دعوت‌نامه را ارسال می‌کند.
    
3. نرمال‌سازی ایمیل.
    
4. بررسی وجود حساب ادمین با این ایمیل.
    
5. بررسی دعوت‌نامه فعال قبلی.
    
6. تولید توکن تصادفی.
    
7. ذخیره Hash توکن در دیتابیس.
    
8. تعیین تاریخ انقضا.
    
9. ایجاد دعوت‌نامه در دیتابیس.
فعلاً ارسال ایمیل را انجام نمی‌دهیم؛ آن را در مرحله بعد جداگانه پیاده‌سازی می‌کنیم.
## ۲. یک تصمیم مهم: خروجی Action

قبل از نوشتن Action باید یک مسئله را حل کنیم.

ما توکن را به شکل Hash در دیتابیس ذخیره می‌کنیم:
```php
$tokenHash = hash('sha256', $token);
```

اما برای ساخت لینک دعوت‌نامه به توکن اصلی نیاز داریم.

پس Action باید دو خروجی در اختیار ما قرار دهد:

- رکورد دعوت‌نامه ایجادشده.
    
- توکن اصلی برای ارسال ایمیل.
    

به همین دلیل یک DTO خروجی هم ایجاد می‌کنیم.
ساختار پوشه:
```Plain text
app/
├── Actions/
│   └── AdminInvitations/
│       └── CreateAdminInvitationAction.php
│
├── DTOs/
│   └── AdminInvitations/
│       ├── CreateAdminInvitationData.php
│       └── CreateAdminInvitationResultData.php
│
└── Models/
    ├── Admin.php
    └── AdminInvitation.php
```

## ۳. ایجاد DTO خروجی

فایل:
`app/DTOs/AdminInvitations/CreateAdminInvitationResultData.php`

```php

<?php

namespace App\DTOs\AdminInvitations;

use App\Models\AdminInvitation;

readonly class CreateAdminInvitationResultData
{
    public function __construct(
        public AdminInvitation $invitation,
        public string $token,
    ) {}
}

```

چرا دو DTO داریم؟

|DTO|مسئولیت|
|---|---|
|`CreateAdminInvitationData`|انتقال اطلاعات ورودی|
|`CreateAdminInvitationResultData`|انتقال نتیجه عملیات|
این تفکیک باعث می‌شود ورودی و خروجی Action با یکدیگر مخلوط نشوند.

نکته امنیتی: توکن اصلی فقط در حافظه برنامه و برای ارسال ایمیل نگهداری می‌شود. نباید آن را در دیتابیس، Log یا خروجی JSON عمومی قرار بدهیم.

## ۴. ایجاد Action

ابتدا پوشه را ایجاد کن:
```Bash
mkdir app/Actions/AdminInvitations
```

سپس فایل:

`app/Actions/AdminInvitations/CreateAdminInvitationAction.php`

کد کامل:
```php

<?php

namespace App\Actions\AdminInvitations;

use App\DTOs\AdminInvitations\CreateAdminInvitationData;
use App\DTOs\AdminInvitations\CreateAdminInvitationResultData;
use App\Models\Admin;
use App\Models\AdminInvitation;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Facades\ValidationException;
use Illuminate\Support\Str;

class CreateAdminInvitationAction
{
    public function handle(
        CreateAdminInvitationData $data,
        Admin $inviter,
    ): CreateAdminInvitationResultData {

        return DB::transaction(function () use ($data, $inviter) {

            // 1. Normalize email
            $email = Str::lower(trim($data->email));

            // 2. Check existing admin
            $adminExists = Admin::query()
                ->where('email', $email)
                ->exists();

            if ($adminExists) {
                throw ValidationException::withMessages([
                    'email' => 'این ایمیل قبلاً برای یک ادمین ثبت شده است.',
                ]);
            }

            // 3. Check active invitation
            $activeInvitation = AdminInvitation::query()
                ->where('email', $email)
                ->whereNull('accepted_at')
                ->whereNull('revoked_at')
                ->where('expires_at', '>', now())
                ->exists();

            if ($activeInvitation) {
                throw ValidationException::withMessages([
                    'email' => 'برای این ایمیل قبلاً دعوت‌نامه فعال ارسال شده است.',
                ]);
            }

            // 4. Generate secure token
            $token = Str::random(64);

            // 5. Hash token
            $tokenHash = hash('sha256', $token);

            // 6. Create invitation
            $invitation = $inviter->sentInvitations()->create([
                'email' => $email,
                'token_hash' => $tokenHash,
                'expires_at' => now()->addHours(24),
            ]);

            // 7. Return result
            return new CreateAdminInvitationResultData(
                invitation: $invitation,
                token: $token,
            );
        });
    }
}

```


# ۵. بررسی منطق Action؛ چرا این‌طور نوشتیم؟

حالا قسمت‌های مهم را بررسی کنیم.

### الف) چرا handle() دو پارامتر می‌گیرد؟

```php
public function handle(
    CreateAdminInvitationData $data,
    Admin $inviter,
)
```

پارامتر اول اطلاعات دعوت‌نامه است.

پارامتر دوم ادمینی است که دعوت را ایجاد می‌کند.

چرا ادمین را از داخل Action با `auth()` دریافت نکردیم؟

چون Action نباید به محیط اجرای Livewire یا Session وابسته باشد.

فرض کن فردا همین Action را از یک API اجرا کردی. در آن صورت هم می‌توانی ادمین احراز هویت‌شده را به آن تحویل بدهی.

این یعنی وابستگی کمتر و قابلیت استفاده مجدد بیشتر.

### ب) چرا DB::transaction استفاده کردیم؟
```php
return DB::transaction(function () {
    // ...
});
```

فرض کن سه عملیات داریم:

1. بررسی ایمیل.
    
2. بررسی دعوت قبلی.
    
3. ایجاد دعوت‌نامه.
    

اگر هنگام ذخیره اطلاعات خطایی رخ دهد، Transaction باعث می‌شود تغییرات دیتابیس به‌صورت ناقص باقی نمانند.

نکته: در این Action فعلاً فقط یک INSERT داریم؛ Transaction برای حفظ مرز عملیات تجاری و توسعه بعدی مفید است. در مرحله پذیرش دعوت، که ایجاد Admin و مصرف دعوت‌نامه با هم انجام می‌شوند، اهمیت آن حتی بیشتر خواهد شد.

### ج) چرا دعوت‌نامه قبلی را حذف نمی‌کنیم؟

```php
$activeInvitation = AdminInvitation::query()
    ->where('email', $email)
    ->whereNull('accepted_at')
    ->whereNull('revoked_at')
    ->where('expires_at', '>', now())
    ->exists();
```

اگر برای ایمیل موردنظر دعوت‌نامه فعال وجود داشته باشد، عملیات را متوقف می‌کنیم.

چرا؟

چون نمی‌خواهیم چند لینک معتبر برای یک ایمیل داشته باشیم.

بعداً یک Action مستقل به نام:
```Plain text
ResendAdminInvitationAction
```

می‌سازیم که وظیفه‌اش لغو دعوت قبلی و ایجاد دعوت جدید است.

### د) چرا از رابطه sentInvitations استفاده کردیم؟

```php
$inviter->sentInvitations()->create([...]);
```

این همان رابطه‌ای است که در مرحله قبل یاد گرفتی.

لاراول خودش مقدار زیر را تنظیم می‌کند:

```php
'invited_by' => $inviter->id
```

پس لازم نیست Foreign Key را دستی وارد کنیم.

### هـ) چرا از Hash استفاده نکردیم؟

```php
$tokenHash = hash('sha256', $token);
```

چون توکن یک مقدار تصادفی و با آنتروپی بالا است و قرار نیست از آن برای رمز عبور استفاده کنیم.

برای رمز عبور از Hashing اختصاصی Laravel استفاده می‌کنیم؛ برای توکن تصادفی دعوت‌نامه، SHA-256 انتخاب مناسبی است.

# ۶. یک نکته امنیتی مهم برای نسخه نهایی

این Action هنوز یک لایه امنیتی تکمیلی لازم دارد:

- مجوز دعوت ادمین باید با Policy یا Gate کنترل شود.
    
- ادمین غیرفعال نباید امکان دعوت داشته باشد.
    
- برای جلوگیری از درخواست‌های هم‌زمان و ایجاد دعوت‌نامه تکراری باید کنترل هم‌زمانی در نظر بگیریم.
    
- خطای وجود ایمیل ادمین را در رابط عمومی طوری مدیریت کنیم که اطلاعات حساب‌ها افشا نشود.
    

این موارد را هنگام اتصال به Livewire و نهایی‌سازی Action بررسی می‌کنیم.