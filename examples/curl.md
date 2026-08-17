# با curl

کل چرخه از خط فرمان. برای فهمیدن API قبل از نوشتن کد، و برای عیب‌یابی وقتی کد کار نمی‌کند.

```bash
export ABAN_TOKEN=ag_test_xxxx…xxxx
export ABAN_BASE=https://api.abangateway.ir
```

## بساز

```bash
curl -sS -X POST "$ABAN_BASE/api/v1/invoices" \
  -H "Authorization: Bearer $ABAN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"amount_rial": 5990000, "order_id": "ORD-1043"}'
```

## بخوان

```bash
curl -sS "$ABAN_BASE/api/v1/invoices/inv_9fk2m4qx7t3p8wzy1a6b" \
  -H "Authorization: Bearer $ABAN_TOKEN"
```

## تأیید کن

```bash
curl -sS -X POST "$ABAN_BASE/api/v1/invoices/inv_9fk2m4qx7t3p8wzy1a6b/verify" \
  -H "Authorization: Bearer $ABAN_TOKEN"
```

## لغو کن

```bash
curl -sS -X POST "$ABAN_BASE/api/v1/invoices/inv_9fk2m4qx7t3p8wzy1a6b/cancel" \
  -H "Authorization: Bearer $ABAN_TOKEN"
```

## یک چرخه‌ی کامل، بدون پول

با توکن تست. فاکتور می‌سازد، پرداختش را شبیه‌سازی می‌کند و تأییدش می‌کند:

```bash
ID=$(curl -sS -X POST "$ABAN_BASE/api/v1/invoices" \
      -H "Authorization: Bearer $ABAN_TOKEN" -H "Content-Type: application/json" \
      -d '{"amount_rial":100000}' \
    | python3 -c 'import sys,json; print(json.load(sys.stdin)["invoice_id"])')

echo "invoice: $ID"

curl -sS -X POST "$ABAN_BASE/api/v1/invoices/$ID/simulate-payment" \
  -H "Authorization: Bearer $ABAN_TOKEN" >/dev/null

curl -sS -X POST "$ABAN_BASE/api/v1/invoices/$ID/verify" \
  -H "Authorization: Bearer $ABAN_TOKEN"
```

## دیدن کد وضعیت

وقتی چیزی کار نمی‌کند، بدنه تنها نصف جواب است:

```bash
curl -sS -o /tmp/body.json -w '%{http_code}\n' \
  -X POST "$ABAN_BASE/api/v1/invoices" \
  -H "Authorization: Bearer $ABAN_TOKEN" -H "Content-Type: application/json" \
  -d '{"amount_rial": 1}'
cat /tmp/body.json
```

## وضعیت سرویس

بدون توکن، عمومی است:

```bash
curl -sS "$ABAN_BASE/api/v1/status"
```
