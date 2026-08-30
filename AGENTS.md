# AGENTS

Machine-readable contract for agents working on or integrating with the Loot platform.

Last updated: 2026-08-30

---

## Overview

Loot provides a simple API-backed service that lets local merchants include digital reward codes with purchases. Merchants provide minimal transaction details (merchant id, purchase type, optional price or SKU). Loot generates, stores, and secures the reward codes and manages delivery to customers. This spec is the source of truth for payload shapes and endpoints used by clients and integrators.

Key principles:

- Starts at the purchase. Merchants only provide code/transaction details — no new hardware, no SDK, no app required on the merchant side.
- We are selling loot codes / purchase incentives. We do not manage merchant business logic or inventory beyond code issuance and attribution.
- Keep the contract small and machine-readable. Link more detailed how‑tos to this file.

---

## Actors

| Actor        | Role                                                      |
|--------------|-----------------------------------------------------------|
| Customer     | Scans QR or redeems a code in the Loot app.               |
| Merchant     | Provides purchase metadata and includes codes with sales. |
| Partner      | Game or app that redeems items via the partner API.       |
| Admin        | Platform operator for catalog and campaign operations.    |

Phase 1 focuses on Customers and Merchants.

---

## Merchant participation (tester run)

- Minimal input: merchant id, purchase type (coffee, food, fun, art), optional price or SKU, and a code or QR payload.
- Typical flow: merchant prints or displays a QR sticker or includes a short code on a receipt. No merchant-side infrastructure needed.
- Options: merchants can choose a free option for low/no-value rewards (prepackaged keys, low-value digital goods) or opt into premium rewards that carry higher perceived value ("gold" vs "gems").
- We accept simple transaction attribution (phone number or email) so merchants can link a sale to a claimed reward when desired; privacy-preserving defaults are used.

---

## Reward flow (high level)

1. At purchase the merchant includes a code (QR or short code) with the sale.
2. Customer scans the QR or follows the link and claims the reward in their Loot vault.
3. Loot issues the reward and records attribution. Redemption happens in partner apps or via codes provided at checkout.

Call Loot's API with reward details — type, value, expiry. We handle generation, storage, security, and compliance.

---

## API (Phase 1 - essential)

### POST /v1/rewards/claim

Customer claims a reward via QR or short code.

Request:

```json
{
  "code": "coffee-beans-j7x2k",
  "customer_contact": "phone_or_email_optional",
  "merchant_id": "mrc_local_01",
  "purchase_type": "coffee|food|fun|art",
  "source": "qr|text|receipt"
}
```

Response:

```json
{
  "success": true,
  "reward": {
    "id": "rwd_abc",
    "type": "points|sku|voucher",
    "value": 1
  },
  "message": "Reward claimed."
}
```

Notes:
- Identity for customer flows is optional on Phase 1 — a customer can claim via link and provide email/phone for delivery. When authenticated, the server derives identity from the token.
- The platform issues and stores codes; idempotency keys are used on POSTs to make retries safe.

---

## Payload guidance

QR payloads are compact JSON objects (not URLs). Example payload the client decodes and POSTs to /v1/rewards/claim:

```json
{
  "t": "reward",
  "id": "sku_abc123",
  "m": "mrc_local_01",
  "p": "coffee",
  "sig": "hmac-sha256-hex"
}
```

The `sig` covers the critical fields and is computed by the merchant (or our code generator) with a per-merchant HMAC secret.

---

## Assets and imagery

- Use content-hashed filenames for static assets served via CDN.
- For reward artwork, we recommend using Nerd Fonts / icon sets to create consistent, compact reward imagery (icons for chest, gem, coin, etc.). App icon guidance: a treasure chest or gem works well as a simple brand mark.

---

## Security & auth

- All authenticated endpoints require a Bearer token in `Authorization`.
- Use `x-idempotency-key` on POSTs that may be retried.
- Merchant HMAC secrets are stored and rotated server-side; clients never receive those secrets.

---

## Errors

Standard error shape:

```json
{
  "error": true,
  "code": "INVALID_CODE",
  "message": "Code not recognized",
  "http_status": 400
}
```

---

*This file is intentionally concise. For integration or testing questions, open an issue in this repository with the `[agent-feedback]` tag.*
