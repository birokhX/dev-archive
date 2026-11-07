من ساختار را طوری طراحی می‌کنم که:

- ظاهر Toast از منطق مدیریت آن جدا باشد.
    
- تنظیمات `success`، `error`، `warning` و `info` یک محل مرکزی داشته باشند.
    
- بتوانی قالب نمایش هر نوع Toast را تغییر بدهی.
    
- مدیریت Timer، توقف زمان هنگام Hover و ادامه زمان بعد از خروج مستقل باشد.
    
- تعداد Toastهای هم‌زمان محدود باشد.
    
- بتوانی بعداً Progress Bar، دکمه Action، آیکون و حالت‌های مختلف نمایش را اضافه کنی.
    
- Livewire فقط نتیجه عملیات را اعلام کند و درگیر جزئیات Toast نشود.

## ۱. معماری نهایی پیشنهادی

به‌جای اینکه همه‌چیز را در یک فایل قرار بدهیم، چهار لایه ایجاد می‌کنیم:![[Screenshot 2026-11-06 081424.png]]

نکته معماری: تنظیمات و قالب نمایش دو چیز متفاوت هستند. تنظیمات مشخص می‌کنند یک Toast از نوع Success چه ویژگی‌هایی دارد؛ Template مشخص می‌کند این ویژگی‌ها چگونه نمایش داده شوند.

## ۲. ساختار پوشه‌ها

ساختار زیر را پیشنهاد می‌کنم:
```Plain text
resources/
├── js/
│   ├── app.js
│   │
│   └── components/
│       └── toast/
│           ├── index.js
│           ├── toast.store.js
│           ├── toast.config.js
│           └── toast.utils.js
│
└── views/
    └── components/
        └── ui/
            └── toast/
                ├── container.blade.php
                └── item.blade.php
```

مسئولیت هر فایل:

|فایل|مسئولیت|
|---|---|
|`toast.config.js`|تنظیمات انواع Toast|
|`toast.store.js`|مدیریت وضعیت، صف و Timer|
|`toast.utils.js`|توابع کمکی مستقل|
|`index.js`|نقطه ورود و ثبت Store|
|`container.blade.php`|محل نمایش تمام Toastها|
|`item.blade.php`|قالب نمایش یک Toast|
چرا این تفکیک؟

چون احتمالاً در آینده می‌خواهی Toast را از نظر ظاهری تغییر بدهی، بدون اینکه مدیریت Timer را دست‌کاری کنی. یا نوع جدیدی از پیام اضافه کنی، بدون اینکه ساختار Store تغییر کند.

## ۳. تنظیمات مرکزی انواع Toast

فایل:`toast.config.js`
```javascript
export const TOAST_CONFIG = {
    maxVisible: 5,

    defaults: {
        duration: 5000,
        position: 'top-left',
        pauseOnHover: true,
        pauseOnFocus: true,
    },

    types: {
        success: {
            title: 'عملیات موفق',
            icon: 'check-circle',
            duration: 5000,
            role: 'status',
            classes: {
                icon: 'text-success',
                border: 'border-success/20',
                progress: 'bg-success',
            },
        },

        error: {
            title: 'خطا',
            icon: 'alert-circle',
            duration: 7000,
            role: 'alert',
            classes: {
                icon: 'text-danger',
                border: 'border-danger/20',
                progress: 'bg-danger',
            },
        },

        warning: {
            title: 'هشدار',
            icon: 'alert-triangle',
            duration: 6000,
            role: 'status',
            classes: {
                icon: 'text-warning',
                border: 'border-warning/20',
                progress: 'bg-warning',
            },
        },

        info: {
            title: 'اطلاع‌رسانی',
            icon: 'info',
            duration: 5000,
            role: 'status',
            classes: {
                icon: 'text-info',
                border: 'border-info/20',
                progress: 'bg-info',
            },
        },
    },
};
```

چند تصمیم مهم:

۱. تنظیمات هر نوع در یک محل قرار می‌گیرد.

مثلاً اگر بخواهی تمام Toastهای خطا ۸ ثانیه نمایش داده شوند، فقط تنظیمات `error` را تغییر می‌دهی.

۲. عنوان پیش‌فرض داریم.

اگر هنگام ارسال Toast عنوان مشخص نکنی، عنوان متناسب با نوع پیام استفاده می‌شود.

۳. تنظیمات عمومی و اختصاصی جدا هستند.

مثلاً `maxVisible` برای تمام Toastها مشترک است، اما `duration` می‌تواند برای هر نوع متفاوت باشد.

۴. کلاس‌های Tailwind را به‌صورت کامل نوشته‌ایم.

کلاس‌ها را به شکل `text-${color}-500` تولید نکن؛ چون Tailwind ممکن است نتواند کلاس‌های پویا را شناسایی کند.

## ۴. طراحی Store و مدیریت Toast

فایل: `toast.store.js`

در این قسمت مهم‌ترین بخش معماری را پیاده می‌کنیم.

یک تصمیم مهم نسبت به کد فعلی تو دارم: Timerها را از آبجکت‌های Reactive مربوط به Alpine جدا می‌کنیم و در یک `Map` خصوصی نگه می‌داریم.

چرا؟ چون Timer یک جزئیات اجرایی است، نه بخشی از وضعیت رابط کاربری. نیازی نیست Alpine آن را رصد کند.

```javascript
import { TOAST_CONFIG } from './toast.config';

export function createToastStore() {
    let nextId = 0;

    const timers = new Map();

    return {
        items: [],

        add({
            title,
            message,
            type = 'info',
            duration,
            action = null,
        }) {
            const config =
                TOAST_CONFIG.types[type] ??
                TOAST_CONFIG.types.info;

            const id = ++nextId;

            const toast = {
                id,
                title: title ?? config.title,
                message,
                type: TOAST_CONFIG.types[type] ? type : 'info',
                config,

                duration: duration ?? config.duration,
                remaining: duration ?? config.duration,

                paused: false,
                startedAt: null,

                action,
            };

            this.items.push(toast);

            this.enforceLimit();

            if (toast.duration > 0 && !action) {
                this.startTimer(toast);
            }

            return id;
        },

        remove(id) {
            this.clearTimer(id);

            this.items = this.items.filter(
                item => item.id !== id
            );
        },

        clear() {
            for (const id of timers.keys()) {
                this.clearTimer(id);
            }

            this.items = [];
        },

        enforceLimit() {
            while (
                this.items.length > TOAST_CONFIG.maxVisible
            ) {
                const oldest = this.items[0];

                this.remove(oldest.id);
            }
        },

        startTimer(toast) {
            if (toast.remaining <= 0 || toast.action) return;

            this.clearTimer(toast.id);

            toast.paused = false;
            toast.startedAt = Date.now();

            const timer = setTimeout(() => {
                this.remove(toast.id);
            }, toast.remaining);

            timers.set(toast.id, timer);
        },

        pauseTimer(toast) {
            if (
                toast.paused ||
                toast.duration <= 0 ||
                toast.action
            ) return;

            this.clearTimer(toast.id);

            const elapsed = Date.now() - toast.startedAt;

            toast.remaining = Math.max(
                toast.remaining - elapsed,
                0
            );

            toast.paused = true;
        },

        resumeTimer(toast) {
            if (!toast.paused) return;

            if (toast.remaining <= 0) {
                this.remove(toast.id);
                return;
            }

            this.startTimer(toast);
        },

        clearTimer(id) {
            const timer = timers.get(id);

            if (timer) {
                clearTimeout(timer);
                timers.delete(id);
            }
        },
    };
}
```

### چه مشکلاتی در این نسخه حل شد؟

|مسئله|راه‌حل|
|---|---|
|تکرار تنظیمات انواع پیام|`TOAST_CONFIG`|
|وابستگی Timer به UI|`Map` خصوصی|
|تعداد نامحدود Toast|`enforceLimit()`|
|باقی ماندن Timer بعد از حذف|`clearTimer()`|
|توقف زمان هنگام Hover|`pauseTimer()`|
|ادامه زمان بعد از Hover|`resumeTimer()`|
|پیام بدون زمان انقضا|`duration: 0`|
|پیام دارای Action|عدم حذف خودکار|
|شناسه تصادفی|شمارنده داخلی|
یک نکته: این پیاده‌سازی فعلاً حداکثر پنج پیام فعال را نگه می‌دارد و پیام قدیمی‌تر را حذف می‌کند. اگر بعداً بخواهی صف انتظار داشته باشی، باید مفهوم `queue` و `visibleItems` را جدا کنیم؛ این دو رفتار یکسان نیستند.

## ۵. ثبت Store و نقطه ورود

فایل `index.js`:

```javascript
import { createToastStore } from './toast.store';

export function registerToast() {
    document.addEventListener('alpine:init', () => {
        Alpine.store('toast', createToastStore());
    });
}
```

در `app.js`:

```javascript
import { registerToast } from './components/toast';

registerToast();
```

در این معماری Store فقط یک بار ثبت می‌شود و تمام صفحات به آن دسترسی دارند. این روش با الگوی Global Store خود Alpine سازگار است.

## ۶. ساخت قالب Toast

اینجا یک تصمیم مهم داریم:

برای هر نوع Toast یک فایل Blade جدا نساز.

چهار نوع Toast از نظر رفتار تقریباً یکسان هستند؛ تفاوت اصلی آن‌ها رنگ، آیکون و عنوان است. پس یک Template مشترک داشته باش که از تنظیمات مرکزی استفاده کند.

فایل `container.blade.php`:

```html
<div
    x-data
    x-on:toast.window="$store.toast.add($event.detail)"
    class="fixed top-5 left-5 z-[100] flex w-full max-w-sm flex-col gap-3"
>
    <template
        x-for="item in $store.toast.items"
        :key="item.id"
    >
        <x-ui.toast.item />
    </template>
</div>
```

فایل `item.blade.php`:

```html
<div
    x-transition:enter="transition ease-out duration-300"
    x-transition:enter-start="opacity-0 -translate-y-3"
    x-transition:enter-end="opacity-100 translate-y-0"
    x-transition:leave="transition ease-in duration-200"
    x-transition:leave-start="opacity-100"
    x-transition:leave-end="opacity-0 translate-x-5"

    @mouseenter="$store.toast.pauseTimer(item)"
    @mouseleave="$store.toast.resumeTimer(item)"
    @focusin="$store.toast.pauseTimer(item)"
    @focusout="$store.toast.resumeTimer(item)"

    :class="item.config.classes.border"

    :role="item.config.role"
    aria-atomic="true"

    class="rounded-xl border bg-surface p-4 shadow-lg"
>
    <div class="flex items-start gap-3">
        <div
            :class="item.config.classes.icon"
            class="shrink-0"
            aria-hidden="true"
        >
            <span x-text="item.config.icon"></span>
        </div>

        <div class="min-w-0 flex-1">
            <p x-text="item.title" class="text-sm font-semibold text-foreground"></p>
            <p
                x-text="item.message"
                class="mt-1 text-sm text-muted"
            ></p>
        </div>

        <button
            type="button"
            @click="$store.toast.remove(item.id)"
            aria-label="بستن پیام"
            class="shrink-0 text-muted hover:text-foreground">✕</button>
    </div>
</div>
```

## ۷. استفاده از Toast در Livewire

حالا استفاده از سیستم بسیار ساده می‌شود.
```php
$this->dispatch(
    'toast',
    type: 'success',
    message: 'دسته‌بندی با موفقیت ایجاد شد.'
);
```

استفاده با عنوان اختصاصی:
```php
$this->dispatch(
    'toast',
    type: 'success',
    title: 'ذخیره اطلاعات',
    message: 'اطلاعات پروفایل با موفقیت به‌روزرسانی شد.'
);
```

استفاده با زمان اختصاصی:
```php
$this->dispatch(
    'toast',
    type: 'warning',
    message: 'این عملیات نیاز به بررسی دارد.',
    duration: 10000
);
```

این همان مزیت اصلی جدا کردن سیستم است: Livewire فقط Event ارسال می‌کند و نیازی ندارد بداند Toast چه رنگی است، چه آیکونی دارد یا چگونه انیمیشن اجرا می‌کند. مستندات Livewire نیز ارتباط Event با Alpine را از همین طریق پشتیبانی می‌کند.

## ۸. چند تصمیم مهم برای توسعه آینده

این قسمت را جدی بگیر؛ چون تفاوت یک Toast ساده با یک Notification System قابل توسعه همین‌جاست.

### قابلیت‌هایی که از ابتدا باید در نظر بگیریم

|قابلیت|تصمیم معماری|
|---|---|
|انواع پیام|تنظیمات مرکزی|
|چند قالب ظاهری|Template Strategy|
|انیمیشن|مستقل از Store|
|Progress Bar|متصل به زمان باقی‌مانده|
|توقف با Hover|Store|
|توقف با Keyboard Focus|Store|
|دکمه Action|پیام بدون Auto-dismiss|
|صف پیام‌ها|Queue مستقل|
|محدودیت تعداد|تنظیمات مرکزی|
|موقعیت نمایش|تنظیمات Container|
|جلوگیری از XSS|نمایش متن با `x-text`|

### آیا در آینده چند Template داشته باشیم؟

فعلاً یک Template کافی است.

اما اگر بعداً بخواهی این مدل‌ها را داشته باشی:

- `default`: پیام استاندارد
    
- `compact`: پیام کوچک
    
- `rich`: پیام دارای تصویر و دکمه عملیات
    

آن موقع یک ویژگی مثل `variant` به تنظیمات اضافه کن و رندر قالب را بر اساس آن انجام بده.

برای چهار نوع Success و Error و Warning و Info، چهار Template جداگانه نساز.

### سه اصلاح تکمیلی که پیشنهاد می‌کنم

۱. تنظیمات عمومی را واقعاً اعمال کن.

در آبجکت Toast این موارد را نیز اضافه کن:

```javascript
pauseOnHover: TOAST_CONFIG.defaults.pauseOnHover,
pauseOnFocus: TOAST_CONFIG.defaults.pauseOnFocus,
```

و برای زمان:
```javascript
duration:
    duration ??
    config.duration ??
    TOAST_CONFIG.defaults.duration,
```

در نتیجه تنظیمات عمومی و اختصاصی هر دو قابل استفاده خواهند بود.

۲. هنگام تغییر Focus داخل Toast، تایمر را بی‌دلیل از سر نگیر.

در `focusout` بررسی کن که Focus واقعاً از محدوده Toast خارج شده باشد:

```html
@focusout="
    if (!$event.currentTarget.contains($event.relatedTarget)) {
        $store.toast.resumeTimer(item)
    }
"
```

۳. پیام دارای Action را خودکار حذف نکن.

اگر Toast دکمه‌ای مثل «مشاهده تیکت» یا «تلاش مجدد» دارد، بهتر است تا زمان اقدام کاربر باقی بماند. این موضوع در راهنماهای دسترس‌پذیری نیز توصیه شده است.