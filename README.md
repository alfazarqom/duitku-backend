# Duitku POP Backend — Supabase

Backend integration for Duitku POP using Supabase Edge Functions (Deno/TypeScript).

> **Status:** Initial repository scaffold. This README is an orientation document, not evidence that any function, database schema, credential, or deployment already exists.

## Scope

The planned repository layout is:

- `docs/` — official-source register, architecture, security, and API contracts.
- `supabase/functions/_shared/` — shared Duitku HTTP/signature helpers and input validation.
- `supabase/functions/duitku-create-invoice/` — server-side invoice creation.
- `supabase/functions/duitku-callback/` — server-to-server payment notification receiver.
- `supabase/functions/get-payment-status/` — application-facing payment status read endpoint; its data source and authorization contract must be defined before implementation.
- `supabase/migrations/` — database changes, only after the schema and transaction model are reviewed and explicitly approved.
- `supabase/tests/` — test plan and test instructions.
- `.github/workflows/ci.yml` — automated checks for the repository.

The layout is a target structure, not a claim that all listed files have been created.

## Official Duitku POP contract — verified starting points

Use the official Indonesian POP documentation as the primary source of truth:

- [Duitku POP API Reference (Indonesia)](https://docs.duitku.com/pop/id/)
- [Duitku API Reference (Indonesia)](https://docs.duitku.com/api/id/)

The POP documentation currently specifies the Create Invoice request as HTTP POST with JSON to:

- Sandbox: `https://api-sandbox.duitku.com/api/merchant/createInvoice`
- Production: `https://api-prod.duitku.com/api/merchant/createInvoice`

It documents the request headers `Content-Type`, `x-duitku-timestamp`, `x-duitku-signature`, and `x-duitku-merchantcode`. The documented request signature uses HMAC-SHA256 over `merchantCode + timestamp`, keyed with the API key. The callback is a separate server-to-server HTTP POST using `application/x-www-form-urlencoded`; its documented signature uses HMAC-SHA256 over `merchantCode + amount + merchantOrderId`, keyed with the API key.

These contracts must be checked against the linked official documentation again when implementing the corresponding file. Do not substitute contracts from another Duitku product or an outdated MD5 signature example.

The POP documentation explicitly warns that browser JS callback and return-URL parameters are not authoritative evidence for changing payment state. Payment-state changes must be based on the server callback after validation and matching against the local transaction.

## Security and implementation rules

1. Keep Duitku credentials in Supabase Edge Function secrets or another approved server-side secret store. Never commit real credentials or expose them to browser code.
2. Select Sandbox or Production explicitly through server-side configuration. Do not silently fall back between environments.
3. Validate request fields, provider responses, callback signatures, merchant/order/amount matching, and allowed state transitions.
4. Make callback processing idempotent and safe under retries and concurrent requests. Database writes must follow a reviewed transaction contract.
5. Do not trust client-supplied totals or browser redirect/JS callback values as payment proof.
6. Do not invent undocumented Duitku endpoints, fields, response codes, signature inputs, or retry semantics. Record ambiguities and obtain confirmation where official documentation is insufficient.
7. Do not perform real payments, change production configuration, deploy functions, or apply database migrations without explicit approval.

## Configuration

See [`.env.example`](./.env.example) for variable-name placeholders when that file is added. Configure actual secrets in the approved secret manager; never put secret values in this repository.

## Development status

No database schema, migration, Edge Function, CI workflow, or deployment is asserted to be complete by this README alone. Each file should be created and reviewed individually, against the relevant official Duitku and Supabase documentation.

## Documentation

- [Source of truth](./docs/source-of-truth.md) — planned official-source register.
- [Architecture](./docs/architecture.md) — planned component and data-flow description.
- [Security](./docs/security.md) — planned threat controls and authorization boundaries.
- [API contracts](./docs/api-contracts.md) — planned request/response contracts.
- [Test instructions](./supabase/tests/README.md) — planned testing guide.
