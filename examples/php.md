# نمونه‌ی PHP

یک اتصال کامل، بدون هیچ کتابخانه‌ای. سه فایل: ساخت فاکتور، دریافت وبهوک، تحویل سفارش.

> [!NOTE]
> توکن‌ها و کلیدهای این صفحه ساختگی‌اند. مال خودتان را از متغیر محیطی بخوانید، نه از کد.

## ۱. ساخت فاکتور

```php
<?php
const ABAN_BASE  = 'https://api.abangateway.ir';
$token  = getenv('ABAN_TOKEN');

function aban_post(string $path, array $body, string $token): array {
    $ch = curl_init(ABAN_BASE . $path);
    curl_setopt_array($ch, [
        CURLOPT_POST           => true,
        CURLOPT_POSTFIELDS     => json_encode($body, JSON_UNESCAPED_UNICODE),
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT        => 20,
        CURLOPT_HTTPHEADER     => [
            'Authorization: Bearer ' . $token,
            'Content-Type: application/json',
        ],
    ]);
    $raw  = curl_exec($ch);
    $code = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);

    $data = json_decode($raw ?: '{}', true);
    if ($code >= 400) {
        // روی code شرط بگذارید نه روی message.
        throw new RuntimeException($data['error']['code'] ?? 'request_failed');
    }
    return $data;
}

$invoice = aban_post('/api/v1/invoices', [
    'amount_rial'  => 5990000,
    'order_id'     => 'ORD-1043',
    'callback_url' => 'https://shop.example/abangate/hook.php',
    'return_url'   => 'https://shop.example/order/1043',
], $token);

// شناسه را کنار سفارش خودتان ذخیره کنید — وبهوک با همین پیدایش می‌کند.
save_invoice_id('ORD-1043', $invoice['invoice_id']);

header('Location: ' . $invoice['payment_url']);
```

## ۲. دریافت وبهوک

```php
<?php
// hook.php
$secret = getenv('ABAN_WEBHOOK_SECRET');
$raw    = file_get_contents('php://input');
$sent   = $_SERVER['HTTP_X_SIGNATURE'] ?? '';

// روی بایت خام، و با مقایسه‌ی زمان‌ثابت.
if (!hash_equals(hash_hmac('sha256', $raw, $secret), $sent)) {
    http_response_code(401);
    exit;
}

$event = json_decode($raw, true);

if ($event['event'] !== 'invoice.paid') {
    http_response_code(200);   // منقضی و لغوشده هم «رسید» می‌خواهند.
    exit;
}

// تکراری را تحمل کنید: اگر پاسخ ما گم شود دوباره می‌فرستیم.
if (already_delivered($event['invoice_id'])) {
    http_response_code(200);
    exit;
}

// سریع جواب بدهید، کار سنگین را به صف بسپارید.
enqueue_fulfilment($event['invoice_id'], $event['order_id']);
http_response_code(200);
```

## ۳. تحویل سفارش

```php
<?php
function fulfil(string $invoiceId, string $token): void {
    try {
        $result = aban_post("/api/v1/invoices/$invoiceId/verify", [], $token);
    } catch (RuntimeException $problem) {
        if ($problem->getMessage() === 'already_verified') {
            return;   // خطا نیست: قبلاً تحویل شده.
        }
        throw $problem;
    }

    if ($result['verified']) {
        deliver_order($result['order_id']);
    }
}
```

`verify` فقط یک بار موفق می‌شود — پشتش یک `UPDATE` شرطی است، پس دو کارگر همزمان هر دو موفق نمی‌شوند و سفارش دوبار نمی‌رود.

## تست، بدون پول

با توکن `ag_test_` همین کد را اجرا کنید و به‌جای پرداخت واقعی:

```bash
curl -X POST "https://api.abangateway.ir/api/v1/invoices/$ID/simulate-payment" \
  -H "Authorization: Bearer $ABAN_TEST_TOKEN"
```

وبهوک واقعاً فرستاده می‌شود، امضا واقعی است، و هیچ ریالی جابه‌جا نمی‌شود.
