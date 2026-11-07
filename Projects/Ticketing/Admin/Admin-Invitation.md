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
use Illuminate\Validation\ValidationException;
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

**یک نکته معماری برای آینده:** بررسی دعوت‌نامه فعال با `exists()` به‌تنهایی جلوی درخواست‌های هم‌زمان را نمی‌گیرد. ممکن است دو درخواست هم‌زمان هر دو بررسی را پشت سر بگذارند و دو دعوت‌نامه بسازند. در نسخه نهایی باید این Race Condition را هم کنترل کنیم.

# مرحله ۳: ساخت Form Object در Livewire

حالا باید ورودی کاربر را از منطق Action جدا کنیم.

در این مرحله می‌خواهیم یک Form Object بسازیم که مسئول این موارد باشد:

- نگهداری ایمیل واردشده.
    
- اعتبارسنجی ایمیل.
    
- آماده‌سازی داده برای ارسال به Action.


## ۱. ایجاد فایل Form

مسیر پیشنهادی:

```
app/
└── Livewire/
    └── Forms/
        └── Admin/
            └── CreateAdminInvitationForm.php
```

کد:
```php

<?php

namespace App\Livewire\Forms\Admin;

use App\DTOs\Admin\AdminInvitation\CreateAdminInvitationData;
use Livewire\Form;

class CreateAdminInvitationForm extends Form
{
    public string $email = '';

    public function rules(): array
    {
        return [
            'email' => [
                'required',
                'string',
                'email',
                'max:255',
            ],
        ];
    }

    public function toDTO(): CreateAdminInvitationData
    {
        return new CreateAdminInvitationData(
            email: $this->email,
        );
    }
}

```


### چرا این متدها را داریم؟

`rules()`

قوانین اعتبارسنجی ورودی فرم را مشخص می‌کند. فعلاً بررسی نمی‌کنیم که ایمیل قبلاً ادمین شده یا دعوت‌نامه فعال دارد؛ این بررسی‌ها در Action باقی می‌مانند، چون بخشی از منطق کسب‌وکار هستند.

`toDTO()`

داده فرم را به DTO تبدیل می‌کند. به این ترتیب Action مستقیماً به Livewire وابسته نمی‌شود.


![[Screenshot 2026-11-03 053624.png]]

## مرحله اول: ساخت Mailable

در Laravel هر نوع ایمیل را بهتر است با یک کلاس مستقل از نوع Mailable مدیریت کنیم. این کار باعث می‌شود منطق ارسال از قالب HTML ایمیل جدا باشد.


ابتدا اجرا کن:

```
php artisan make:mail Admin/AdminInvitationMail
```

مسیر فایل:

```Plain text
app/
└── Mail/
    └── Admin/
        └── AdminInvitationMail.php
```

```php
<?php

namespace App\Mail\Admin;

use App\Models\AdminInvitation;
use Illuminate\Bus\Queueable;
use Illuminate\Mail\Mailable;
use Illuminate\Mail\Mailables\Content;
use Illuminate\Mail\Mailables\Envelope;
use Illuminate\Queue\SerializesModels;

final class AdminInvitationMail extends Mailable
{
    use Queueable, SerializesModels;

    public function __construct(
        public readonly AdminInvitation $invitation,
        private readonly string $token,
    ) {}

    public function envelope(): Envelope
    {
        return new Envelope(
            subject: 'دعوت‌نامه مدیریت تیکسو',
        );
    }

    public function content(): Content
    {
        return new Content(
            view: 'emails.admin-invitations.invitation',
            with: [
                'acceptUrl' => route(
                    'admin.invitations.accept',
                    ['token' => $this->token]
                ),
            ],
        );
    }
}
```

چند نکته معماری:

- مدل دعوت‌نامه و توکن به Mailable داده می‌شوند.
    
- توکن به‌صورت `private` نگهداری می‌شود تا جزو متغیرهای عمومی قالب ایمیل قرار نگیرد.
    
- لینک پذیرش با نام Route ساخته می‌شود، نه با نوشتن دستی دامنه سایت.
    
- این Mailable فعلاً ارسال هم‌زمان دارد؛ بعد از تنظیم Queue آن را به ارسال پس‌زمینه تبدیل می‌کنیم.
    

## مرحله دوم: طراحی قالب ایمیل

این مسیر را ایجاد کن:

```
resources/
└── views/
    └── emails/
        └── admin-invitations/
            └── invitation.blade.php
```

محتوای فایل:

```html
<!DOCTYPE html>
<html lang="fa" dir="rtl">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">

    <title>دعوت به مدیریت تیکسو</title>
</head>

<body style="
    margin: 0;
    padding: 30px 15px;
    background: #f5f6fa;
    font-family: Tahoma, Arial, sans-serif;
    direction: rtl;
">

    <div style="
        max-width: 560px;
        margin: 0 auto;
        padding: 32px;
        background: #ffffff;
        border: 1px solid #e5e7eb;
        border-radius: 16px;
    ">

        <h2 style="color: #111827;">
            دعوت به پنل مدیریت تیکسو
        </h2>

        <p style="color: #4b5563; line-height: 2;">
            سلام،
        </p>

        <p style="color: #4b5563; line-height: 2;">
            شما توسط
            <strong>
                {{ $invitation->inviter?->full_name ?? 'یکی از مدیران تیکسو' }}
            </strong>
            برای پیوستن به تیم مدیریت تیکسو دعوت شده‌اید.
        </p>

        <p style="color: #4b5563; line-height: 2;">
            برای تکمیل ثبت‌نام و ایجاد حساب کاربری مدیر، روی دکمه زیر کلیک کنید.
        </p>

        <div style="margin: 30px 0;">
            <a
                href="{{ $acceptUrl }}"
                style="
                    display: inline-block;
                    padding: 13px 25px;
                    color: #ffffff;
                    background: #4f46e5;
                    border-radius: 10px;
                    text-decoration: none;
                    font-weight: bold;
                "
            >
                پذیرش دعوت و ایجاد حساب
            </a>
        </div>

        <p style="color: #6b7280; font-size: 13px; line-height: 2;">
            این دعوت‌نامه تا
            {{ $invitation->expires_at->format('Y/m/d H:i') }}
            معتبر است.
        </p>

        <hr style="border: 0; border-top: 1px solid #e5e7eb;">

        <p style="color: #9ca3af; font-size: 12px; line-height: 2;">
            چنانچه انتظار دریافت این دعوت‌نامه را نداشتید، می‌توانید این ایمیل را نادیده بگیرید.
        </p>

    </div>

</body>
</html>
```

## مرحله سوم: تعریف Route پذیرش دعوت‌نامه

چون فرد دعوت‌شده هنوز حساب مدیر ندارد، لینک دعوت نباید پشت Middleware مربوط به `auth:admin` قرار بگیرد.

در `routes/web.php`، بیرون از گروه احراز هویت مدیران، این Route را تعریف کن:
```php
Route::livewire(
    '/admin/invitations/{token}',
    'pages::admin.invitations.accept'
)->name('admin.invitations.accept');
```

فعلاً کامپوننت پذیرش را لازم نیست بسازی؛ در مرحله بعد آن را ایجاد می‌کنیم.

## مرحله چهارم: اتصال Mailable به Livewire

حالا کامپوننت فعلی‌ات را اصلاح کن.

بالای فایل اضافه کن:
```php
use App\Mail\Admin\AdminInvitationMail;
use Illuminate\Support\Facades\Mail;
use Throwable;
```

متد `createInvitation` را به این شکل تغییر بده:

```php
public function createInvitation(CreateAdminInvitationAction $action): void
{
    $this->form->validate();

    $inviter = Auth::guard('admin')->user();

    abort_unless($inviter instanceof Admin, 403);

    $result = $action->handle(
        data: $this->form->toDTO(),
        inviter: $inviter,
    );

    try {

        Mail::to($result->invitation->email)->send(
            new AdminInvitationMail(
                invitation: $result->invitation,
                token: $result->token,
            )
        );

    } catch (Throwable $exception) {

        report($exception);

        // با شکست ارسال ایمیل، دعوت قبلی را غیرفعال کن.
        $result->invitation
            ->forceFill(['revoked_at' => now()])
            ->save();

        $this->addError(
            'form.email',
            'ارسال ایمیل دعوت‌نامه ناموفق بود. لطفاً مجدداً تلاش کنید.'
        );

        return;
    }

    $this->form->reset();
    $this->resetValidation();

    session()->flash(
        'success',
        'دعوت‌نامه با موفقیت ارسال شد.'
    );
}
```

چرا ایمیل را داخل Action قبلی قرار ندادیم؟

چون `CreateAdminInvitationAction` مسئول ایجاد رکورد دعوت‌نامه است. ارسال ایمیل یک عملیات جانبی مستقل است.

همچنین دقت کن که `handle()` بعد از پایان تراکنش دیتابیس نتیجه را برمی‌گرداند؛ بنابراین ارسال ایمیل بیرون از تراکنش انجام می‌شود و زمان ارتباط با SMTP را داخل تراکنش دیتابیس تلف نمی‌کنیم.

## دو نکته مهم درباره مدل فعلی تو

۱. موضوع Fillable

در مدل `AdminInvitation` نوشتی:

```php
#[Fillable('email', 'token_hash', 'expires_at')]
```

این تصمیم برای جلوگیری از تغییر مستقیم فیلدهای حساس از طریق Mass Assignment مناسب است.

اما در مراحل بعد باید بتوانیم این ستون‌ها را هم تغییر دهیم:

```
accepted_at
revoked_at
accepted_admin_id
```

پیشنهاد من این است که این فیلدها را همچنان از ورودی کاربر قابل Mass Assignment نکنیم و در Actionهای داخلی با `forceFill()` تغییر دهیم.

به همین دلیل در کد بالا برای لغو دعوت‌نامه از `forceFill()` استفاده کردیم.

۲. موضوع ارسال ایمیل در محیط واقعی

فعلاً از `send()` استفاده کردیم تا ابتدا صحت تنظیمات SMTP را بررسی کنیم. در محیط Production بهتر است ارسال ایمیل را با Queue انجام بدهیم تا پاسخ Livewire منتظر سرور ایمیل نماند. Laravel این قابلیت را از طریق `queue()` و `ShouldQueue` فراهم کرده است.


## حالا قبل از رفتن به مرحله پذیرش دعوت‌نامه
  

بعد از بررسی این موارد می‌رویم سراغ مرحله مهم بعدی:

مرحله بعد: Accept Admin Invitation

1. دریافت توکن از URL
    
2. محاسبه Hash و پیدا کردن دعوت‌نامه
    
3. بررسی انقضا، لغو و استفاده قبلی
    
4. نمایش فرم تکمیل اطلاعات مدیر
    
5. ساخت Admin داخل تراکنش
    
6. ثبت `accepted_at` و `accepted_admin_id`
    
7. ورود مدیر و هدایت به پنل Tixo
    

یک اصل امنیتی را هم از همین الان حفظ می‌کنیم: داشتن لینک دعوت به‌تنهایی نباید باعث ورود خودکار به پنل شود؛ ابتدا باید دعوت معتبر باشد و حساب مدیر با موفقیت ساخته شود.

حتماً. چون اولین بار است در پروژه Tixo ایمیل ارسال می‌کنی، می‌خواهم SMTP را از پایه و به شکل استاندارد Production یاد بگیریم، نه اینکه صرفاً چند خط داخل `.env` قرار بدهیم.

هدف ما این است که در نهایت:

1. Laravel 13 بتواند ایمیل ارسال کند.
    
2. ایمیل دعوت مدیر جدید واقعاً به Inbox برسد.
    
3. خطاهای SMTP را بتوانیم تشخیص دهیم.
    
4. اطلاعات محرمانه ایمیل در Git و کد پروژه افشا نشود.
    
5. بعداً بتوانیم ارسال ایمیل را با Queue انجام بدهیم.
    

## ۱. SMTP چیست؟

SMTP مخفف Simple Mail Transfer Protocol است.

به زبان ساده، Laravel خودش سرویس ایمیل نیست. برای ارسال ایمیل باید به یک سرور ایمیل متصل شود.![[Screenshot 2026-11-07 015036.png]]

## ۲. انتخاب سرویس ایمیل

چند انتخاب داریم:

- Gmail: مناسب یادگیری و تست اولیه.
    
- ایمیل دامنه اختصاصی: مثل `noreply@tixo.com`، مناسب برند Tixo.
    
- سرویس‌های ارسال ایمیل مثل Brevo یا Mailgun: مناسب ارسال انبوه و Production.
    

برای یک سیستم حرفه‌ای بهتر است در نهایت از ایمیل دامنه اختصاصی استفاده کنیم.

## مرحله دوم: تنظیم فایل `.env`

در ریشه پروژه Tixo، فایل `.env` را باز کن.

تنظیمات قبلی مربوط به Mail را پیدا کن و با این مقادیر جایگزین کن:

```env
MAIL_MAILER=smtp

MAIL_SCHEME=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587

MAIL_USERNAME=your-email@gmail.com
MAIL_PASSWORD=your-16-character-app-password

MAIL_FROM_ADDRESS=your-email@gmail.com
MAIL_FROM_NAME="Tixo"
```

مقادیر زیر را با اطلاعات واقعی خودت جایگزین کن:

|متغیر|مقدار|
|---|---|
|`MAIL_USERNAME`|آدرس کامل Gmail|
|`MAIL_PASSWORD`|App Password|
|`MAIL_FROM_ADDRESS`|همان Gmail|
|`MAIL_FROM_NAME`|نام نمایشی Tixo|

چرا `MAIL_FROM_ADDRESS` را برابر Gmail قرار دادیم؟

چون فعلاً Gmail فرستنده ایمیل‌های ماست. بهتر است آدرس فرستنده با حساب احراز هویت‌شده یکسان باشد.

تنظیم `MAIL_SCHEME=smtp` با پورت ۵۸۷ از اتصال SMTP و ارتقای ارتباط به TLS پشتیبانی می‌کند. Gmail برای این حالت از STARTTLS استفاده می‌کند.

GitHub+1

امنیت

فایل `.env` نباید وارد Git شود. رمز App Password را داخل کد PHP یا فایل‌های عمومی پروژه قرار نده.

## مرحله سوم: بررسی `config/mail.php`

فایل زیر را باز کن:

```
config/mail.php
```

در قسمت `mailers` بررسی کن که تنظیم SMTP به شکل زیر باشد:

```php
'smtp' => [
    'transport' => 'smtp',
    'scheme' => env('MAIL_SCHEME'),
    'url' => env('MAIL_URL'),
    'host' => env('MAIL_HOST', '127.0.0.1'),
    'port' => env('MAIL_PORT', 2525),
    'username' => env('MAIL_USERNAME'),
    'password' => env('MAIL_PASSWORD'),
    'timeout' => null,
    'local_domain' => env(
        'MAIL_EHLO_DOMAIN',
        parse_url(
            (string) env('APP_URL', 'http://localhost'),
            PHP_URL_HOST
        )
    ),
],
```
این ساختار با فایل پیش‌فرض Laravel 13 هماهنگ است. اگر همین ساختار را داری، لازم نیست چیزی را تغییر بدهی.


## مرحله چهارم: پاک‌کردن Cache تنظیمات

بعد از ذخیره `.env`، ترمینال پروژه را باز کن و اجرا کن:

```bash
php artisan optimize:clear
```

این دستور Cache تنظیمات و سایر Cacheهای بهینه‌سازی Laravel را پاک می‌کند تا مقادیر جدید خوانده شوند.

## مرحله پنجم: ارسال اولین ایمیل آزمایشی

برای اینکه مطمئن شویم SMTP واقعاً کار می‌کند، از Tinker استفاده می‌کنیم.

در ترمینال:

```
php artisan tinker
```

سپس این کد را اجرا کن:

```
use Illuminate\Support\Facades\Mail;

Mail::raw(
    'This is a test email from Tixo Laravel application.',
    function ($message) {
        $message
            ->to('YOUR-TEST-EMAIL@gmail.com')
            ->subject('Tixo SMTP Test');
    }
);
```

آدرس گیرنده را با ایمیلی که به آن دسترسی داری جایگزین کن.

اگر خطایی دریافت نکردی، ارسال توسط SMTP انجام شده است. حالا Inbox و پوشه Spam را بررسی کن.

برای خروج از Tinker:

```
exit
```

## اگر خطا دریافت کردی

چند خطای رایج:

|خطا|علت احتمالی|
|---|---|
|`535 Authentication failed`|App Password اشتباه یا نامعتبر|
|`Connection timed out`|مشکل اتصال شبکه یا پورت|
|`Connection refused`|اتصال به سرور برقرار نشده|
|`Username and Password not accepted`|مشکل احراز هویت Gmail|
|`STARTTLS` error|مشکل مذاکره TLS|

## ۱. بررسی ساختار دیتابیس

|ستون|ارزیابی|
|---|---|
|`email`|مناسب، برای پیدا کردن دعوت‌نامه|
|`token_hash`|بسیار خوب؛ هش ۶۴ کاراکتری و Unique|
|`invited_by`|مناسب؛ با حذف دعوت‌کننده مقدار NULL می‌شود|
|`accepted_admin_id`|مناسب؛ هر مدیر فقط به یک دعوت‌نامه پذیرفته‌شده متصل می‌شود|
|`expires_at`|مناسب|
|`accepted_at`|مناسب|
|`revoked_at`|مناسب|

یک نکته مهم: `accepted_admin_id` با `unique()` باعث می‌شود هر مدیر فقط یک دعوت‌نامه پذیرفته‌شده داشته باشد. این محدودیت با طراحی فعلی تو سازگار است.

اما یک مسئله داریم: دیتابیس فعلاً مانع ثبت چند دعوت‌نامه فعال برای یک ایمیل نمی‌شود. این موضوع را در Action و در صورت نیاز با قیدهای دیتابیس مدیریت می‌کنیم.

## ۲. جریان پذیرش دعوت‌نامه

این معماری را پیاده می‌کنیم:![[Screenshot 2026-11-07 015238.png]]

نکته امنیتی مهم: برای جلوگیری از پذیرش هم‌زمان یک دعوت‌نامه، از `lockForUpdate()` داخل تراکنش استفاده می‌کنیم. همچنین توکن خام فقط در URL و زمان اعتبارسنجی استفاده می‌شود و هرگز در دیتابیس ذخیره نخواهد شد.

حالا می‌رویم سراغ پیاده‌سازی واقعی. من پیشنهاد می‌کنم پذیرش دعوت‌نامه را به یک Action مستقل تبدیل کنیم تا هم Livewire و هم در آینده API بتوانند از همان منطق استفاده کنند.

# مرحله سوم: پیاده‌سازی پذیرش دعوت‌نامه

ساختار موردنظر:
```Plain text
app/
├── Actions/
│   └── Admin/
│       └── AdminInvitations/
│           ├── CreateAdminInvitationAction.php
│           └── AcceptAdminInvitationAction.php
│
├── DTOs/
│   └── Admin/
│       └── AdminInvitation/
│           ├── CreateAdminInvitationData.php
│           ├── CreateAdminInvitationResultData.php
│           └── AcceptAdminInvitationData.php
│
├── Livewire/
│   └── Forms/
│       └── Admin/
│           └── AcceptAdminInvitationForm.php
```

## ۱. ساخت DTO

فایل:

`app/DTOs/Admin/AdminInvitation/AcceptAdminInvitationData.php`

```php
<?php

namespace App\DTOs\Admin\AdminInvitation;

readonly class AcceptAdminInvitationData
{
    public function __construct(
        public string $firstName,
        public string $lastName,
        public string $password,
    ) {}
}
```

ایمیل را عمداً داخل DTO قرار ندادیم.

چرا؟

چون ایمیل مدیر باید از دعوت‌نامه معتبر دریافت شود، نه از اطلاعاتی که کاربر در فرم ارسال می‌کند. این کار جلوی تغییر ایمیل دعوت‌نامه توسط کاربر را می‌گیرد.

## ۲. ساخت AcceptAdminInvitationAction

فایل:
`app/Actions/Admin/AdminInvitations/AcceptAdminInvitationAction.php`
```php
<?php

namespace App\Actions\Admin\AdminInvitations;

use App\DTOs\Admin\AdminInvitation\AcceptAdminInvitationData;
use App\Models\Admin;
use App\Models\AdminInvitation;
use Illuminate\Support\Facades\DB;
use Illuminate\Validation\ValidationException;

final class AcceptAdminInvitationAction
{
    public function handle(
        int $invitationId,
        AcceptAdminInvitationData $data,
    ): Admin {

        return DB::transaction(function () use ($invitationId, $data) {

            // Lock invitation
            $invitation = AdminInvitation::query()
                ->lockForUpdate()
                ->find($invitationId);

            if (!$invitation) {
                throw ValidationException::withMessages([
                    'invitation' => 'دعوت‌نامه معتبر نیست.',
                ]);
            }

            // Check invitation lifecycle
            if (
                $invitation->accepted_at ||
                $invitation->revoked_at ||
                $invitation->expires_at->isPast()
            ) {
                throw ValidationException::withMessages([
                    'invitation' => 'این دعوت‌نامه منقضی یا قبلاً استفاده شده است.',
                ]);
            }

            // Check existing admin
            $adminExists = Admin::query()
                ->where('email', $invitation->email)
                ->exists();

            if ($adminExists) {
                throw ValidationException::withMessages([
                    'invitation' => 'برای این ایمیل قبلاً حساب مدیر ایجاد شده است.',
                ]);
            }

            // Create admin
            $admin = Admin::query()->create([
                'first_name' => $data->firstName,
                'last_name' => $data->lastName,
                'email' => $invitation->email,
                'password' => $data->password,
            ]);

            // Verify invited email
            $admin->forceFill([
                'email_verified_at' => now(),
            ])->save();

            // Complete invitation
            $invitation->forceFill([
                'accepted_at' => now(),
                'accepted_admin_id' => $admin->id,
            ])->save();

            return $admin;
        });
    }
}
```
### چرا `lockForUpdate()` استفاده کردیم؟

فرض کن یک مدیر جدید دو بار روی دکمه پذیرش کلیک کند یا دو درخواست هم‌زمان ارسال شود.

بدون قفل رکورد، ممکن است هر دو درخواست وضعیت دعوت‌نامه را معتبر تشخیص دهند.

با `lockForUpdate()` درخواست دوم منتظر می‌ماند تا تراکنش اول تمام شود. سپس وضعیت جدید دعوت‌نامه را می‌بیند و دیگر نمی‌تواند آن را دوباره بپذیرد.

همچنین Unique بودن `admins.email` آخرین لایه محافظت در برابر ایجاد ایمیل تکراری است.

## ۳. ساخت Form Object
فایل:
`app/Livewire/Forms/Admin/AcceptAdminInvitationForm.php`

```php
<?php

namespace App\Livewire\Forms\Admin;

use App\DTOs\Admin\AdminInvitation\AcceptAdminInvitationData;
use Livewire\Form;

class AcceptAdminInvitationForm extends Form
{
    public string $first_name = '';

    public string $last_name = '';

    public string $password = '';

    public string $password_confirmation = '';

    protected function rules(): array
    {
        return [
            'first_name' => ['required', 'string', 'max:100'],

            'last_name' => ['required', 'string', 'max:100'],

            'password' => [
                'required',
                'string',
                'min:12',
                'confirmed',
            ],
        ];
    }

    public function toDTO(): AcceptAdminInvitationData
    {
        return new AcceptAdminInvitationData(
            firstName: $this->first_name,
            lastName: $this->last_name,
            password: $this->password,
        );
    }
}
```

## ۴. ساخت کامپوننت Livewire

دستور:

```
php artisan make:livewire pages::admin.invitations.accept --mfc
```

مسیر کامپوننت:

```bash
resources/views/pages/admin/invitations/
    ⚡accept.blade.php
```

کد بخش PHP کامپوننت:

```php
<?php

use App\Actions\Admin\AdminInvitations\AcceptAdminInvitationAction;
use App\Livewire\Forms\Admin\AcceptAdminInvitationForm;
use App\Models\AdminInvitation;
use Illuminate\Support\Facades\Hash;
use Livewire\Attributes\Locked;
use Livewire\Component;

new class extends Component
{
    public AcceptAdminInvitationForm $form;

    #[Locked]
    public int $invitationId;

    #[Locked]
    public string $email;

    public function mount(string $token): void
    {
        $invitation = AdminInvitation::query()
            ->where('token_hash', hash('sha256', $token))
            ->whereNull('accepted_at')
            ->whereNull('revoked_at')
            ->where('expires_at', '>', now())
            ->first();

        abort_unless($invitation, 404);

        $this->invitationId = $invitation->id;
        $this->email = $invitation->email;
    }

    public function accept(AcceptAdminInvitationAction $action)
    {
        $this->form->validate();

        $admin = $action->handle(
            invitationId: $this->invitationId,
            data: $this->form->toDTO(),
        );

        return redirect('/admin/login')->with(
            'success',
            'حساب مدیر با موفقیت ایجاد شد. اکنون وارد حساب خود شوید.'
        );
    }
};
```

یک نکته: `#[Locked]` باعث می‌شود کاربر نتواند شناسه دعوت‌نامه و ایمیل را از سمت مرورگر تغییر دهد. ما توکن خام را هم در State عمومی Livewire نگه نمی‌داریم؛ فقط هنگام `mount` از URL دریافت و هش آن را بررسی می‌کنیم.
## ۵. تعریف Route

در `routes/web.php` و خارج از گروه `auth:admin`:
```php
Route::livewire(
    '/admin/invitations/{token}',
    'pages::admin.invitations.accept'
)->name('admin.invitations.accept');
```

این مسیر عمومی است؛ چون فرد دعوت‌شده هنوز وارد حساب مدیر خود نشده است.

## ۶. بخش HTML فرم

داخل همان فایل MFC:
```html
<div class="max-w-lg mx-auto p-6">

    <form wire:submit="accept" class="space-y-5">

        <div>
            <label>ایمیل دعوت‌نامه</label>

            <input
                type="email"
                value="{{ $email }}"
                disabled
            >
        </div>

        <div>
            <label>نام</label>

            <input
                type="text"
                wire:model="form.first_name"
            >

            @error('form.first_name')
                <span>{{ $message }}</span>
            @enderror
        </div>

        <div>
            <label>نام خانوادگی</label>

            <input
                type="text"
                wire:model="form.last_name"
            >

            @error('form.last_name')
                <span>{{ $message }}</span>
            @enderror
        </div>

        <div>
            <label>رمز عبور</label>

            <input
                type="password"
                wire:model="form.password"
            >

            @error('form.password')
                <span>{{ $message }}</span>
            @enderror
        </div>

        <div>
            <label>تکرار رمز عبور</label>

            <input
                type="password"
                wire:model="form.password_confirmation"
            >
        </div>

        @error('invitation')
            <div>{{ $message }}</div>
        @enderror

        <button
            type="submit"
            wire:loading.attr="disabled"
            wire:target="accept"
        >
            ایجاد حساب مدیر
        </button>

    </form>

</div>
```

این فقط اسکلت عملکردی فرم است؛ استایل آن را بعداً با سیستم کامپوننت‌های Tixo هماهنگ می‌کنیم.

## ۷. یک نکته امنیتی مهم درباره سیستم فعلی

در Action ساخت دعوت‌نامه، بررسی وجود دعوت فعال و ساخت دعوت جدید در یک تراکنش انجام می‌شود؛ اما این به‌تنهایی جلوی Race Condition برای دو دعوت هم‌زمان با یک ایمیل را نمی‌گیرد.

بعداً باید برای آن هم راهکار دیتابیسی در نظر بگیریم.

همچنین در سیستم واقعی بهتر است پس از پذیرش دعوت‌نامه، کاربر مستقیماً وارد پنل نشود و ابتدا از صفحه ورود مدیران احراز هویت شود؛ همان چیزی که در این پیاده‌سازی انجام دادیم.