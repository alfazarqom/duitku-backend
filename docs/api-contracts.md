# API Contracts — Duitku POP Backend

**Status:** Draft for review; not a finalized implementation contract  
**Updated:** 2026-10-27  
**Repository:** alfazarqom/duitku-backend

This document separates (1) contracts documented by Duitku, (2) proposed contracts for this repository's Supabase Edge Functions, and (3) decisions that remain open. Proposals below are not claims that Duitku specifies the internal endpoint shape.

## 1. Sources and scope

- Duitku POP API Reference (Indonesian): https://docs.duitku.com/pop/id/
- Duitku API Reference (Indonesian): https://docs.duitku.com/api/id/
- Supabase Edge Functions: https://supabase.com/docs/guides/functions
- Supabase Edge Functions Authorization headers: https://supabase.com/docs/guides/functions/auth-headers
- Supabase Edge Functions Auth: https://supabase.com/docs/guides/functions/auth

Duitku documentation is authoritative for provider-facing Create Invoice requests and callbacks. Supabase documentation is authoritative for Edge Function runtime and platform authentication behavior. Neither source defines this repository's database schema or customer authorization policy; those require explicit project decisions.

## 2. Official Duitku provider contract

### 2.1 Create Invoice request

Duitku documents Create Invoice as HTTP POST with a JSON body.

| Environment | Endpoint |
|---|---|
| Sandbox | https://api-sandbox.duitku.com/api/merchant/createInvoice |
| Production | https://api-prod.duitku.com/api/merchant/createInvoice |

Required headers documented by Duitku:

- Content-Type: application/json
- x-duitku-timestamp: Unix timestamp in milliseconds
- x-duitku-signature: HMAC-SHA256 signature
- x-duitku-merchantcode: merchant code

Documented request signature:

- stringToSign = merchantCode + timestamp
- signature = HMAC_SHA256(stringToSign, apiKey)

The API key is the HMAC key. It must remain server-side and must never be sent to a browser.

Duitku's parameter table marks these Create Invoice body fields as required:

| Field | Documented type/limit | Meaning |
|---|---|---|
| paymentAmount | integer | Payment amount |
| merchantOrderId | string, up to 50 characters | Merchant's order identifier |
| productDetails | string, up to 255 characters | Product/payment description |
| email | string, up to 255 characters | Customer email |
| callbackUrl | string, up to 255 characters | Server callback URL |

The documentation lists optional fields including additionalParam, merchantUserInfo, customerVaName, phoneNumber, itemDetails, and customerDetail. Other documented objects and fields may depend on the feature used. Consult the current provider reference before adopting optional fields or tokenization-related features.

Duitku's response documentation includes reference, paymentUrl, statusCode, and statusMessage. Duitku states that repeated requests using the same order ID can return the same response for five minutes. Concurrent or repeated requests may return errors such as “The transaction is still in progress” or “Amount is different please try again later.” This provider behavior does not replace merchant-side idempotency.

### 2.2 Server-to-server payment callback

Duitku documents its payment callback as:

- HTTP method: POST
- Content type: application/x-www-form-urlencoded
- The callback URL must be publicly reachable.
- The receiver is required to return HTTP 200 OK. Duitku documents retries if it does not receive HTTP 200, up to five attempts, followed by a callback-failure notification email. A resend facility is available in the Duitku dashboard.

Documented callback signature:

- stringToSign = merchantCode + amount + merchantOrderId
- signature = HMAC_SHA256(stringToSign, apiKey)

Documented callback fields include:

| Field | Meaning / handling note |
|---|---|
| merchantCode | Merchant/project code; validate against configured merchant code |
| amount | Transaction amount; compare with the expected persisted amount |
| merchantOrderId | Merchant order identifier |
| resultCode | Provider result code; see ambiguity below |
| reference | Duitku reference; persist for reconciliation/tracking |
| signature | HMAC signature to verify before accepting the notification |
| productDetail, additionalParam, paymentCode, merchantUserId | Other documented callback fields |
| bankRespCode, bankRespMsg, transactionState, transactionStateStatus, merchantCustomerId, tokenId, expiryDate, customerName | Additional fields documented for relevant payment flows; do not assume every callback includes them |

Do not trust a callback merely because it reached a public URL. Verify its signature and match merchant code, merchant order ID, and amount to trusted transaction data before applying a state change. These are security requirements proposed for this integration based on the provider's signed callback contract.

#### Important ambiguity: result codes

The current POP reference's server callback parameter table says 00 = Success and 01 = Failed. Elsewhere on the same POP reference, the JavaScript popup callback describes 01 as pending and 02 as closing the popup without completing payment. The browser JavaScript callback and return/redirect parameters are explicitly not authoritative for updating server-side payment status.

These are different flows and must not be conflated. This document does not infer that browser 01 or 02 meanings apply to the server-to-server callback. Confirm any unclear server callback result-code mapping against the current provider specification or Duitku support before finalizing it.

### 2.3 Browser callback and return redirect

Duitku documents the JavaScript callback response with resultCode, merchantOrderId, and reference. The return redirect can include merchantOrderId, resultCode, and reference.

Duitku explicitly warns that neither the JavaScript callback nor redirect parameters should update merchant payment status: redirect parameters can be changed by the customer. They may be used to display a provisional message and direct the customer to a server-side status check.

### 2.4 Provider error responses

The POP reference documents HTTP errors including 400 Bad Request, 401 Unauthorized, 404 Not Found, 409 for a transaction still in progress, and 500 Internal Server Error. It also documents business-specific errors, including amount mismatch during a repeated invoice request. The internal API should not expose raw provider responses or credentials to clients.

## 3. Proposed internal Supabase Edge Function contracts

This section is a design proposal, not a provider-defined contract. Endpoint names are intended repository names. Exact request and response schemas remain subject to review and approval.

### 3.1 duitku-create-invoice

**Purpose:** Validate a checkout request against trusted server-side order data, create a Duitku invoice, and record the provider reference/payment URL only after validating the provider response.

**Proposed request:**

- Method: POST
- Content type: application/json
- Body: a merchant-side order identifier; do not accept a client-calculated total as the source of truth.
- Apply caller authorization/ownership checks according to the approved guest/account policy.

**Proposed behavior:**

1. Validate request shape and caller authorization.
2. Load the order and payable amount from trusted persisted data.
3. Confirm the order is eligible for payment and has not already reached a terminal/paid state.
4. Build the Duitku request on the server using configured merchant credentials and the official signature formula.
5. Validate the provider HTTP status and response shape.
6. Persist the Duitku reference and payment URL using an approved idempotent transaction strategy.
7. Return only the minimum payment information the authorized caller needs.

**Illustrative response envelope (proposal only):**

    {
      "data": {
        "merchantOrderId": "<merchant-order-id>",
        "reference": "<duitku-reference>",
        "paymentUrl": "<provider-payment-url>"
      }
    }

This is an example internal response shape, not a Duitku response schema. The error envelope, whether provider statusCode/statusMessage should be exposed, and whether an existing invoice is returned or rejected must be decided before implementation.

### 3.2 duitku-callback

**Purpose:** Receive and validate Duitku's server-to-server payment notification.

**Proposed request:**

- Method: POST
- Expected provider content type: application/x-www-form-urlencoded
- Parse fields as form data; do not assume JSON.
- Verify the signature using the configured API key and official concatenation order.
- Validate configured merchantCode, merchantOrderId, amount, and reference against trusted transaction data before a payment update.

**Proposed behavior:**

1. Reject malformed or incomplete payloads from state-changing processing.
2. Verify the signature using a constant-time comparison where supported.
3. Resolve the transaction from a trusted merchant order identifier.
4. Check merchant code, expected amount, and any known Duitku reference.
5. Apply the approved server-callback result-code mapping.
6. Process duplicate callbacks idempotently; repeated notifications must not duplicate stock/order side effects.
7. Record enough event metadata for audit and reconciliation, without logging API keys, full signatures, or unnecessary sensitive payment data.
8. Return HTTP 200 only according to an approved acknowledgement policy. Duitku documents the HTTP 200 requirement and retries, but handling for invalid signatures, unknown orders, and temporary database failures must be designed deliberately.

No final callback response body is specified here. The official reference requires HTTP 200 acknowledgement, but the repository's exact response body and per-error HTTP status behavior remain open.

### 3.3 get-payment-status

**Purpose:** Return the merchant's current persisted payment/order status to a customer-facing client after checkout or redirect.

**Proposed request:**

- Method: GET or POST — not decided.
- Input: a merchant order identifier or opaque lookup token — not decided.
- Do not accept a browser-provided resultCode as the source of payment truth.

**Proposed behavior:**

- Read the merchant's persisted state; do not infer payment success from a redirect or JavaScript callback.
- Enforce the approved authorization/ownership policy, including guest-checkout access.
- Return only fields needed for the UI.
- If provider inquiry/reconciliation is added later, define it as a separate trusted server-side operation and follow the applicable official Duitku contract.

The response schema and access mechanism are not finalized. A predictable identifier alone is not authorization.

## 4. Cross-endpoint error categories (proposed)

These are internal categories for design and testing, not Duitku-defined error codes:

- Invalid request or unsupported content type
- Unauthorized or forbidden caller
- Order not found or not accessible
- Order not eligible for payment
- Stored amount mismatch
- Duitku authentication/signature error
- Duitku provider rejection or temporary unavailability
- Invalid or mismatched callback signature/merchant/order/amount/reference
- Duplicate callback or already-processed event
- Persistence/transaction failure
- Payment state unavailable

Before implementation, decide the stable HTTP status and response body for each category. Client responses should be useful without disclosing credentials, raw provider internals, stack traces, or sensitive order data.

## 5. Decisions required before implementation

1. **Authentication:** Which functions use Supabase JWT verification, which are publicly reachable, and how are callers authorized inside each handler? A provider callback generally cannot supply a customer Supabase JWT; disabling gateway JWT verification does not remove the need for callback signature validation and transaction matching.
2. **Order and amount source:** Confirm the active schema and authoritative source for order total, currency, payable state, and order ownership. Do not finalize database queries or RPC contracts until the actual schema is audited.
3. **Guest checkout:** Decide how a guest is authorized to create an invoice and retrieve status without exposing another customer's order.
4. **Result-code mapping:** Confirm server-to-server callback result-code meanings with the current Duitku reference/support; do not reuse JavaScript popup meanings.
5. **Callback acknowledgement:** Define response status/body for valid callbacks, duplicates, invalid signatures, unknown orders, amount mismatches, and temporary database failures.
6. **Idempotency and concurrency:** Define uniqueness constraints/transaction handling for invoice creation and callbacks, including repeated provider requests and duplicate/out-of-order notifications.
7. **Payment state machine:** Approve allowed transitions and handling of conflicting, delayed, or repeated notifications.
8. **Provider response validation:** Define required fields and safe behavior for incomplete or unexpected Create Invoice responses.
9. **Internal response schema:** Finalize success/error envelope and which order/payment fields may be returned to clients.
10. **Environment and secrets:** Confirm sandbox/production selection and secret names/availability separately. Never put the Duitku API key in frontend code or commit it to this repository.
11. **Rate limiting and observability:** Decide controls for public endpoints, safe structured logging, retention, and alerting.

## 6. Minimum contract tests to agree on

- Create Invoice: valid order, invalid body, unauthorized caller, missing order, wrong amount source, ineligible/already-paid order, provider 4xx/5xx, malformed provider response, repeated request, concurrent request.
- Callback: valid signature/success result, documented failure result, invalid signature, wrong merchant code, unknown order, amount mismatch, reference mismatch, duplicate callback, repeated/out-of-order callback, unexpected content type, persistence failure, and correct acknowledgement behavior.
- Status endpoint: authorized customer, unauthorized customer, guest lookup, nonexistent identifier, paid/pending/failed/expired state, and forged browser redirect parameters.
- Security: ensure logs and responses do not leak API keys, signatures, or unrelated customer/order information.

## 7. Change-control boundary

This file documents proposed API contracts only. It does not implement Edge Functions, select final authentication configuration, alter database tables/RPCs/grants, create or change secrets, deploy functions, or perform a payment. Each implementation change requires explicit approval under the repository's change-control workflow.
