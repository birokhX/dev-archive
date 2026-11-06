

چون در پروژه‌های واقعی ممکن است ده‌ها نوع فایل داشته باشی:

- تصویر پروفایل ادمین، کاربر و پشتیبان
    
- تصویر دسته‌بندی‌ها
    
- فایل پیوست تیکت‌ها (PDF، تصویر و اسناد)
    
- فایل‌های مستندات
    
- تصاویر گالری
    
- فایل‌های مربوط به محصولات یا خدمات
    

اگر برای هرکدام منطق اعتبارسنجی، ذخیره‌سازی، تغییر نام، حذف و مدیریت مسیر را جداگانه بنویسی، به‌مرور کدها تکراری و نگهداری پروژه سخت می‌شود.

## ۱. معماری پیشنهادی

من ساختار را به سه لایه تقسیم می‌کنم:

![[Screenshot 2026-11-06 004856.png]]

اصل معماری این است:

کامپوننت Livewire نباید مسئول پیاده‌سازی جزئیات ذخیره‌سازی فایل باشد.

کامپوننت فقط فایل را دریافت می‌کند و عملیات را به لایه مدیریت فایل واگذار می‌کند.

## ۲. ساختار پوشه‌ها در Laravel

برای Tixo این ساختار را پیشنهاد می‌کنم:

```Plain text
app/
├── Actions/
│   └── Admin/
│       └── Profile/
│           └── UpdateAdminAvatarAction.php
│
├── Services/
│   └── Files/
│       └── FileStorageService.php
│
├── Rules/
│   └── Files/
│       ├── ProfileImageRules.php
│       ├── TicketAttachmentRules.php
│       └── DocumentRules.php
│
├── Livewire/
│   └── Forms/
│       └── Admin/
│           └── Profile/
│               └── UpdateAvatarForm.php
│
resources/
└── views/
    └── components/
        └── forms/
            └── file-upload.blade.php
```

مسئولیت هر قسمت:

|بخش|مسئولیت|
|---|---|
|Livewire|دریافت فایل و نمایش Preview|
|Form|مدیریت State و اعتبارسنجی فرم|
|Action|اجرای عملیات مربوط به یک قابلیت|
|FileStorageService|ذخیره، حذف و مدیریت فایل|
|Rules|قوانین اختصاصی هر نوع فایل|
|Blade Component|رابط کاربری مشترک آپلود|
یک نکته مهم: قرار نیست برای هر فایل یک Service جدا بسازی. عملیات عمومی فایل باید مشترک باشد؛ اما قوانین و عملیات تجاری هر قابلیت باید مستقل بمانند.
## ۳. سرویس مرکزی مدیریت فایل

مثلاً:
`app/Services/Files/FileStorageService.php`

```php
<?php

namespace App\Services\Files;

use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;
use RuntimeException;

class FileStorageService
{
    public function store(
        UploadedFile $file,
        string $directory,
        string $disk = 'public'
    ): string {

        $path = $file->store($directory, $disk);

        if (! $path) {
            throw new RuntimeException('File upload failed.');
        }

        return $path;
    }

    public function delete(
        ?string $path,
        string $disk = 'public'
    ): void {

        if (! $path) {
            return;
        }

        Storage::disk($disk)->delete($path);
    }

    public function exists(
        ?string $path,
        string $disk = 'public'
    ): bool {

        return $path && Storage::disk($disk)->exists($path);
    }
}
```

این سرویس فقط مسئول عملیات عمومی Storage است.

مثلاً:

- ذخیره فایل
    
- حذف فایل
    
- دریافت مسیر ذخیره‌شده
    

اعتبارسنجی فایل و تصمیم‌گیری درباره اینکه چه کسی اجازه آپلود چه فایلی را دارد، وظیفه این سرویس نیست.

این تفکیک بسیار مهم است.

## ۴. ساختار دیتابیس

برای تصویر پروفایل ادمین، فعلاً نیازی به جدول جداگانه نداری.

در جدول `admins` یک ستون اضافه کن:
```php
$table->string('avatar_path')->nullable();
```

مثلاً فایل‌ها را به این شکل ذخیره کن:
```Plain text
storage/app/public/
└── uploads/
    └── admins/
        └── avatars/
            ├── random-name-1.webp
            └── random-name-2.webp
```

در دیتابیس فقط مسیر فایل ذخیره می‌شود:
```
uploads/admins/avatars/random-name-1.webp
```

برای دسترسی عمومی به فایل‌های Public نیز:
```bash
php artisan storage:link
```

Laravel از طریق Filesystem امکان تغییر Disk از Local به S3 را بدون تغییر منطق اصلی ذخیره‌سازی فراهم می‌کند.

## ۵. پیاده‌سازی آپلود تصویر پروفایل با Livewire 4

در کامپوننت MFC پروفایل:

```php
<?php

use Livewire\Component;
use Livewire\WithFileUploads;

new class extends Component
{
    use WithFileUploads;

    public $avatar;

    public function updateAvatar(): void
    {
        // ...
    }
};
```

Livewire فایل را ابتدا به‌صورت Temporary Upload دریافت می‌کند. بعد از اعتبارسنجی باید آن را در Storage دائمی ذخیره کنی. برای پیش‌نمایش تصویر هم `temporaryUrl()` وجود دارد.

Laravel Livewire

در Blade:
```html
<div>
    <input
        type="file"
        wire:model="avatar"
        accept="image/*"
    >

    @if ($avatar)
        <img
            src="{{ $avatar->temporaryUrl() }}"
            class="size-24 rounded-full object-cover"
        >
    @endif

    @error('avatar')
        <span>{{ $message }}</span>
    @enderror

    <button
        type="button"
        wire:click="updateAvatar"
    >
        ذخیره تصویر
    </button>
</div>
```

## ۶. ساخت Action اختصاصی تصویر پروفایل

مثلاً:
`app/Actions/Admin/Profile/UpdateAdminAvatarAction.php`
```php
<?php

namespace App\Actions\Admin\Profile;

use App\Models\Admin;
use App\Services\Files\FileStorageService;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;
use Throwable;

class UpdateAdminAvatarAction
{
    public function __construct(
        private FileStorageService $files
    ) {}

    public function handle(
        Admin $admin,
        UploadedFile $avatar
    ): void {
        $oldPath = $admin->avatar_path;

        $newPath = $this->files->store(
            $avatar,
            'uploads/admins/avatars'
        );

        try {
            $admin->update([
                'avatar_path' => $newPath,
            ]);
        } catch (Throwable $exception) {
            $this->files->delete($newPath);

            throw $exception;
        }

        // Delete old avatar after successful update.
        $this->files->delete($oldPath);
    }
}
```

چرا اول فایل جدید را ذخیره می‌کنیم و بعد فایل قبلی را حذف می‌کنیم؟

چون اگر ذخیره فایل جدید یا بروزرسانی دیتابیس شکست بخورد، نباید تصویر قبلی ادمین را از دست بدهیم.

## ۷. اعتبارسنجی اختصاصی

برای تصویر پروفایل:
```php
[
    'avatar' => [
        'required',
        'image',
        'mimes:jpg,jpeg,png,webp',
        'max:2048',
    ],
]
```

برای پیوست تیکت:
```php
[
    'attachments.*' => [
        'file',
        'mimes:pdf,jpg,jpeg,png,doc,docx',
        'max:10240',
    ],
]
```

## ۸. آیا باید تمام فایل‌ها در یک جدول ذخیره شوند؟

اینجا یک تصمیم معماری مهم داریم.

دو حالت وجود دارد:

|نوع فایل|ساختار پیشنهادی|
|---|---|
|تصویر پروفایل ادمین|`admins.avatar_path`|
|تصویر پروفایل کاربر|`users.avatar_path`|
|تصویر دسته‌بندی|`ticket_categories.image_path`|
|پیوست تیکت|جدول `ticket_attachments`|
|چند تصویر برای یک محصول|جدول اختصاصی تصاویر محصول|

#### ساختار App\Enums\FileType

```php
<?php

namespace App\Enums;

enum FileType: string
{
    case IMAGE = 'image';
    case VIDEO = 'video';
    case DOCUMENT = 'document';
    case AUDIO = 'audio';

    /**
     * Summary of extensions
     * @return array<int, string>
     */
    public function extensions(): array
    {
        return match ($this) {
            self::IMAGE => ['jpg', 'jpeg', 'png', 'webp', 'gif', 'svg'],
            self::DOCUMENT => ['pdf', 'doc', 'docx', 'xls', 'xlsx', 'ppt', 'pptx', 'txt'],
            self::VIDEO => ['mp4', 'mkv', 'avi', 'mov', 'webm'],
            self::AUDIO => ['mp3', 'wav', 'ogg', 'aac'],
        };
    }
}
```

#### یک قانون اعتبارسنجی برای همه تصاویر پImageRule
دستور ساخت
`php artisan make:rule Files/ImageRule`
```php
<?php

namespace App\Rules\Files;

use App\Enums\FileType;
use Closure;
use Illuminate\Contracts\Validation\ValidationRule;
use Illuminate\Translation\PotentiallyTranslatedString;
use Illuminate\Validation\Rule;
use Illuminate\Validation\Rules\File;

class ImageRule implements ValidationRule
{
    /**
     * Run the validation rule.
     *
     * @param  Closure(string, ?string=): PotentiallyTranslatedString  $fail
     */
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        $fileRule = File::image()
            ->type(FileType::IMAGE->extensions())
            ->min(5)
            ->max(2048)
            ->dimensions(
                Rule::dimensions()->maxWidth(3000)->maxHeight(3000)
            );
        $fileRule->validate($attribute, $value, $fail);
    }
}
```


#### استفاده در UpdateAdminAvatarForm.php

```php
<?php

namespace App\Livewire\Forms\Admin\Profile;

use App\Rules\Files\ImageRule;
use Livewire\Form;
  
class UpdateAvatarForm extends Form
{
    /**
     * Summary of rules
     * @return array{avatar: array<ImageRule|string>}
     */
    protected function rules(): array
    {
        return [
            'avatar' => ['required', new ImageRule()]
        ];
    }
}
```