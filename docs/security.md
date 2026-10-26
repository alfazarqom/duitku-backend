# Security — Duitku POP Backend on Supabase

**Status:** security guidance for review; not a claim that controls are already implemented.  
**Last reviewed:** 2026-10-27  
**Repository:** `alfazarqom/duitku-backend`  
**Scope:** Supabase Edge Functions that create Duitku POP invoices, receive Duitku callbacks, and expose payment status.

## 1. Sources and how to read this document

This document separates:

- **Officially documented facts** — behavior stated by Supabase or Duitku documentation.
- **Security recommendations** — controls proposed for this repository based on those facts and common secure implementation practice.
- **Open decisions** — behavior that cannot safely be fixed until the business rules, data model, or provider contract is clarified.

Official sources:
- [Supabase Edge Functions](https://supabase.com/docs/guides/functions)
- [Supabase Securing Edge Functions](https://supabase.com/docs/guides/functions/auth)
- [Supabase Authorization headers](https://supabase.com/docs/guides/functions/auth-headers)
- [Supabase Function Configuration](https://supabase.com/docs/guides/functions/function-configuration)
- [Supabase Environment Variables and Secrets](https://supabase.com/docs/guides/functions/secrets)
- [Duitku POP API Reference (Bahasa Indonesia)](https://docs.duitku.com/pop/id/)

## 2. Officially documented facts

### 2.1 Supabase Edge Function authentication

Supabase documents that Edge Function requests pass through a platform gateway and that `verify_jwt` is enabled by default. When enabled, the platform checks the Authorization credentials before the function handler runs. Supabase documents disabling JWT verification for use cases such as external webhooks that do not send the merchant's Supabase user JWT; the documentation warns that disabling it permits requests to reach the function without a valid JWT.

Supabase also distinguishes:
- `Authorization: Bearer <user-jwt>` for a signed-in Supabase user;
- `apikey` for a Supabase API key.

A publishable or secret API key is not itself a user JWT and should not be treated as proof of a user's identity.

Reference: [Authorization headers](https://supabase.com/docs/guides/functions/auth-headers), [Function configuration](https://supabase.com/docs/guides/functions/function-configuration), and [Securing Edge Functions](https://supabase.com/docs/guides/functions/auth).

### 2.2 Secrets and privileged keys

Supabase documents that Edge Functions can read environment variables using `Deno.env.get(...)`. Local function secrets are loaded through `supabase/functions/.env`; production secrets can be managed through the Supabase Dashboard or CLI. The local environment file must be excluded from Git.

Supabase's secret/service-role key bypasses Row Level Security (RLS). It must remain server-side and must never be exposed in browser code. The documented secret naming rule also reserves the `SUPABASE_` prefix for Supabase-provided variables.

Reference: [Environment variables and secrets](https://supabase.com/docs/guides/functions/secrets).

### 2.3 Duitku POP Create Invoice request authentication

The Duitku POP documentation specifies the Create Invoice endpoint and requires these headers:
- `Content-Type`
- `x-duitku-timestamp`
- `x-duitku-signature`
- `x-duitku-merchantcode`

The documented request signature formula is:

~~~text
stringToSign = merchantCode + timestamp
signature = HMAC_SHA256(stringToSign, apiKey)
~~~

The timestamp is documented as UNIX time in milliseconds. Use the exact request contract and signing inputs for POP; do not substitute a signature formula from another Duitku product.

Reference: [Duitku POP API Reference](https://docs.duitku.com/pop/id/), Backend/Server → Create Invoice.

### 2.4 Duitku POP HTTP Callback

The POP documentation describes the server-to-server callback as HTTP POST using `application/x-www-form-urlencoded`. It documents a callback signature formula:

~~~text
stringToSign = merchantCode + amount + merchantOrderId
signature = HMAC_SHA256(stringToSign, apiKey)
~~~

The documentation says the older MD5 signature method is obsolete. It also says callback delivery is retried if Duitku does not receive HTTP 200, with a documented maximum of five attempts.

The same POP page contains conflicting descriptions of callback `resultCode`: one callback parameter table lists `00` as success and `01` as failed, while an error section lists `00` as success and `02` as failed. Redirect and JavaScript callback codes are described in a different context. The repository must not assume these contexts are interchangeable.

Reference: [Duitku POP API Reference](https://docs.duitku.com/pop/id/), Backend/Server → Callback, Frontend/View → JS Callback and Window Redirection, and Errors.

### 2.5 Frontend callback is not payment proof

Duitku's POP documentation states that the frontend JavaScript callback is for customer notification and must not be used to update the merchant's payment status. The server-side HTTP Callback is the documented mechanism for payment-status processing.

## 3. Security recommendations for this repository

The items below are recommended controls; they are not claims that Supabase or Duitku mandates these exact implementation details.

### 3.1 Protect secrets and environments

1. Keep `DUITKU_API_KEY` and `DUITKU_MERCHANT_CODE` on the server only. These names are a proposed repository convention, not provider-mandated names.
2. Never commit real credentials, local `.env` files, raw API responses containing sensitive customer information, or production tokens.
3. Keep sandbox and production credentials separate and choose the endpoint through explicit server-side configuration. Do not let a request parameter choose an arbitrary upstream URL.
4. Do not return secrets, signing inputs, internal stack traces, or full provider error bodies to a public caller.
5. Do not log the API key, Authorization header, full signature inputs, or unnecessary customer/payment data. Log a safe correlation identifier and sanitized failure category instead.
6. If a secret is exposed, treat it as compromised and follow the credential rotation procedure for the provider/project. Do not paste the replacement into Git or chat logs.

### 3.2 Authenticate and authorize each caller

Authentication and authorization are separate checks: authentication establishes who is calling; authorization establishes which order/payment they may act on or read.

| Endpoint | Caller expected | Recommended security boundary |
| --- | --- | --- |
| `duitku-create-invoice` | Storefront, authenticated customer, or trusted server | Verify the caller may initiate payment for the target order. If guest checkout is supported, define a deliberate guest ownership proof or trusted server-to-server mechanism. Never rely only on a guessable order ID. |
| `duitku-callback` | Duitku server | If platform JWT verification is disabled so Duitku can reach the handler, independently validate the provider signature and transaction fields before processing. |
| `get-payment-status` | Customer or trusted server | Enforce access to that specific order/payment and return only the minimum fields needed. Do not expose another customer's status by accepting an arbitrary order ID. |

Do not disable JWT verification globally merely because the callback requires it. Any `verify_jwt` setting should be per-function, explicitly reviewed, and consistent with the real caller. A disabled platform JWT check is not equivalent to authenticated access.

### 3.3 Validate the Create Invoice request

Before calling Duitku:
- Validate HTTP method, content type, body size, JSON shape, required fields, field lengths, and allowed values.
- Load the order and payable amount from a trusted database/service. Never use a browser-supplied total as the authoritative amount.
- Verify the order exists, is payable, and has not already reached a state that forbids another invoice.
- Match the requested amount against the server-side amount and the exact documented request contract.
- Use a bounded network timeout and handle non-2xx responses without exposing provider internals.
- Avoid blind retries after timeout or an ambiguous provider response. Duitku may have received a request even when the caller timed out.
- Treat duplicate invoice creation and concurrent attempts as a data-consistency problem requiring an explicit idempotency strategy.

### 3.4 Verify callbacks before any payment-state mutation

Recommended order of operations:

1. Accept the documented callback method and content type.
2. Parse the form-urlencoded body safely; reject malformed payloads and define how duplicate keys are handled.
3. Validate required fields and their expected format before concatenating values for signature verification.
4. Compute the documented HMAC-SHA256 callback signature using the exact official signing string and compare it safely with the received signature.
5. Confirm the merchant code is the expected merchant.
6. Match `merchantOrderId` to an existing stored payment/order and compare the callback amount to the expected payable amount using the agreed canonical format.
7. Validate any stored Duitku reference or other transaction identity when available.
8. Resolve `resultCode` only through a documented and confirmed mapping. Unknown codes must not default to success.
9. Persist the callback event and apply state changes atomically and idempotently, once the data model supports that guarantee.
10. Return the agreed HTTP response only after the processing outcome is known.

Signature verification alone does not prove that the callback is applicable to the local order: the amount, merchant code, order identity, and expected transaction must also match. Conversely, a mismatched callback must never be used to mark an order paid.

Do not assume a particular HTTP response for invalid or mismatched callbacks without deciding and testing the policy. Duitku documents callback retries when HTTP 200 is not received; acknowledgment policy therefore affects recovery and should be explicit.

### 3.5 Use safe cryptographic handling

- Use the documented HMAC-SHA256 algorithm and exact POP signing inputs. Request and callback signatures use different input strings.
- Do not use the obsolete MD5 callback formula.
- Use a vetted platform cryptography API and a comparison approach appropriate for fixed-format signature values; normalize encoding only where the official contract supports it.
- Do not include the API key in the signing string as literal text; it is the HMAC key according to the documented formula.
- Add deterministic tests against official examples or independently verified test vectors before trusting the implementation.
- Do not invent timestamp tolerance, case normalization, or signature acceptance rules not established by the provider contract.

### 3.6 Handle duplicate and out-of-order events

Callbacks can be retried, and clients can submit repeated invoice requests. Recommended controls:
- Give each incoming event a trace/correlation identifier.
- Use database constraints and transactions or an equivalent atomic operation to prevent duplicate business effects.
- Store enough safe event metadata to investigate replays and mismatches without keeping unnecessary sensitive payload data.
- Define how repeated identical callbacks, conflicting callbacks, unknown references, and out-of-order events are handled.
- Never move a payment from a terminal state to success solely because a later untrusted client request says it succeeded.
- Define late-payment, expiry, refund, and duplicate-payment reconciliation separately; do not infer their rules from the basic callback contract.

These controls require schema and state-transition decisions and are not implemented by this document.

### 3.7 Error handling and observability

- Use generic public error responses; retain sanitized diagnostic details in access-controlled server logs.
- Avoid logging credentials, raw authorization headers, full payment payloads, or unnecessary personal information.
- Distinguish validation failure, signature failure, provider rejection, timeout, database failure, and unknown provider status.
- Ensure callback acknowledgment behavior is consistent with durable processing and the agreed retry strategy.
- Add alerts or operational review for repeated signature failures, amount mismatches, unknown result codes, and reconciliation failures, subject to available logging/monitoring.
- Do not claim an event was processed if required persistence failed.

### 3.8 Abuse and availability controls

Consider request-size limits, method/content-type checks, rate limiting or other abuse controls, bounded upstream requests, and safe handling of bursts. Any rate limit must account for legitimate provider callback retries; do not use a policy that silently drops payment notifications without a recovery path.

## 4. Open security decisions

The following must be decided before implementation or release:

1. **Caller identity:** whether the storefront uses Supabase Auth, whether guest checkout is allowed, and how guest order ownership is proven.
2. **JWT configuration:** whether each function uses platform JWT verification. The callback may need a public gateway route, while the invoice and status endpoints may need user or trusted-service authentication. This document does not set `verify_jwt`.
3. **Data source and privileges:** actual table names, RLS policies, grants, RPCs, and whether a privileged secret key is necessary. No schema or access-control changes are authorized by this document.
4. **Callback result mapping:** resolve the official POP documentation's conflicting `resultCode` definitions before defining internal payment statuses.
5. **Mismatch policy:** HTTP response and operational handling for invalid signatures, wrong merchant code, unknown order, amount mismatch, and unknown result code.
6. **Atomicity/idempotency:** event storage, unique constraints, allowed state transitions, concurrency handling, and duplicate invoice prevention.
7. **Timestamp validation:** any allowed clock skew or replay window for the Create Invoice timestamp, if required by the current provider contract.
8. **Data retention:** which callback fields are necessary for audit/reconciliation and how long to retain them.
9. **Incident response:** how to rotate exposed Duitku/Supabase credentials and investigate suspicious callback traffic.
10. **Production readiness:** required tests, sandbox evidence, monitoring, review, and explicit approval before setting production secrets or deploying.

## 5. Minimum security test matrix

Before treating the functions as ready, tests should cover at least:

| Area | Cases |
| --- | --- |
| Invoice request | Missing fields, malformed JSON, invalid lengths, wrong caller, order not payable, amount altered by client |
| Request signature | Valid signature, wrong API key, wrong timestamp/signing input, missing signature headers |
| Callback parsing | Correct form encoding, malformed body, missing required fields, duplicate keys |
| Callback signature | Valid signature, modified amount/order ID, wrong signature, missing signature |
| Transaction matching | Unknown order, wrong merchant code, amount mismatch, wrong/unknown reference |
| Result codes | Confirmed success, confirmed failure, unknown code, conflicting/ambiguous code |
| Replay/idempotency | Same callback repeated, concurrent duplicate callbacks, conflicting/out-of-order callbacks |
| Failure recovery | Provider timeout, HTTP 409, non-2xx provider response, database unavailable during callback |
| Access control | Unauthenticated caller, unauthorized order access, cross-customer status query |
| Secret hygiene | No credentials in Git, browser bundle, response bodies, or logs |

Tests must use sandbox/test credentials and synthetic data. Do not perform real payments as part of this test plan.

## 6. Out of scope

This file does not create or modify code, schema, migrations, RLS, grants, RPCs, secrets, JWT configuration, or deployment. It does not assert that any of the recommendations have been implemented or verified in a running project.

## 7. Official references

### Supabase

1. [Securing Edge Functions](https://supabase.com/docs/guides/functions/auth)
2. [Authorization headers](https://supabase.com/docs/guides/functions/auth-headers)
3. [Function configuration](https://supabase.com/docs/guides/functions/function-configuration)
4. [Environment variables and secrets](https://supabase.com/docs/guides/functions/secrets)
5. [Edge Functions overview](https://supabase.com/docs/guides/functions)

### Duitku

1. [Duitku POP API Reference (Bahasa Indonesia)](https://docs.duitku.com/pop/id/) — Create Invoice request signature, HTTP Callback signature and payload, frontend callback guidance, retry behavior, error responses, and changelog.

If the official documentation changes or contradicts itself, record the exact ambiguity and obtain a confirmed provider contract rather than inventing security behavior.
