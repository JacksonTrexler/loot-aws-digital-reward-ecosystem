# 🏴‍☠️ Loot Platform Contract

Machine-readable specification for integrators building on Loot's small-business reward ecosystem.  
**Last updated:** 2026-08-30 | Source of truth — keep synchronized with platform changes.

---

## Platform Overview

**Loot** is a unified rewards network that helps small merchants build customer loyalty and game developers distribute meaningful rewards. Customers earn merchant-specific items (Coffee Beans from a café, Craft Tokens from a bookstore) by scanning QR codes or entering three-word codes at purchase. These items are immediately usable in partner games and apps.

### The Flow

```
MERCHANT                    CUSTOMER                      DEVELOPER
  │                            │                              │
  ├─ Display QR/Code ──────────┤                              │
  │                            ├─ Scan in Loot app ──────────┤
  │                            │                              │
  │<─ Verification ────────────┤<─ Confirm reward ────────────┤
  │                            │                              │
  │<─ Track redemption ────────┤─ Item added to inventory ───┤
  │                            │                              │
  │                            ├─ Redeem in partner games ───┤
```

---

## System Actors

| Actor              | Role                                                    | Example              |
|--------------------|-------------------------------------------------------|----------------------|
| **Customer**       | Scans QR/codes, collects items, redeems in games      | Coffee shop customer |
| **Merchant**       | Displays codes, receives loyalty metrics, funds rewards | Local café, bookstore|
| **Developer**      | Integrates item data, creates redemption UX           | Game studio partner  |
| **Admin**          | Manages catalog, campaigns, merchant onboarding       | Internal operations  |

**Phase 1 (Active):** Customer and Merchant flows.  
**Phase 2 (Planned):** Developer API for item integration.

---

## Reward Mechanism

### How Customers Earn

1. **QR Scan** – Point device camera at merchant display
2. **Three-Word Code** – Type code printed on receipt or sent via text
3. **Automatic (Linked)** – Phone number tied to merchant payment method
4. **Confirmation** – Immediate in-app notification + optional SMS

### How Merchants Participate

- **Zero setup cost** — fund only rewards you issue
- **No app required** — printed/digital QR codes work with any smartphone
- **Tracked metrics** — simple dashboard showing redemption trends
- **Custom items** — Coffee Beans, Craft Tokens, etc. tailored to your business

### How Developers Integrate

Partners access a clean API to:
- Query customer item inventory (with permission)
- Validate redemption codes in-game
- Track usage analytics

---

## Data Model

### Core Entities

#### Item

```json
{
  "id": "item_abc123",
  "sku": "coffee_beans",
  "name": "Coffee Beans",
  "description": "Earned at The Daily Roast. Redeem in Pixel Café.",
  "merchant_id": "merch_xyz789",
  "artwork_url": "https://cdn.loot.local/items/coffee_beans.webp",
  "max_quantity": 100,
  "created_at": "2026-01-15T10:30:00Z"
}
```

#### Redemption

```json
{
  "id": "redemption_def456",
  "customer_id": "cust_abc789",
  "item_id": "item_abc123",
  "quantity": 5,
  "redeemed_at": "2026-08-30T14:22:00Z",
  "game_id": "pixel_cafe",
  "merchant_id": "merch_xyz789"
}
```

#### Campaign (Merchant-to-Customer)

```json
{
  "id": "campaign_001",
  "merchant_id": "merch_xyz789",
  "name": "Back-to-School Boost",
  "item_id": "item_abc123",
  "start_date": "2026-09-01",
  "end_date": "2026-09-30",
  "budget": 500,
  "status": "active"
}
```

---

## API Endpoints (Phase 1)

### Customer Endpoints

#### `POST /v1/rewards/claim`
Customer claims a reward via QR or three-word code.

**Request**
```json
{
  "code": "coffee-beans-j7x2k",
  "customer_id": "cust_abc789",
  "merchant_id": "merch_xyz789",
  "source": "qr|text|auto"
}
```

**Response**
```json
{
  "success": true,
  "reward": {
    "item_id": "item_abc123",
    "quantity": 1,
    "name": "Coffee Beans"
  },
  "message": "You earned Coffee Beans! Redeem in Pixel Café."
}
```

#### `GET /v1/customer/inventory`
Fetch all items owned by the customer.

**Query Parameters**
- `customer_id` (required)
- `merchant_id` (optional – filter by business)
- `game_id` (optional – filter by integration)

**Response**
```json
{
  "inventory": [
    {
      "item_id": "item_abc123",
      "name": "Coffee Beans",
      "quantity": 15,
      "merchant_name": "The Daily Roast"
    }
  ]
}
```

#### `POST /v1/rewards/redeem`
Customer redeems items in a partner game.

**Request**
```json
{
  "customer_id": "cust_abc789",
  "item_id": "item_abc123",
  "quantity": 5,
  "game_id": "pixel_cafe",
  "redemption_code": "code_xyz"
}
```

**Response**
```json
{
  "success": true,
  "message": "5 Coffee Beans used. Your café is now level 2!",
  "remaining": 10
}
```

### Merchant Endpoints

#### `GET /v1/merchant/campaigns`
List campaigns and their performance.

**Response**
```json
{
  "campaigns": [
    {
      "id": "campaign_001",
      "name": "Back-to-School",
      "issued": 245,
      "redeemed": 189,
      "redemption_rate": 0.77,
      "budget_remaining": 255
    }
  ]
}
```

#### `POST /v1/merchant/verify`
Verify a customer's reward claim (merchant dashboard).

**Request**
```json
{
  "code": "coffee-beans-j7x2k",
  "merchant_id": "merch_xyz789"
}
```

**Response**
```json
{
  "valid": true,
  "customer_name": "Jamie S.",
  "reward_name": "Coffee Beans"
}
```

---

## Enum Reference

### Reward Source

```
"qr"        – Scanned QR code
"text"      – Three-word code typed in
"auto"      – Automatic (linked payment)
"admin"     – Issued by internal team
```

### Campaign Status

```
"draft"     – Not yet live
"active"    – Currently running
"paused"    – Temporarily stopped
"ended"     – Completed or expired
```

### Integration Status

```
"pending"   – Developer awaiting approval
"active"    – Live integration
"suspended" – Temporary hold
"archived"  – No longer used
```

---

## Authentication

### Scope Requirements

| Endpoint              | Required Scope   |
|-----------------------|------------------|
| `/v1/rewards/claim`   | `customer`       |
| `/v1/customer/*`      | `customer`       |
| `/v1/merchant/*`      | `merchant`       |
| `/v1/developer/*`     | `partner`        |
| `/v1/admin/*`         | `admin`          |

### Bearer Token

All requests require a Bearer token in the `Authorization` header:

```
Authorization: Bearer <token>
```

Tokens are scoped to a single actor type and issued via the platform's account creation flow.

---

## Error Responses

All errors follow this format:

```json
{
  "error": true,
  "code": "REWARD_EXPIRED",
  "message": "This code was valid until 2026-08-28. Try another!",
  "http_status": 410
}
```

### Common Codes

| Code                  | HTTP | Meaning                              |
|-----------------------|------|--------------------------------------|
| `INVALID_CODE`        | 400  | Code format unrecognized             |
| `REWARD_EXPIRED`      | 410  | Code no longer valid                 |
| `LIMIT_REACHED`       | 429  | Customer already claimed max items   |
| `MERCHANT_INACTIVE`   | 403  | Merchant suspended or not onboarded  |
| `UNAUTHORIZED`        | 401  | Invalid or missing token             |
| `NOT_FOUND`           | 404  | Item, customer, or merchant missing  |

---

## Versioning

- **Current Version:** v1
- **Breaking changes** introduce a new version path (`/v2/*`)
- **Non-breaking changes** (new optional fields, new endpoints) stay on current version
- **Deprecation window:** 6 months before removal

---

## Integration Checklist

- [ ] Auth token provisioned for your scope
- [ ] Error handling for expired codes and limit reached
- [ ] Reward confirmation UI (in-app notification or SMS)
- [ ] Inventory display for customer
- [ ] Redemption flow wired to game logic
- [ ] Test with merchant sandbox account
- [ ] Monitor error rates and latency

---

## Support & Feedback

Questions? Unclear fields? Missing an integration point?  
**Open an issue on this repository** with the `[agent-feedback]` tag.

---

*Less is more. This spec is machine-readable and human-brief. Avoid lengthy docs — link to this file instead.*
