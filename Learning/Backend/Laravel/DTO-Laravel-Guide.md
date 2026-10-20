# DTO در Laravel؛ راهنمای کاربردی برای Livewire 4 و REST API

> نسخه راهنما: Laravel 13، PHP 8.2+ و Livewire 4\
> هدف: شناخت DTO، انتخاب متدهای مناسب و استفاده از یک قرارداد داده مشترک
> در Actionهایی که از Livewire و REST API فراخوانی می‌شوند.

## 1. DTO چیست؟

DTO مخفف **Data Transfer Object** است؛ یعنی شیئی برای انتقال داده بین
بخش‌های مختلف برنامه. DTO قرارداد مشخصی برای داده‌هایی تعریف می‌کند که یک
عملیات دریافت می‌کند یا در صورت نیاز به بخش دیگری تحویل می‌دهد.

برای مثال، عملیات ایجاد مخاطب ممکن است این داده‌ها را نیاز داشته باشد:

-   نام
-   شماره موبایل
-   ایمیل
-   نشانی وب

به‌جای اینکه هر بخش برنامه یک آرایه آزاد با کلیدهای دلخواه به Action
بدهد، می‌توانیم یک کلاس تایپ‌شده تعریف کنیم:

``` php
final readonly class CreateContactData
{
    public function __construct(
        public string $name,
        public ?string $mobile,
        public ?string $email,
        public ?string $url,
    ) {}
}
```

اکنون امضای Action روشن است: برای ایجاد مخاطب باید یک
`CreateContactData` دریافت کند.

### DTO چه چیزی نیست؟

DTO به‌خودی‌خود:

-   جایگزین اعتبارسنجی نیست.
-   جایگزین Form Object در Livewire نیست.
-   جایگزین `FormRequest` در API نیست.
-   جایگزین Eloquent Model نیست.
-   جایگزین API Resource برای شکل‌دهی پاسخ JSON نیست.
-   الزاماً نباید منطق کسب‌وکار یا کوئری دیتابیس داشته باشد.

قاعده پیشنهادی: DTO داده را تعریف و منتقل می‌کند؛ Form Object یا
FormRequest ورودی را اعتبارسنجی می‌کند؛ Action عملیات را اجرا می‌کند؛
Model با داده‌های پایگاه داده کار می‌کند؛ API Resource شکل پاسخ عمومی API
را کنترل می‌کند.

## 2. جریان معماری

``` text
Livewire Form Object ── validate() ──┐
                                     ├──> CreateContactData
API FormRequest ── validated() ──────┘          |
                                                v
                                      CreateContactAction
                                                |
                                                v
                                      Eloquent / Database

API response: Contact Model ──> ContactResource ──> JSON
```

این جداسازی اجازه می‌دهد Livewire و REST API یک Action مشترک را اجرا
کنند، بدون آنکه Action به HTTP یا رابط کاربری وابسته شود.

## 3. ساختار پوشه‌ها

ساختار پیشنهادی برای یک پروژه متوسط یا بزرگ:

``` text
app/
├── Actions/
│   └── Contact/
│       └── CreateContactAction.php
├── Data/
│   └── Contact/
│       └── CreateContactData.php
├── Models/
│   ├── Contact.php
│   └── User.php
├── Livewire/
│   └── Contact/
│       ├── ContactForm.php
│       └── CreateContact.php
└── Http/
    ├── Requests/
    │   └── StoreContactRequest.php
    └── Resources/
        └── ContactResource.php
```

`app/Data` یک قرارداد اجباری Laravel نیست؛ پوشه‌ای متداول برای DTOهای
پروژه است. نام پوشه‌های دامنه را در کل پروژه یکدست نگه دارید؛ برای نمونه
`Contact` مفرد.

## 4. متدهای احتمالی DTO

همه DTOها به تمام این متدها نیاز ندارند. فقط متدهایی را اضافه کنید که
کاربرد واقعی دارند.

  -----------------------------------------------------------------------
  متد                     کاربرد                  پیشنهاد
  ----------------------- ----------------------- -----------------------
  `__construct()`         دریافت داده‌های تایپ‌شده  معمولاً اصلی‌ترین بخش DTO

  `fromArray()`           ساخت DTO از آرایه       مفید وقتی ورودی
                                                  اعتبارسنجی شده به آرایه
                                                  تبدیل می‌شود

  `toArray()`             تبدیل DTO به آرایه      وقتی یک API داخلی یا
                                                  لایه ذخیره‌سازی به آرایه
                                                  نیاز دارد

  `fromModel()`           ساخت DTO از مدل         برای تبدیل مدل به یک
                                                  قرارداد داده داخلی، در
                                                  صورت نیاز

  `fromRequest()`         ساخت DTO مستقیماً از     معمولاً از آن اجتناب
                          HTTP Request            کنید تا DTO به لایه
                                                  HTTP وابسته نشود

  `with...()`             ساخت یک DTO جدید با     فقط برای نیاز مشخص در
                          تغییر یک مقدار          DTOهای immutable

  `toModel()`             تبدیل DTO به مدل و      معمولاً مسئولیت مناسبی
                          ذخیره‌سازی               برای DTO نیست؛ این کار
                                                  را در Action نگه دارید

  `validate()`            اعتبارسنجی داده         معمولاً در DTO قرار
                                                  ندهید؛ از FormRequest
                                                  یا Livewire Form Object
                                                  استفاده کنید
  -----------------------------------------------------------------------

### نکته مهم درباره `readonly`

`readonly` در PHP باعث می‌شود property پس از مقداردهی اولیه دوباره
مقداردهی نشود. این ویژگی برای DTOهایی که بعد از ساخت نباید تغییر کنند
مناسب است. `final` نیز از ارث‌بری کلاس جلوگیری می‌کند. این دو ویژگی
قرارداد معماری Laravel نیستند؛ قابلیت‌های PHP هستند.

## 5. DTO کامل برای ایجاد مخاطب

فایل `app/Data/Contact/CreateContactData.php`:

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
     * ورودی باید قبلاً اعتبارسنجی و محدود شده باشد.
     *
     * @param array{
     *     name: string,
     *     mobile?: string|null,
     *     email?: string|null,
     *     url?: string|null
     * } $data
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

    /**
     * فقط فیلدهای متعلق به این DTO را برمی‌گرداند.
     *
     * @return array{
     *     name: string,
     *     mobile: string|null,
     *     email: string|null,
     *     url: string|null
     * }
     */
    public function toArray(): array
    {
        return [
            'name' => $this->name,
            'mobile' => $this->mobile,
            'email' => $this->email,
            'url' => $this->url,
        ];
    }
}
```

### چرا `fromArray()` داده را اعتبارسنجی نمی‌کند؟

این متد صرفاً تبدیل آرایه به DTO را انجام می‌دهد. اگر `name` وجود نداشته
باشد یا نوع داده اشتباه باشد، ممکن است خطای PHP ایجاد شود. پس ابتدا باید
ورودی را با Form Object یا FormRequest اعتبارسنجی کنید.

### آیا `toArray()` اجباری است؟

خیر. اگر Action مستقیماً از propertyهای DTO استفاده می‌کند، نیازی به
`toArray()` ندارد. آن را تنها وقتی نگه دارید که مصرف‌کننده واقعاً آرایه
لازم داشته باشد.

## 6. Action: اجرای عملیات

فایل `app/Actions/Contact/CreateContactAction.php`:

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

فرض‌های این مثال:

-   مدل `User` رابطه `contacts()` دارد.
-   مدل `Contact` فیلدهای `name`، `mobile`، `email` و `url` را می‌شناسد.
-   کاربر ورودی از guard مورد انتظار برنامه دریافت می‌شود.
-   مجوز ایجاد مخاطب در لایه مناسب بررسی شده است.

استفاده از رابطه مالکیت کمک می‌کند `user_id` از کاربر مورد اعتماد تعیین
شود؛ آن را از ورودی کاربر دریافت نکنید. DTO جایگزین احراز هویت، Policy
یا تنظیم درست mass assignment نیست.

در این مثال تراکنش جداگانه لازم نیست، چون فقط یک رکورد ایجاد می‌شود. اگر
عملیات چند تغییر دیتابیسی وابسته به هم داشته باشد، `DB::transaction()`
را بررسی کنید.

## 7. استفاده در Livewire 4

فرض کنیم یک Form Object به نام `ContactForm` دارید.

فایل نمونه `app/Livewire/Contact/ContactForm.php`:

``` php
<?php

namespace App\Livewire\Contact;

use Livewire\Form;

final class ContactForm extends Form
{
    public string $name = '';

    public ?string $mobile = null;

    public ?string $email = null;

    public ?string $url = null;

    protected function rules(): array
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

سپس کامپوننت:

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

    public function render()
    {
        return view('livewire.contact.create-contact');
    }
}
```

این نمونه فرض می‌کند کاربر احراز هویت شده و guard پیش‌فرض مناسب است. در
برنامه‌ای با guardهای متعدد، کاربر را از guard درست بگیرید و نوع آن را با
امضای Action هماهنگ کنید.

نکته: `validate()` باید فقط فیلدهایی را برگرداند که در قواعد فرم تعریف
شده‌اند. DTO را از داده اعتبارسنجی‌شده بسازید، نه از کل state یا ورودی
خام.

## 8. استفاده در REST API

### 8.1 FormRequest

فایل `app/Http/Requests/StoreContactRequest.php`:

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

این `authorize()` صرفاً وجود کاربر احراز هویت‌شده را بررسی می‌کند؛ در پروژه
واقعی اگر ایجاد مخاطب به Policy یا مجوز خاصی نیاز دارد، آن را نیز بررسی
کنید.

### 8.2 API Resource

فایل `app/Http/Resources/ContactResource.php`:

``` php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

final class ContactResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'mobile' => $this->mobile,
            'email' => $this->email,
            'url' => $this->url,
            'created_at' => $this->created_at,
        ];
    }
}
```

Resource شکل عمومی پاسخ JSON را کنترل می‌کند. DTO ورودی و Resource خروجی
وظیفه‌های متفاوتی دارند.

### 8.3 Controller

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

در این جریان، FormRequest اعتبارسنجی می‌کند، DTO داده تایپ‌شده می‌سازد،
Action عملیات را اجرا می‌کند و Resource خروجی را شکل می‌دهد.

## 9. چه زمانی DTO خروجی یا `fromModel()` نیاز است؟

برای همه پاسخ‌ها لازم نیست DTO خروجی بسازید. در REST API معمولاً
`JsonResource` انتخاب مناسبی برای کنترل پاسخ عمومی است.

`fromModel()` زمانی می‌تواند مفید باشد که بخواهید داده مدل را به یک
قرارداد داخلی مستقل از Eloquent تبدیل کنید؛ مثلاً برای ارسال داده به یک
سیستم خارجی یا ساختن یک پیام برای صف. این تبدیل باید فیلدهای لازم را
به‌صورت صریح انتخاب کند.

از `toModel()` که مسئولیت ذخیره‌سازی را در DTO قرار می‌دهد، معمولاً پرهیز
کنید. ذخیره‌سازی و هماهنگ‌سازی عملیات متعلق به Action یا لایه کاربردی
مناسب است.

## 10. نکات امنیتی و طراحی

1.  **ورودی خام را وارد Action نکنید.** داده را ابتدا اعتبارسنجی کنید.
2.  **از `$request->all()` استفاده نکنید.** برای API از
    `$request->validated()` استفاده کنید.
3.  **مالکیت را از کاربر احراز هویت‌شده بگیرید.** شناسه مالک نباید از
    ورودی قابل‌کنترل کاربر پذیرفته شود، مگر اینکه مورد استفاده با مجوزدهی
    دقیق طراحی شده باشد.
4.  **DTO را جایگزین Policy نکنید.** تایپ‌شدن داده به معنای مجاز بودن
    عملیات نیست.
5.  **DTO را با آرایه‌ای از تمام فیلدهای مدل یکی نگیرید.** فقط داده‌های
    مورد نیاز عملیات را وارد قرارداد کنید.
6.  **متدهای غیرضروری اضافه نکنید.** یک DTO ساده ممکن است فقط
    constructor داشته باشد.
7.  **در عملیات چندمرحله‌ای، تراکنش را در محل مناسب مدیریت کنید.** معمولاً
    Action یا لایه هماهنگ‌کننده مسئول مناسبی است.
8.  **تست بنویسید.** تبدیل DTO، رفتار Action و شکل پاسخ Resource را
    می‌توان جداگانه آزمود.

## 11. چک‌لیست انتخاب متدها

  -----------------------------------------------------------------------
  سؤال                                تصمیم
  ----------------------------------- -----------------------------------
  DTO فقط چند مقدار تایپ‌شده نگه       constructor کافی است
  می‌دارد؟                             

  ورودی از آرایه اعتبارسنجی‌شده می‌آید؟ `fromArray()` می‌تواند مفید باشد

  یک مصرف‌کننده آرایه می‌خواهد؟         در صورت نیاز `toArray()` اضافه کنید

  داده از Eloquent Model به قرارداد   در صورت نیاز `fromModel()`
  داخلی تبدیل می‌شود؟                  

  DTO مستقیم به Request دسترسی پیدا   ترجیحاً از این وابستگی اجتناب کنید
  می‌کند؟                              

  DTO خودش داده را ذخیره می‌کند؟       ذخیره‌سازی را به Action منتقل کنید

  DTO خودش اعتبارسنجی HTTP انجام      از FormRequest یا Form Object
  می‌دهد؟                              استفاده کنید
  -----------------------------------------------------------------------

## 12. منابع رسمی

-   [Laravel 13 ---
    Validation](https://laravel.com/docs/13.x/validation): اعتبارسنجی
    درخواست‌ها و دریافت داده معتبر.
-   [Laravel 13 --- Eloquent API
    Resources](https://laravel.com/docs/13.x/eloquent-resources): تبدیل
    مدل‌ها به پاسخ JSON.
-   [Laravel 13 --- Eloquent
    ORM](https://laravel.com/docs/13.x/eloquent): رابطه‌ها و ذخیره‌سازی
    مدل‌ها.
-   [PHP --- Classes and
    Objects](https://www.php.net/manual/en/language.oop5.php): مفاهیم
    کلاس، property و constructor.
-   [PHP --- Readonly
    classes](https://www.php.net/manual/en/language.oop5.basic.php):
    مرجع کلی قابلیت‌های کلاس در PHP؛ برای جزئیات نسخه، مستندات رسمی همان
    نسخه PHP را بررسی کنید.

## جمع‌بندی

برای معماری Laravel همراه Livewire و REST API:

-   DTO قرارداد داده ورودی عملیات است.
-   Form Object و FormRequest اعتبارسنجی را انجام می‌دهند.
-   Action عملیات کاربردی را اجرا می‌کند.
-   Eloquent Model داده را در پایگاه داده مدیریت می‌کند.
-   API Resource خروجی JSON را کنترل می‌کند.

با `CreateContactData`، `CreateContactAction` و یک Form Object و
FormRequest، یک عملیات مشترک ساخته‌ایم که از دو ورودی متفاوت قابل
فراخوانی است و منطق اصلی آن تکرار نمی‌شود.
