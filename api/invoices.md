# فاکتورها

چرخه‌ی کامل یک پرداخت: فاکتور بسازید، خریدار را به صفحه‌اش بفرستید، و وقتی وبهوک رسید تأییدش کنید.

## ساخت فاکتور

```
POST /api/v1/invoices
```

<div dir="rtl">

| فیلد | نوع | اجباری | توضیح |
|---|---|---|---|
| `amount_rial` | عدد | بله | بزرگ‌تر از صفر |
| `order_id` | رشته | خیر | شماره‌ی سفارش خودتان، تا ۱۲۸ کاراکتر. در وبهوک برمی‌گردد |
| `callback_url` | رشته | خیر | وبهوک به اینجا فرستاده می‌شود. HTTPS الزامی است |
| `return_url` | رشته | خیر | خریدار بعد از پرداخت به اینجا برمی‌گردد |
| `description` | رشته | خیر | تا ۱۰۰۰ کاراکتر. روی صفحه‌ی پرداخت دیده می‌شود |
| `metadata` | آبجکت | خیر | هرچه بخواهید؛ دست‌نخورده در وبهوک برمی‌گردد |
| `expiry_minutes` | عدد | خیر | بین ۱ تا ۱۴۴۰. پیش‌فرض از تنظیمات فروشگاه می‌آید |

</div>

```bash
curl -X POST https://api.abangateway.ir/api/v1/invoices \
  -H "Authorization: Bearer ag_test_xxxx…xxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "amount_rial": 5990000,
    "order_id": "ORD-1043",
    "callback_url": "https://shop.example/abangate/hook",
    "return_url": "https://shop.example/order/1043",
    "metadata": {"customer": "u_882"}
  }'
```

پاسخ `201`:

```json
{
  "invoice_id": "inv_9fk2m4qx7t3p8wzy1a6b",
  "status": "pending",
  "amount_rial": 5990000,
  "payable_rial": 5993240,
  "payable_toman": 599324,
  "fee_rial": 40000,
  "order_id": "ORD-1043",
  "card_number": "6037991234567890",
  "card_holder": "سارا رضایی",
  "bank": "ملی",
  "iban": null,
  "payment_url": "https://abangateway.ir/pay/inv_9fk2m4qx7t3p8wzy1a6b",
  "expires_at": "2026-08-17T14:32:11+00:00",
  "paid_at": null,
  "is_test": true
}
```

خریدار را به `payment_url` بفرستید. صفحه‌اش شماره‌ی کارت، مبلغ دقیق و شمارش معکوس را نشان می‌دهد و خودش وضعیت را زنده به‌روز می‌کند.

> [!WARNING]
> اگر مبلغ را خودتان جایی نشان می‌دهید، `payable_rial` را نشان دهید نه `amount_rial`. آن چند ریال اختلاف تنها چیزی است که واریز این خریدار را قابل تشخیص می‌کند.

### فاکتور چندتکه

بالای سقف کارت به کارت، فاکتور خودکار به چند تکه می‌شکند و خریدار روی همان یک صفحه چند واریز پشت سر هم انجام می‌دهد. از دید API فرقی نمی‌کند: یک `invoice_id`، یک وبهوک وقتی همه‌ی تکه‌ها تسویه شدند. [توضیح کامل](../guides/card-to-card-ceiling.md).

## خواندن فاکتور

```
GET /api/v1/invoices/{invoice_id}
```

همان بدنه‌ی بالا با وضعیت روز. برای هماهنگ‌سازی بعد از یک وبهوک ازدست‌رفته خوب است — ولی **به‌جای وبهوک نگذاریدش**؛ نظرخواهی مکرر به محدودیت نرخ می‌خورد.

وضعیت‌ها: `pending` · `paid` · `expired` · `cancelled`

## تأیید پرداخت

```
POST /api/v1/invoices/{invoice_id}/verify
```

این همان جایی است که سفارش را تحویل می‌دهید. **فقط یک بار موفق می‌شود** — یک `UPDATE` شرطی پشتش است، پس دو درخواست همزمان هر دو موفق نمی‌شوند و سفارش دوبار تحویل نمی‌رود.

```json
{
  "verified": true,
  "invoice_id": "inv_9fk2m4qx7t3p8wzy1a6b",
  "order_id": "ORD-1043",
  "amount_rial": 5990000,
  "paid_at": "2026-08-17T13:58:02+00:00"
}
```

اگر قبلاً تأیید شده باشد `409 already_verified` می‌گیرید. این را **موفقیت** حساب کنید نه خطا: یعنی سفارش قبلاً تحویل شده.

اگر هنوز پرداخت نشده `402 not_yet_paid`.

## لغو فاکتور

```
POST /api/v1/invoices/{invoice_id}/cancel
```

فقط روی `pending` کار می‌کند. کارمزد رزروشده آزاد می‌شود و لینک پرداخت از کار می‌افتد.

## شبیه‌سازی پرداخت

```
POST /api/v1/invoices/{invoice_id}/simulate-payment
```

**فقط با توکن تست.** فاکتور را طوری تسویه می‌کند که انگار واریز رسیده — وبهوک هم می‌رود. روی توکن زنده `403 sandbox_only` می‌دهد.

این همان چیزی است که اتصالتان را انتها به انتها تست می‌کند بدون اینکه پولی جابه‌جا شود:

```bash
ID=$(curl -sS -X POST https://api.abangateway.ir/api/v1/invoices \
  -H "Authorization: Bearer $ABAN_TEST_TOKEN" -H "Content-Type: application/json" \
  -d '{"amount_rial":100000,"callback_url":"https://shop.example/hook"}' \
  | python3 -c 'import sys,json;print(json.load(sys.stdin)["invoice_id"])')

curl -X POST "https://api.abangateway.ir/api/v1/invoices/$ID/simulate-payment" \
  -H "Authorization: Bearer $ABAN_TEST_TOKEN"
```
