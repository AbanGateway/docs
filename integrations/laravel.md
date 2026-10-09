# لاراول

بسته رسمی آبان گیت وی برای لاراول ۱۰، ۱۱ و ۱۲. با یک خط نصب میشود و همه چیز را خودش ثبت میکند: تنظیمات، فساد `AbanGateway`، یک مسیر آماده برای وبهوک، و رویدادهایی که فقط بعد از تایید وضعیت از API فرستاده میشوند.

## نصب

```bash
composer require abangateway/abangateway-php-package
```

در `.env`:

```
ABANGATEWAY_KEY=ag_live_xxxx…xxxx
ABANGATEWAY_WEBHOOK_SECRET=...
```

<div dir="rtl">

| مقدار | از کجا |
|---|---|
| `ABANGATEWAY_KEY` | پنل، تنظیمات، کلیدهای API. کلید کامل فقط لحظه ساخت نشان داده میشود |
| `ABANGATEWAY_WEBHOOK_SECRET` | پنل، تنظیمات، وبهوک، دکمه «ساخت کلید امضا» |

</div>

با کلیدی که با `ag_test_` شروع میشود، فاکتورها آزمایشی اند و پولی جابه جا نمیشود.

## ساخت فاکتور

```php
use AbanGateway\Laravel\Facades\AbanGateway;

$invoice = AbanGateway::invoiceForOrder((string) $order->id, [
    'amount_toman' => $order->total,
    'callback_url' => route('abangateway.webhook'),
    'return_url'   => route('orders.show', $order),
]);

if ($invoice->isPaid()) {
    return redirect()->route('orders.show', $order);
}
return redirect()->away($invoice->paymentUrl);
```

`invoiceForOrder` برای هر سفارش یک فاکتور باز نگه میدارد. اگر خریدار بعد از منقضی شدن فاکتور برگردد، تلاش تازه ای مثل `1043#2` میسازد. اگر مبلغ سفارش عوض شده باشد، فاکتور قبلی را لغو میکند. سفارشی را هم که قبلا پرداخت شده دوباره نمیفروشد.

## تحویل سفارش

مسیر `POST /abangateway/webhook` خودش ثبت شده و بیرون از گروه web است، پس به CSRF گیر نمیکند. این مسیر امضا را چک میکند، وضعیت فاکتور را با کلید شما از API میپرسد و بر اساس جواب API، نه ادعای وبهوک، یکی از این رویدادها را میفرستد:

<div dir="rtl">

| رویداد | یعنی |
|---|---|
| `InvoicePaid` | پرداخت کامل شد؛ سفارش را تحویل دهید |
| `InvoicePartiallyPaid` | بخشی از فاکتور چندتکه رسید؛ هنوز تحویل نه |
| `InvoiceExpired` | مهلت تمام شد |
| `InvoiceCancelled` | فاکتور لغو شد |

</div>

```php
use AbanGateway\Laravel\Events\InvoicePaid;
use Illuminate\Support\Facades\Event;

Event::listen(function (InvoicePaid $event) {
    $order = Order::findOrFail($event->orderId());
    if ($order->paid_at === null) {
        $order->update(['paid_at' => now()]);
        // تحویل سفارش
    }
});
```

**تکرار را تحمل کنید.** اگر شنونده خطا بدهد، مسیر کد ۵۰۰ برمیگرداند و آبان گیت وی خبر را بعدا دوباره میفرستد. پس سفارشی که قبلا پرداخت شده نباید دوباره تحویل شود، مثل نمونه بالا. `orderId()` شماره سفارش خودتان را بدون پسوند تلاش برمیگرداند.

## تنظیمات

```bash
php artisan vendor:publish --tag=abangateway-config
```

در `config/abangateway.php` میشود مسیر وبهوک را عوض کرد، به آن middleware داد، یا اگر وبهوک را خودتان مدیریت میکنید، خاموشش کرد.

## آزمایش بدون پول

```php
AbanGateway::simulatePayment($invoice->id);
```

فقط با کلید آزمایشی کار میکند. وبهوک واقعی و امضاشده میفرستد، پس مسیر و شنونده ها هم واقعا اجرا میشوند.

## نسخه ها

بسته روی لاراول ۱۰، ۱۱ و ۱۲ آزمون شده است. پشتیبانی امنیتی لاراول ۱۰ تمام شده؛ کار میکند، ولی به روزرسانی توصیه میشود.

## بیرون از لاراول

همین بسته در PHP ساده هم کار میکند: [نمونه PHP](../examples/php.md).
