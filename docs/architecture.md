# Architecture — Duitku POP Backend on Supabase

**Status:** proposed architecture for review; not yet an implementation specification.  
**Last reviewed:** 2026-10-27  
**Scope:** repository `alfazarqom/duitku-backend`; Duitku POP payment integration using Supabase Edge Functions.  
**Source of truth:** [Duitku POP documentation](https://docs.duitku.com/pop/id/) for Duitku's contract; [Supabase Edge Functions documentation](https://supabase.com/docs/guides/functions) for runtime/platform behavior.

This document separates documented platform/provider facts from recommended architecture and decisions that remain open. It does not assert that the database schema, functions, secrets, or deployment already exist.

## 1. Documented facts

### 1.1 Supabase Edge Functions

According to the official Supabase documentation:

- Edge Functions are server-side TypeScript functions running in a Deno-compatible Edge Runtime.
- Functions can be used for HTTP endpoints, webhooks, and third-party integrations.
- The platform gateway can perform JWT validation before the function handler runs.
- By default, function JWT verification is enabled. Per-function settings can change this behavior in `supabase/config.toml`, including `verify_jwt = false`.
- Disabling platform JWT verification allows requests to reach the handler without a valid user JWT; the function must then apply appropriate controls for its use case.
- Edge Functions can read environment variables with `Deno.env.get(...)`. Supabase provides project environment values and supports storing additional secrets for server-side use.
- Supabase's secret/service-role keys bypass Row Level Security and must never be exposed in a browser.
- The Supabase CLI supports local development and serving functions; deployment is a separate operation.

References:
- [Edge Functions overview](https://supabase.com/docs/guides/functions)
- [Securing Edge Functions](https://supabase.com/docs/guides/functions/auth)
- [Function configuration](https://supabase.com/docs/guides/functions/function-configuration)
- [Environment variables and secrets](https://supabase.com/docs/guides/functions/secrets)
- [Local development quickstart](https://supabase.com/docs/guides/functions/quickstart)

### 1.2 Duitku POP payment flow

The official Duitku POP documentation describes this high-level flow:

1. The merchant server sends a Create Invoice request to Duitku.
2. Duitku returns a Duitku reference and payment URL/details.
3. The merchant presents or redirects the customer to the Duitku payment page.
4. Duitku JS can notify the frontend, and Duitku separately sends a server-side HTTP Callback to the merchant.
5. The merchant uses HTTP Callback for payment-status notification/processing.

Create Invoice is documented as HTTP POST with JSON. The POP documentation lists separate sandbox and production endpoints and describes request authentication using merchant code, timestamp, and signature headers.

The HTTP Callback is documented as an HTTP POST using `application/x-www-form-urlencoded`. Its signature is based on merchant code, amount, and merchant order ID using HMAC-SHA256 with the API key. The official POP page contains an inconsistency in the documented callback `resultCode` values; do not finalize status mapping from the conflicting entries without clarification.

Duitku's documentation says the frontend JavaScript callback is for customer notification and must not be used to change the merchant's payment status. The server-side HTTP Callback is the documented mechanism for that purpose.

Reference: [Duitku POP API Reference](https://docs.duitku.com/pop/id/), especially Backend/Server → Create Invoice and Callback, Frontend/View → JS Callback and Window Redirection, Errors, and Changelog.

## 2. Proposed component layout

The repository's proposed layout separates provider protocol helpers from individual HTTP entry points:

~~~text
supabase/
├── config.toml
├── migrations/                 # Only if a schema change is separately designed and approved
├── functions/
│   ├── _shared/
│   │   ├── duitku.ts            # Provider HTTP request/response helpers
│   │   ├── duitku-signature.ts  # Signature generation and verification
│   │   └── validation.ts        # Input and shared validation helpers
│   ├── duitku-create-invoice/
│   │   └── index.ts             # Merchant request to create an invoice
│   ├── duitku-callback/
│   │   └── index.ts             # Server-to-server callback receiver
│   └── get-payment-status/
│       └── index.ts             # Status query endpoint; access model not decided yet
└── tests/
    └── README.md
~~~

This is a proposed file organization, not a statement that the files or database objects exist. Shared code should contain reusable protocol behavior only; each function should still enforce its own request validation and authorization policy.

## 3. Proposed logical responsibilities

### 3.1 `duitku-create-invoice`

**Purpose:** validate a merchant-side checkout request and request a Duitku POP invoice.

Proposed sequence:

1. Validate HTTP method, content type, request size, and input shape.
2. Establish that the caller is allowed to create an invoice for the referenced order.
3. Load the authoritative order and payable amount from a trusted data source; do not treat a client-supplied amount as authoritative.
4. Check the order's current state and whether a payment/invoice already exists.
5. Select sandbox or production configuration from server-side environment/configuration, never from an untrusted URL supplied by the caller.
6. Build the documented Create Invoice request and compute its request signature.
7. Call the matching documented Duitku endpoint with a bounded timeout.
8. Validate the provider response before storing the Duitku reference/payment URL or returning a safe response to the caller.
9. Avoid logging API keys, signatures where unnecessary, full customer data, or other secrets.

The exact database reads/writes, allowed order states, duplicate-request policy, and error response contract remain pending until the schema and business rules are agreed.

### 3.2 `duitku-callback`

**Purpose:** receive Duitku's server-to-server payment notification.

Proposed sequence:

1. Accept only the callback method and encoding documented by Duitku POP.
2. Parse the form-urlencoded payload while handling missing, malformed, or duplicate fields safely.
3. Validate required fields and verify the documented HMAC-SHA256 callback signature.
4. Match merchant code, merchant order ID, amount, and known payment/reference information against the stored transaction.
5. Apply only a recognized and confirmed callback status mapping. The conflicting official `resultCode` documentation must be resolved before treating non-success codes as definitive states.
6. Persist the callback event and update payment/order state atomically and idempotently, once the database design supports that guarantee.
7. Return an HTTP response consistent with the agreed callback processing policy. Duitku documents retries when it does not receive HTTP 200, so acknowledgment behavior must be deliberate.
8. Do not trust frontend JS callbacks or return/redirect parameters as proof of payment.

Exact HTTP responses for invalid signatures, unmatched orders, amount mismatch, unknown result codes, or database failure need a documented policy. Do not acknowledge a callback as successfully processed if its required durable processing failed.

### 3.3 `get-payment-status`

**Purpose:** expose a minimal payment-status view to an authorized caller.

The name and intended role are part of the proposed tree only. Its access model, response schema, source of truth, rate limits, and whether it is user-authenticated or uses a scoped order-access token have not yet been decided. It must not expose provider credentials, callback signatures, internal logs, or another customer's payment details.

### 3.4 Shared modules

- `_shared/duitku.ts`: centralize provider endpoint selection, HTTP request construction, timeout handling, and safe response parsing. Do not make an arbitrary caller-provided URL fetchable.
- `_shared/duitku-signature.ts`: implement only the official POP signing formulas verified against the source-of-truth document and tests. Request and callback signatures have different input strings and must not be conflated.
- `_shared/validation.ts`: shared parsing and validation primitives. Business authorization and database consistency checks remain in the relevant endpoint/service layer.

Shared modules should not silently infer missing values, default a transaction to paid, or treat unknown provider response codes as success.

## 4. Proposed request and notification flows

### Create Invoice

~~~text
Caller
  -> Supabase Edge Gateway
  -> duitku-create-invoice
  -> validate caller and request
  -> load trusted order + calculate/verify payable amount
  -> sign documented POP request
  -> Duitku POP Create Invoice API
  <- provider response (reference, paymentUrl, status fields)
  -> persist validated invoice data
  <- safe invoice response to caller
~~~

The exact persistence ordering and recovery behavior for network timeouts must be specified alongside the database model. A timeout does not prove that Duitku failed to create the invoice.

### Payment Callback

~~~text
Duitku POP
  -> duitku-callback (HTTP POST form-urlencoded)
  -> parse and verify signature
  -> match existing merchant order and amount
  -> resolve confirmed callback result code
  -> atomic/idempotent event + payment-state update
  <- HTTP acknowledgment according to callback policy
~~~

A callback should not create an unrelated order based solely on an incoming payload. Unknown or mismatched events should be handled according to a documented security and operational policy.

### Status query

~~~text
Authorized caller
  -> get-payment-status
  -> authorize access to the relevant order/payment
  -> read trusted stored state
  <- minimal status response
~~~

This flow remains conceptual until the authorization model and data schema are selected.

## 5. Security boundaries and configuration

### 5.1 Secrets and environment

Recommended server-side configuration names, consistent with the repository's proposed integration, are:

- `DUITKU_MERCHANT_CODE`
- `DUITKU_API_KEY`
- A deliberate environment selector or separate explicit sandbox/production configuration.

These names are a repository convention, not names mandated by Supabase or Duitku. Do not store real values in Git. Use local-only secret configuration for local development and Supabase's secret mechanism for a hosted project when deployment is separately approved. Keep sandbox and production credentials distinct.

### 5.2 Authentication policy per function

Supabase supports function-specific JWT verification settings. The right policy depends on who calls each endpoint:

| Function | Proposed caller | Decision status |
| --- | --- | --- |
| `duitku-create-invoice` | Storefront or trusted application server | Authentication/guest-checkout authorization policy is still open |
| `duitku-callback` | Duitku server | May need public gateway access because provider callbacks do not carry the merchant's Supabase user JWT; callback signature validation and transaction matching remain mandatory |
| `get-payment-status` | Customer or trusted application | Access-control model is still open |

Do not interpret `verify_jwt = false` as “secure without authentication.” It disables the platform JWT check; the handler must apply the appropriate validation and authorization. A public callback endpoint must validate the provider signature and transaction details. A customer-facing status endpoint must not expose arbitrary order IDs or permit cross-customer reads.

No `verify_jwt` values are being configured by this document; the actual `supabase/config.toml` will be a separate file and decision.

### 5.3 Database and privileged access

No database schema or persistence API is assumed by this document. Before implementation, define the source of truth for orders, payments, callback events, and any idempotency constraints. If privileged credentials are used in an Edge Function, keep them server-side and restrict the operations the function performs. Do not create or modify database tables, RLS policies, grants, or RPCs as part of this architecture document.

### 5.4 Operational protections

Recommended controls to design and test:
- Input size and shape limits; method/content-type checks.
- Bounded outbound requests and safe handling of timeouts.
- Rate limits or abuse controls for public-facing endpoints, sized for expected provider retries.
- Redacted logs with correlation identifiers that do not expose secrets.
- Idempotent processing for callback retries and duplicate client requests.
- Explicit handling for unknown result codes, mismatched amounts, duplicate references, and persistence failures.
- Sandbox tests before any production readiness claim.

These are architecture recommendations, not claims that the provider requires these exact implementation mechanisms.

## 6. Failure and reconciliation principles

The system must distinguish a definite provider response from an ambiguous network outcome.

- **Validation/signature failure:** do not mutate payment state.
- **Order/amount mismatch:** do not mark the order paid; record enough safe diagnostic information for investigation.
- **Duplicate callback:** repeated delivery should not repeat business side effects.
- **Unknown callback result code:** do not map it to success or assume its meaning; use the documented/confirmed policy.
- **Create Invoice timeout or 409:** check existing transaction state and the documented provider semantics before retrying. Do not change the amount or generate a new merchant order ID automatically as a workaround.
- **Database failure during callback:** do not claim durable processing succeeded; define the response/retry policy so the event can be recovered safely.
- **Late payment, expiry, refund, or duplicate payment:** require explicit merchant business rules and reconciliation procedures. Do not infer these policies from the basic Create Invoice contract.

## 7. Decisions still required

The following decisions should be recorded before implementation:

1. **Data model:** existing table names/columns, constraints, unique keys, and the authoritative source for payable amount.
2. **Caller authentication:** guest checkout rules, authenticated customer ownership, and any trusted server-to-server credentials.
3. **Status model:** internal order/payment states and the mapping from confirmed Duitku HTTP Callback result codes. Resolve the official documentation's `resultCode` inconsistency first.
4. **Idempotency:** how invoice creation, duplicate callback events, and concurrent requests are serialized and recovered.
5. **Callback acknowledgment:** HTTP response policy for valid, invalid, unknown, mismatched, duplicate, and temporarily unprocessable events.
6. **Reconciliation:** how to resolve timeouts, delayed callbacks, expired invoices, late payments, refunds, and duplicate payments.
7. **Status endpoint contract:** which caller may query which payment, and the minimal response fields.
8. **Environment strategy:** explicit sandbox/production selection and how credentials will be managed in local and hosted environments.
9. **Testing and release gates:** automated tests, sandbox validation, code review, and separate approval for secrets, migrations, and deployment.

## 8. Out of scope

This document does **not**:
- create or change a database schema, migration, RLS policy, grant, or RPC;
- create, edit, or upload secrets;
- set or change function JWT configuration;
- deploy functions or configure a live Supabase project;
- perform real payments;
- claim that any planned Edge Function is implemented or production-ready.

## 9. Official references

### Supabase

1. [Edge Functions overview](https://supabase.com/docs/guides/functions) — runtime, gateway, integrations, and observability.
2. [Securing Edge Functions](https://supabase.com/docs/guides/functions/auth) — authentication patterns and caller modes.
3. [Function configuration](https://supabase.com/docs/guides/functions/function-configuration) — per-function `verify_jwt` behavior.
4. [Environment variables and secrets](https://supabase.com/docs/guides/functions/secrets) — environment access and secret handling.
5. [Getting started with Edge Functions](https://supabase.com/docs/guides/functions/quickstart) — local development and CLI workflow.

### Duitku

1. [Duitku POP API Reference (Bahasa Indonesia)](https://docs.duitku.com/pop/id/) — Backend/Server → Create Invoice and Callback; Frontend/View → JS Callback and Window Redirection; Errors; Changelog.

When a source is updated, re-check the relevant section before implementing or releasing code. If provider documentation conflicts or leaves behavior ambiguous, record the ambiguity and obtain confirmation rather than inventing a contract.
