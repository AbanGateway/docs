# نمونه‌ی Python

با `httpx` و `FastAPI`. همان سه مرحله: بساز، بشنو، تحویل بده.

> [!NOTE]
> توکن‌های این صفحه ساختگی‌اند. مال خودتان را از متغیر محیطی بخوانید.

```bash
pip install httpx fastapi uvicorn
```

## ساخت فاکتور

```python
import os

import httpx

BASE = "https://api.abangateway.ir"
TOKEN = os.environ["ABAN_TOKEN"]


class AbanError(Exception):
    """Carries the code, because that is what you branch on."""

    def __init__(self, code: str, detail: dict | None = None) -> None:
        super().__init__(code)
        self.code = code
        self.detail = detail or {}


def call(method: str, path: str, body: dict | None = None) -> dict:
    response = httpx.request(
        method,
        BASE + path,
        json=body,
        headers={"Authorization": f"Bearer {TOKEN}"},
        timeout=20,
    )
    data = response.json() if response.content else {}
    if response.status_code >= 400:
        error = data.get("error", {})
        raise AbanError(error.get("code", "request_failed"), error.get("details"))
    return data


invoice = call(
    "POST",
    "/api/v1/invoices",
    {
        "amount_rial": 5_990_000,
        "order_id": "ORD-1043",
        "callback_url": "https://shop.example/abangate/hook",
        "return_url": "https://shop.example/order/1043",
    },
)

# Keep the id beside your own order; the webhook finds the order through it.
save_invoice_id("ORD-1043", invoice["invoice_id"])
redirect_to(invoice["payment_url"])
```

## دریافت وبهوک

```python
import hashlib
import hmac
import os

from fastapi import FastAPI, Request, Response

SECRET = os.environ["ABAN_WEBHOOK_SECRET"].encode()
app = FastAPI()


@app.post("/abangate/hook")
async def hook(request: Request) -> Response:
    # The raw bytes, not a re-serialised model: the signature covers what was
    # sent, and json.dumps of a parsed body is not the same string.
    raw = await request.body()
    sent = request.headers.get("x-signature", "")
    mine = hmac.new(SECRET, raw, hashlib.sha256).hexdigest()

    # compare_digest, not ==. A plain comparison returns on the first
    # differing character, and that timing difference leaks the signature.
    if not hmac.compare_digest(mine, sent):
        return Response(status_code=401)

    event = await request.json()
    if event["event"] != "invoice.paid":
        return Response(status_code=200)

    # A delivery can arrive twice if our read of your reply was lost.
    if not already_delivered(event["invoice_id"]):
        enqueue_fulfilment(event["invoice_id"], event["order_id"])

    return Response(status_code=200)
```

## تحویل سفارش

```python
def fulfil(invoice_id: str) -> None:
    try:
        result = call("POST", f"/api/v1/invoices/{invoice_id}/verify")
    except AbanError as problem:
        if problem.code == "already_verified":
            return  # Not a failure: this order went out earlier.
        raise

    if result["verified"]:
        deliver_order(result["order_id"])
```

## تست، بدون پول

```python
invoice = call("POST", "/api/v1/invoices", {"amount_rial": 100_000,
                                            "callback_url": "https://shop.example/abangate/hook"})
call("POST", f"/api/v1/invoices/{invoice['invoice_id']}/simulate-payment")
```

با توکن `ag_test_` وبهوک واقعاً می‌آید و امضایش واقعی است، ولی هیچ پولی جابه‌جا نمی‌شود.
