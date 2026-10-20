# راهنمای Action در Laravel؛ معماری، نام‌گذاری و استفاده در Livewire و REST API

> **محدوده این راهنما:** Laravel 13، PHP مدرن، Livewire 4 و REST API\
> **هدف:** داشتن یک قرارداد روشن برای ساخت Actionهای قابل نگهداری، قابل
> تست و مستقل از رابط کاربری.

------------------------------------------------------------------------

## 1. Action چیست؟

در این راهنما، **Action** یک کلاس کاربردی است که یک عملیات مشخص کسب‌وکار
را اجرا می‌کند؛ برای مثال:

-   ایجاد مخاطب
-   تغییر وضعیت تیکت
-   تخصیص نقش به کاربر
-   لغو سفارش
-   ثبت پرداخت
-   ارسال پیام

Action یک قابلیت داخلی اجباری در Laravel نیست و Laravel به‌صورت پیش‌فرض
الزام نمی‌کند که چنین کلاس‌هایی در `app/Actions` قرار بگیرند یا متد
`handle()` داشته باشند. این یک **الگوی معماری و قرارداد تیمی** است که
می‌توانیم برای یکپارچگی پروژه انتخاب کنیم.

در این راهنما قرارداد پیشنهادی چنین است:

-   هر Action یک عملیات معنادار را اجرا می‌کند.
-   نام کلاس با فعل و موضوع عملیات ساخته می‌شود.
-   متد ورود اصلی Action، `handle()` است.
-   Action به Livewire، Controller یا Request وابسته نمی‌شود.
-   اگر داده ورودی پیچیده یا مشترک است، Action یک DTO دریافت می‌کند.
-   اعتبارسنجی ورودی و مجوز دسترسی در لایه مناسب انجام می‌شود.
-   خروجی Action صریح و تایپ‌شده است.

## 2. Action چه مشکلی را حل می‌کند؟

بدون جداسازی عملیات، ممکن است منطق ایجاد مخاطب داخل Livewire و سپس همان
منطق داخل API Controller تکرار شود. این کار به مرور باعث دو نسخه متفاوت
از یک عملیات می‌شود.

با Action، هر ورودی فقط درخواست را اعتبارسنجی و آماده می‌کند؛ سپس Action
مشترک عملیات را اجرا می‌کند.

``` text
Livewire Component ──> Form Object ── validate() ──┐
                                                   ├──> DTO ──> Action ──> Eloquent
API Controller ──────> FormRequest ── validated() ─┘

API response: Eloquent Model ──> API Resource ──> JSON
```

**اصل معماری:** Livewire و API دو روش ورودی متفاوت‌اند؛ منطق اصلی یک
عملیات مشترک می‌تواند در یک Action باقی بماند.

## 3. ساختار پوشه‌ها

ساختار پیشنهادی برای یک پروژه متوسط یا بزرگ:

``` text
app/
├── Actions/
│   ├── Contact/
│   │   ├── CreateContactAction.php
│   │   ├── UpdateContactAction.php
│   │   └── DeleteContactAction.php
│   ├── Ticket/
│   │   ├── CreateTicketAction.php
│   │   ├── AssignTicketAction.php
│   │   └── CloseTicketAction.php
│   └── User/
│       ├── CreateUserAction.php
│       └── DeactivateUserAction.php
│
├── Data/
│   ├── Contact/
│   │   ├── CreateContactData.php
│   │   └── UpdateContactData.php
│   └── Ticket/
│       └── CreateTicketData.php
│
├── Services/
│   └── Contact/
│       └── NormalizeContactMobileService.php
│
├── Models/
├── Policies/
├── Livewire/
└── Http/
    ├── Controllers/Api/
    ├── Requests/
    └── Resources/
```

این چیدمان اجباری Laravel نیست؛ یک انتخاب معماری است. نام پوشه‌های دامنه
را در کل پروژه یکدست نگه دارید. در این راهنما از نام مفرد، مثل
`Contact`، استفاده می‌کنیم.

### دو سبک پوشه‌بندی رایج

**سبک A --- گروه‌بندی بر اساس دامنه:**

``` text
Actions/Contact/CreateContactAction.php
Actions/Contact/UpdateContactAction.php
Actions/Ticket/CreateTicketAction.php
```

برای پروژه‌های متوسط و بزرگ، این سبک معمولاً یافتن کلاس‌های مربوط به یک
حوزه را آسان می‌کند.

**سبک B --- همه Actionها در یک پوشه:**

``` text
Actions/CreateContactAction.php
Actions/UpdateContactAction.php
Actions/CreateTicketAction.php
```

برای پروژه کوچک قابل قبول است؛ اما با زیادشدن کلاس‌ها پوشه شلوغ می‌شود.

## 4. نام‌گذاری Action

الگوی پیشنهادی:

`Verb + Entity + Action`

یعنی فعل عملیات + موجودیت یا موضوع عملیات + پسوند `Action`.

  عملیات                 نام پیشنهادی
  ---------------------- --------------------------
  ایجاد مخاطب            `CreateContactAction`
  ویرایش مخاطب           `UpdateContactAction`
  حذف مخاطب              `DeleteContactAction`
  بازیابی مخاطب حذف‌شده   `RestoreContactAction`
  جست‌وجوی مخاطبان        `SearchContactsAction`
  بستن تیکت              `CloseTicketAction`
  تخصیص تیکت             `AssignTicketAction`
  تخصیص نقش به کاربر     `AssignRoleToUserAction`
  ثبت پرداخت             `RecordPaymentAction`
  لغو سفارش              `CancelOrderAction`

نام‌هایی مانند `ContactAction` یا `ManageContactAction` معمولاً بیش از حد
کلی‌اند؛ مشخص نیست دقیقاً چه عملیاتی انجام می‌دهند.

### مفرد یا جمع؟

پوشه دامنه را به نام موجودیت مفرد نام‌گذاری کنید:

``` php
namespace App\Actions\Contact;
```

نه:

``` php
namespace App\Actions\Contacts;
```

این قانون Laravel نیست؛ قرارداد پیشنهادی این راهنماست. مهم‌ترین نکته ثبات
در کل پروژه است.

## 5. متد اصلی Action: `handle()`

برای این پروژه، قرارداد ما این است که هر Action متد عمومی `handle()`
داشته باشد.

``` php
<?php

namespace App\Actions\Contact;

use App\Data\Contact\CreateContactData;
use App\Models\Contact;
use App\Models\User;

final class CreateContactAction
{
    public function handle(
        User $user,
        CreateContactData $data,
    ): Contact {
        return $user->contacts()->create([
            'name' => $data->name,
            'mobile' => $data->mobile,
            'email' => $data->email,
            'url' => $data->url,
        ]);
    }
}
```

`handle()` یک قرارداد انتخاب‌شده برای Actionهای ماست، نه الزام عمومی
Laravel. نام‌های `execute()` یا `run()` نیز در پروژه‌های دیگر دیده می‌شوند؛
از چند قرارداد هم‌زمان بدون دلیل استفاده نکنید.

### چرا یک Action برای هر عملیات؟

این کلاس فقط مخاطب ایجاد می‌کند. برای ویرایش و حذف، Actionهای جدا تعریف
می‌کنیم. از یک کلاس بزرگ مانند `ContactAction` با متدهای `create()`،
`update()`، `delete()` و ده‌ها عملیات نامرتبط پرهیز کنید.

یک Action می‌تواند متدهای خصوصی کمکی داشته باشد، اما نقطه ورود عمومی اصلی
آن باید روشن بماند. اگر یک بخش منطق به مسئولیتی مستقل و قابل استفاده
مجدد تبدیل شد، آن را به کلاس تخصصی جدا منتقل کنید.

## 6. آیا هر Action به DTO نیاز دارد؟

خیر. DTO ابزار اختیاری برای تعریف قرارداد داده است، نه شرط اجباری
Action.

DTO معمولاً زمانی ارزشمند است که:

-   ورودی عملیات چند فیلد دارد.
-   داده از چند مسیر مثل Livewire و API می‌آید.
-   می‌خواهید نوع و شکل ورودی واضح باشد.
-   عملیات بخشی از یک جریان کاربردی مهم یا پیچیده است.

برای یک عملیات بسیار ساده، ممکن است ورودی مستقیم تایپ‌شده یا آرایه‌ای مشخص
کافی باشد. از DTO برای هر مقدار کوچک و هر متد ساده، بدون نیاز واقعی،
استفاده نکنید.

نمونه DTO:

``` php
<?php

namespace App\Data\Contact;

final readonly class CreateContactData
{
    public function __construct(
        public string $name,
        public ?string $mobile,
        public ?string $email,
        public ?string $url,
    ) {}

    /**
     * ورودی این متد باید قبلاً اعتبارسنجی شده باشد.
     */
    public static function fromArray(array $data): self
    {
        return new self(
            name: $data['name'],
            mobile: $data['mobile'] ?? null,
            email: $data['email'] ?? null,
            url: $data['url'] ?? null,
        );
    }
}
```

`fromArray()` اعتبارسنجی انجام نمی‌دهد؛ فقط DTO را از آرایه می‌سازد.
اعتبارسنجی باید قبل از آن صورت بگیرد.

## 7. استفاده در Livewire 4

Form Object در Livewire مسئول state و اعتبارسنجی فرم است. بعد از
اعتبارسنجی، داده را به DTO تبدیل و Action را فراخوانی کنید.

``` php
<?php

namespace App\Livewire\Contact;

use App\Actions\Contact\CreateContactAction;
use App\Data\Contact\CreateContactData;
use Livewire\Component;

final class CreateContact extends Component
{
    public ContactForm $form;

    public function save(CreateContactAction $action): void
    {
        $validated = $this->form->validate();

        $data = CreateContactData::fromArray($validated);

        $action->handle(
            user: auth()->user(),
            data: $data,
        );

        $this->form->reset();

        $this->dispatch('contact-created');
    }
}
```

این مثال فرض می‌کند `ContactForm` قواعد اعتبارسنجی و فیلدهای مربوط را
تعریف کرده است و کاربر از guard پیش‌فرض احراز هویت شده است. در پروژه
چند-guard، کاربر را از guard مناسب دریافت کنید.

نکات مهم:

-   از state خام یا ورودی اعتبارسنجی‌نشده به Action داده ندهید.
-   کاربر مالک مخاطب را از منبع احراز هویت‌شده تعیین کنید، نه از یک
    `user_id` قابل دستکاری در فرم.
-   Policy یا مجوز مناسب را بررسی کنید.
-   Action نباید به `Livewire\Component` وابسته شود.

## 8. استفاده در REST API

### 8.1 FormRequest

``` php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

final class StoreContactRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user() !== null;
    }

    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'mobile' => ['nullable', 'string', 'max:30'],
            'email' => ['nullable', 'email', 'max:255'],
            'url' => ['nullable', 'url', 'max:2048'],
        ];
    }
}
```

بررسی وجود کاربر نمونه‌ای ساده است. در برنامه واقعی، مجوز عملیات را با
Policy یا قاعده دسترسی مناسب بررسی کنید.

### 8.2 Controller

``` php
<?php

namespace App\Http\Controllers\Api;

use App\Actions\Contact\CreateContactAction;
use App\Data\Contact\CreateContactData;
use App\Http\Controllers\Controller;
use App\Http\Requests\StoreContactRequest;
use App\Http\Resources\ContactResource;

final class ContactController extends Controller
{
    public function store(
        StoreContactRequest $request,
        CreateContactAction $action,
    ): ContactResource {
        $data = CreateContactData::fromArray(
            $request->validated()
        );

        $contact = $action->handle(
            user: $request->user(),
            data: $data,
        );

        return new ContactResource($contact);
    }
}
```

در API، از `$request->validated()` استفاده می‌کنیم، نه `$request->all()`.
خروجی Action به Resource سپرده می‌شود تا شکل JSON عمومی و فیلدهای
منتشرشده کنترل شوند.

## 9. Action و Service چه تفاوتی دارند؟

این راهنما قرارداد زیر را پیشنهاد می‌کند:

-   **Action:** نقطه ورود یک عملیات کامل کاربردی؛ مثل
    `CreateContactAction`.
-   **Service:** همکار تخصصی برای یک مسئولیت محدود؛ مثل نرمال‌سازی شماره
    تلفن یا ارتباط با سرویس پیامک.
-   **DTO:** قرارداد داده‌ای که بین لایه‌ها منتقل می‌شود.

مثلاً اگر نرمال‌سازی شماره موبایل در ایجاد، ویرایش و ورود گروهی مخاطبان
استفاده می‌شود، می‌تواند Service مستقل داشته باشد. اما برای یک عملیات ساده
ایجاد مخاطب، Action لزوماً به Service نیاز ندارد.

از ایجاد Service صرفاً برای فراخوانی یک خط Eloquent یا برای اینکه هر
Action حتماً یک Service داشته باشد، پرهیز کنید.

## 10. تراکنش‌ها و عملیات چندمرحله‌ای

اگر Action یک رکورد را ایجاد می‌کند، معمولاً نیازی به تراکنش دستی جداگانه
نیست. اگر چند تغییر دیتابیسی باید همگی موفق شوند یا همگی بازگردانده
شوند، از تراکنش استفاده کنید.

``` php
use Illuminate\Support\Facades\DB;

public function handle(
    User $user,
    CreateContactData $data,
): Contact {
    return DB::transaction(function () use ($user, $data) {
        $contact = $user->contacts()->create([
            'name' => $data->name,
            'mobile' => $data->mobile,
            'email' => $data->email,
            'url' => $data->url,
        ]);

        // مثال: ایجاد رکوردهای وابسته در همین تراکنش

        return $contact;
    });
}
```

تراکنش دیتابیس، درخواست HTTP به سرویس بیرونی را به‌صورت خودکار rollback
نمی‌کند. برای کارهایی مانند ارسال ایمیل یا انتشار پیام، از رویدادها، صف‌ها
و الگوهای مناسب برای هماهنگی با سیستم‌های بیرونی استفاده کنید.

## 11. احراز هویت، مجوز و امنیت

وجود Action به معنای مجاز بودن عملیات نیست. این مسئولیت‌ها را جدا نگه
دارید:

1.  **Authentication:** چه کسی درخواست را ارسال کرده است؟
2.  **Authorization:** آیا این کاربر اجازه انجام این عملیات را دارد؟
3.  **Validation:** آیا داده‌های ورودی معتبر هستند؟
4.  **Action:** عملیات کاربردی مجاز را اجرا می‌کند.
5.  **Database constraints:** قواعد ضروری داده، مانند یکتایی، در سطح
    دیتابیس هم تضمین می‌شوند.

برای مخاطب، مالک را از کاربر احراز هویت‌شده بگیرید:

``` php
$user->contacts()->create([...]);
```

نه از یک شناسه مالک که بی‌بررسی از درخواست گرفته شده باشد. همچنین
`fillable` یا `guarded` مدل را درست تنظیم کنید؛ DTO به‌تنهایی از mass
assignment جلوگیری نمی‌کند.

## 12. تست‌پذیری Action

Action را می‌توان مستقل از Livewire و Controller تست کرد. برای مثال، تست
ایجاد مخاطب باید بررسی کند:

-   مخاطب با فیلدهای مورد انتظار ساخته شده است.
-   مخاطب به مالک درست تعلق دارد.
-   داده‌های نامعتبر از مرز اعتبارسنجی عبور نمی‌کنند.
-   عملیات در شرایط خطا رفتار قابل پیش‌بینی دارد.

نمونه ساده تست Feature:

``` php
<?php

namespace Tests\Feature\Actions\Contact;

use App\Actions\Contact\CreateContactAction;
use App\Data\Contact\CreateContactData;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

final class CreateContactActionTest extends TestCase
{
    use RefreshDatabase;

    public function test_it_creates_a_contact_for_the_given_user(): void
    {
        $user = User::factory()->create();

        $data = new CreateContactData(
            name: 'Sara',
            mobile: null,
            email: 'sara@example.com',
            url: null,
        );

        $contact = app(CreateContactAction::class)->handle(
            user: $user,
            data: $data,
        );

        $this->assertDatabaseHas('contacts', [
            'id' => $contact->id,
            'user_id' => $user->id,
            'name' => 'Sara',
            'email' => 'sara@example.com',
        ]);
    }
}
```

این نمونه فرض می‌کند factoryها، رابطه `contacts()` و ستون‌های جدول مطابق
کد تعریف شده‌اند. نام جدول و فیلدها را با پروژه واقعی تطبیق دهید.

## 13. متد `handle()` و Dependency Injection

در این قرارداد، `handle()` متد اصلی Action است. می‌توانید وابستگی‌های
پایدار مثل Gateway یا Service را از طریق constructor تزریق کنید:

``` php
final class SendMessageAction
{
    public function __construct(
        private SmsGateway $gateway,
    ) {}

    public function handle(string $phone, string $message): void
    {
        $this->gateway->send($phone, $message);
    }
}
```

Laravel Service Container می‌تواند کلاس‌هایی با وابستگی‌های قابل‌حل را
خودکار بسازد و تزریق کند. برای Controller و Livewire، تزریق Action
معمولاً روشن‌تر از ساختن دستی نمونه با `new` است.

توجه: `handle()` برای Actionها در این پروژه یک قرارداد انتخابی است.
Laravel از `handle()` در کلاس‌هایی مانند Queue Job نیز استفاده می‌کند، اما
این موضوع به معنای وجود یک نوع Action رسمی و اجباری در هسته Laravel
نیست.

## 14. آیا Artisan دستور `make:action` دارد؟

در نصب استاندارد Laravel، `make:action` یک دستور داخلی عمومی نیست. ممکن
است پکیج یا کد سفارشی پروژه چنین دستوری اضافه کرده باشد. برای دیدن
دستورهای موجود اجرا کنید:

``` bash
php artisan list
```

برای ساخت دستی فایل‌ها می‌توانید پوشه و کلاس را ایجاد کنید. اگر پروژه
تعداد زیادی Action دارد، می‌توانید یک Artisan command یا stub سفارشی
بسازید تا فایل را با namespace و متد `handle()` استاندارد پروژه تولید
کند.

## 15. اشتباهات رایج

-   ساختن یک `UserAction` بزرگ با مسئولیت‌های متعدد.
-   وابسته کردن Action به `Request`، `Livewire\Component` یا View.
-   تکرار منطق اصلی در Controller و Livewire.
-   استفاده از `request()->all()` و ارسال تمام داده‌ها به مدل.
-   فرض کردن اینکه DTO جای Policy یا validation را می‌گیرد.
-   افزودن تراکنش به هر عملیات بدون نیاز.
-   ساخت Serviceهایی که فقط یک فراخوانی ساده را بدون ارزش اضافه عبور
    می‌دهند.
-   تبدیل هر متد خصوصی یا هر خط کد به یک Action جدید.
-   داشتن چند قرارداد نام‌گذاری متد (`run`, `execute`, `handle`) بدون
    دلیل.
-   فرض کردن اینکه Action حتماً باید `void` برگرداند؛ نوع خروجی را بر
    اساس نتیجه واقعی عملیات تعیین کنید.

## 16. قرارداد پیشنهادی پروژه

برای یکدستی پروژه، این استاندارد را انتخاب کنید:

  مورد                  قرارداد
  --------------------- ------------------------------------------
  مسیر                  `app/Actions/{Domain}/`
  نام پوشه دامنه        مفرد؛ مانند `Contact`
  نام کلاس              فعل + موجودیت + `Action`
  متد اصلی              `handle()`
  ورودی پیچیده          DTO تایپ‌شده، در صورت نیاز
  اعتبارسنجی Livewire   Form Object
  اعتبارسنجی API        FormRequest
  مجوز                  Policy یا لایه مجوز مناسب
  دسترسی به دیتابیس     Eloquent در Action یا collaborator مناسب
  تراکنش                برای تغییرات دیتابیسی وابسته
  خروجی API             API Resource
  تست                   تست رفتار عملیات و قواعد مالکیت

## 17. منابع

-   [Laravel 13 --- Service
    Container](https://laravel.com/docs/13.x/container): Dependency
    Injection و ساخت خودکار کلاس‌ها.
-   [Laravel 13 ---
    Validation](https://laravel.com/docs/13.x/validation): اعتبارسنجی و
    FormRequest.
-   [Laravel 13 --- Eloquent
    Resources](https://laravel.com/docs/13.x/eloquent-resources): شکل‌دهی
    پاسخ JSON.
-   [Laravel 13 --- Queues](https://laravel.com/docs/13.x/queues):
    ساختار Job و استفاده از `handle()` در پردازش صف.
-   [Laravel Action and Service
    Guidelines](https://github.com/tegos/laravel-action-and-service-guideline/blob/main/action-and-service-guidelines.md):
    یک نمونه از قرارداد معماری Action/Service؛ راهنما و پیشنهاد جامعه
    است، نه استاندارد رسمی Laravel.

## جمع‌بندی

Action در این معماری، کلاس اجرای یک عملیات مشخص است؛ DTO داده را منتقل
می‌کند؛ Form Object و FormRequest ورودی را اعتبارسنجی می‌کنند؛ Policy مجوز
را بررسی می‌کند؛ مدل با دیتابیس تعامل دارد و API Resource پاسخ عمومی را
می‌سازد.

از `CreateContactAction::handle()` به‌عنوان قرارداد ثابت پروژه استفاده
کنید و فقط وقتی Service یا DTO اضافه کنید که مسئولیت یا قرارداد مشخصی را
روشن‌تر کنند.
