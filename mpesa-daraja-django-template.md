

I just created a template for mpesa intergration using python -django


# M-Pesa Daraja STK Push template (Django)

A copy-and-adapt starting point for initiating an STK Push and receiving its
callback.
 It uses Safaricom Daraja's OAuth and Lipa na M-Pesa Online endpoints.
Check the current Daraja documentation and test with your own sandbox account
before using it for real payments.

## 1. Configuration

Install `requests` in your project and set these environment variables:

# dotenv
MPESA_ENV=sandbox
MPESA_CONSUMER_KEY=...
MPESA_CONSUMER_SECRET=...
MPESA_SHORTCODE=174379
MPESA_PASSKEY=...
MPESA_TRANSACTION_TYPE=CustomerPayBillOnline
MPESA_CALLBACK_URL=https://example.com/api/mpesa/callback/


For a Buy Goods Till, use the transaction type and shortcode required by your
Daraja account and callback URL must be publicly reachable over HTTPS.

## 2. Daraja client (`your_app/mpesa.py`)
 # python-code
import base64
from datetime import datetime
from zoneinfo import ZoneInfo

import requests
from django.conf import settings
from django.core.cache import cache


class MpesaError(Exception):
    """Raised when Daraja rejects a request or cannot be reached."""


def _base_url():
    return (
        "https://sandbox.safaricom.co.ke"
        if settings.MPESA_ENV == "sandbox"
        else "https://api.safaricom.co.ke"
    )


def _access_token():
    token = cache.get("mpesa_access_token")
    if token:
        return token

    try:
        response = requests.get(
            f"{_base_url()}/oauth/v1/generate",
            params={"grant_type": "client_credentials"},
            auth=(settings.MPESA_CONSUMER_KEY, settings.MPESA_CONSUMER_SECRET),
            timeout=10,
        )
        response.raise_for_status()
        data = response.json()
        token = data["access_token"]
        # Leave a margin before the token's actual expiry.
        cache.set("mpesa_access_token", token, timeout=int(data.get("expires_in", 3600)) - 60)
        return token
    except (requests.RequestException, KeyError, ValueError) as exc:
        raise MpesaError("Could not obtain a Daraja access token") from exc


def initiate_stk_push(*, phone: str, amount: int, account_reference: str, description: str):
    """Send an STK Push. `amount` is a positive whole number of KES."""
    if amount <= 0:
        raise ValueError("amount must be a positive whole number of KES")

    timestamp = datetime.now(ZoneInfo("Africa/Nairobi")).strftime("%Y%m%d%H%M%S")
    password = base64.b64encode(
        f"{settings.MPESA_SHORTCODE}{settings.MPESA_PASSKEY}{timestamp}".encode()
    ).decode()
    payload = {
        "BusinessShortCode": settings.MPESA_SHORTCODE,
        "Password": password,
        "Timestamp": timestamp,
        "TransactionType": settings.MPESA_TRANSACTION_TYPE,
        "Amount": amount,
        "PartyA": phone,
        "PartyB": settings.MPESA_SHORTCODE,
        "PhoneNumber": phone,
        "CallBackURL": settings.MPESA_CALLBACK_URL,
        "AccountReference": account_reference[:12],
        "TransactionDesc": description[:13],
    }

    try:
        response = requests.post(
            f"{_base_url()}/mpesa/stkpush/v1/processrequest",
            json=payload,
            headers={"Authorization": f"Bearer {_access_token()}"},
            timeout=15,
        )
        response.raise_for_status()
        data = response.json()
    except (requests.RequestException, ValueError) as exc:
        raise MpesaError("Daraja STK Push request failed") from exc

    if data.get("ResponseCode") != "0":
        raise MpesaError(data.get("ResponseDescription", "Daraja rejected the STK Push"))
    return data
 

Configure these values in Django settings by reading them from the environment.
Use your deployment's secret manager in production. 
The sample expects the
phone number in the format Daraja requires (commonly `2547...` or `2541...`);
validate and normalize user input before calling it.

## 3. Persist a payment and its Daraja IDs

Adapt this model to your existing order/payment schema. Store money as integer
KES (or use integer cents if your business supports fractional KES).

# python-code
from django.db import models


class MpesaPayment(models.Model):
    class Status(models.TextChoices):
        PENDING = "pending", "Pending"
        PAID = "paid", "Paid"
        FAILED = "failed", "Failed"

    amount_kes = models.PositiveIntegerField()
    phone = models.CharField(max_length=15)
    status = models.CharField(
        max_length=10, choices=Status.choices, default=Status.PENDING
    )
    checkout_request_id = models.CharField(max_length=100, unique=True, null=True)
    merchant_request_id = models.CharField(max_length=100, blank=True)
    mpesa_receipt = models.CharField(max_length=30, unique=True, null=True)
    result_code = models.IntegerField(null=True)
    created_at = models.DateTimeField(auto_now_add=True)


Create the pending payment record before sending the prompt. After a successful
response, save both request IDs from Daraja's response:

# python
result = initiate_stk_push(
    phone=payment.phone,
    amount=payment.amount_kes,
    account_reference=str(payment.pk),
    description="Order payment",
)
payment.checkout_request_id = result["CheckoutRequestID"]
payment.merchant_request_id = result["MerchantRequestID"]
payment.save(update_fields=["checkout_request_id", "merchant_request_id"])


If the request fails, report the failure to the caller and leave an auditable
failure state; don't present it as a successful payment. 
In a production flow, also handle the possibility of a callback arriving before the request IDs  have been persisted (for example, by buffering unmatched callbacks for reconciliation).

## 4. Receive callbacks (`your_app/views.py`)

The callback route must be reachable by Safaricom, so it is generally exempt
from Django CSRF middleware. It must not be used as an order ID or payment ID.
This example matches the callback's checkout ID, checks the amount, and makes
repeated callbacks safe to process.

# python-code
import json
import logging

from django.db import transaction
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt
from django.views.decorators.http import require_POST

from .models import MpesaPayment

logger = logging.getLogger(__name__)


@csrf_exempt
@require_POST
def mpesa_callback(request):
    try:
        body = json.loads(request.body)
        callback = body["Body"]["stkCallback"]
        checkout_id = callback["CheckoutRequestID"]
        result_code = int(callback["ResultCode"])
    except (json.JSONDecodeError, KeyError, TypeError, ValueError):
        return JsonResponse({"error": "Invalid callback"}, status=400)

    with transaction.atomic():
        try:
            payment = MpesaPayment.objects.select_for_update().get(
                checkout_request_id=checkout_id
            )
        except MpesaPayment.DoesNotExist:
            # Log this event and reconcile it; do not create/fulfil a payment
            # based solely on an unknown callback.
            logger.warning("M-Pesa callback for unknown checkout request %s", checkout_id)
            return JsonResponse({"error": "Unknown checkout request"}, status=404)

        if payment.status != MpesaPayment.Status.PENDING:
            return JsonResponse({"Result": "Already processed"})

        payment.result_code = result_code
        if result_code == 0:
            try:
                items = callback["CallbackMetadata"]["Item"]
                metadata = {item["Name"]: item.get("Value") for item in items}
                receipt = metadata["MpesaReceiptNumber"]
                amount = int(metadata["Amount"])
                if amount != payment.amount_kes:
                    logger.warning("M-Pesa callback amount mismatch for payment %s", payment.pk)
                    return JsonResponse({"error": "Amount mismatch"}, status=400)
            except (KeyError, TypeError, ValueError):
                return JsonResponse({"error": "Invalid success metadata"}, status=400)

            payment.mpesa_receipt = receipt
            payment.status = MpesaPayment.Status.PAID
        else:
            payment.status = MpesaPayment.Status.FAILED

        payment.save(update_fields=["result_code", "mpesa_receipt", "status"])

    return JsonResponse({"Result": "Accepted"})
 

Add a URL route:

 # python-code
from django.urls import path
from .views import mpesa_callback

urlpatterns = [
    path("api/mpesa/callback/", mpesa_callback, name="mpesa-callback"),
]


## Before production

- Treat a callback as untrusted input: Daraja callbacks are not a substitute for
  checking payment status with Safaricom. For high-value or fulfillment-critical
  payments, reconcile using Daraja's transaction-status flow before fulfillment.
- A hard-to-guess callback path can reduce random traffic, but it does not
  cryptographically authenticate the sender. Never mark an order paid based on
  a callback alone without correlating it to a pending payment and validating it.
- Make fulfillment idempotent too: repeated callbacks must not deliver twice.
- Keep credentials out of source control and logs; use HTTPS and restrictive
  timeouts. Avoid logging phone numbers or full callback bodies.
- Test successful, cancelled, timed-out, malformed, duplicate, unknown-ID, and
  amount-mismatch callbacks in the sandbox. Configure a durable cache backend
  if you run multiple application processes.
