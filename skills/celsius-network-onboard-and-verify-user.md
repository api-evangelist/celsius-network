---
name: Onboard and KYC-verify a Celsius partner user
description: Create an end user under a Segmented Integration partner account, submit their identity data and documents, and poll the KYC application to a final state.
api: openapi/celsius-network-partner-api-openapi.yml
operations: [getSupportedCountries, createUser, startKycVerification, getKycVerificationStatus, getKycStatus, verifyKyc]
generated: '2026-08-02'
method: generated
source: openapi/celsius-network-partner-api-openapi.yml
status: retired
---

# Onboard and KYC-verify a Celsius partner user

> **This API is retired.** `wallet-api.celsius.network` no longer resolves. Celsius Network
> shut down its platform on 2024-02-29 as part of its Chapter 11 wind-down. This skill is a
> historical record of how the flow worked — do not attempt to run it against a live host.

Applies to **Segmented Integration** partners only. Omnibus and Omnibus Treasury partners do
not create users and do not run KYC through this API.

## Authentication

Every request carries `X-Cel-Partner-Token` (the partner key Celsius issued you). Once a user
exists, requests on that user's behalf also carry `X-Cel-User-Token`, whose value is the
`user_token` you supplied at creation.

`createUser` is the exception: it is authenticated by the partner token alone, because the
user does not exist yet.

## Steps

1. **Check the country is supported.** Call `getSupportedCountries` and confirm the applicant's
   country of residence appears in `countries[]`. KYC will fail later on an unsupported
   country, and jurisdictional blocks surface as a `401` with slug `COMPLIANCE_ERROR` — which
   is terminal, not retryable.
2. **Create the user.** Call `createUser` with `multipart/form-data` carrying at minimum
   `first_name`, `last_name`, `date_of_birth`, `citizenship`, `country`, `city`, `zip`,
   `street`, `gender` and your own `user_token`. `state` is required when `country` is
   United States. Country accepts an ISO 3166 name, official state name, alpha-2 or alpha-3
   code; US `state` accepts the two-letter code. Store the returned `userId` — it is the path
   parameter for everything that follows — and store your `user_token` as this user's secret.
3. **Upload identity documents.** Call `startKycVerification` with the `userId` in the path and
   `multipart/form-data` carrying `document_type` (`passport`, `identity_card` or
   `driving_licence`), `document_front_image` and, where the document type has one,
   `document_back_image`. Keep each upload under 25 MB — that is the documented ceiling.
4. **Poll to a final state.** Call `getKycVerificationStatus` with the `userId` (or
   `getKycStatus` for the currently authenticated user). `status` moves through the
   intermediate states `COLLECTING` and `PENDING` to one of the final states `PASSED`,
   `REJECTED` or `PERMANENTLY_REJECTED`. Stop polling on a final state.
5. **Handle rejection.** On `REJECTED`, read the `reasons` object — each key is a rejection
   reason and each value is the string `consider`. Correct the underlying issue and resubmit
   with `startKycVerification`. On `PERMANENTLY_REJECTED`, stop: no further verification can be
   started for this user, and a resubmission returns `422` with slug
   `KYC_PERMANENTLY_REJECTED`.

`verifyKyc` (`POST /kyc`) is the older single-shot variant: it creates the KYC application from
applicant data and documents in one call, without a separate `createUser`. Prefer the
two-step flow above, which is what `celsius-sdk` 1.0.0 shipped.

## Rules

- **No idempotency key exists.** Never blindly retry `createUser` or `startKycVerification` on
  a timeout — read state back with `getKycVerificationStatus` first.
- **These calls carry PII** — names, dates of birth, addresses, SSN/ITIN/national ID and
  identity document images. Do not log request bodies.
- Errors return `{message, slug}` on `application/json`, not RFC 9457 problem details. Branch
  on `slug`, never on `message` — the message text varies by jurisdiction. Some older
  responses return only `msg` with no `slug`.
- `422` with slug `KYC_PASSED` or `KYC_PENDING` means you are duplicating work already in
  flight. Read status instead of resubmitting.
- Verify the `X-Signature` response header against the environment's RSA public key. The
  official SDK does this automatically and rejects unverified responses.

## Artifacts

- Error catalog: `errors/celsius-network-problem-types.yml`
- Conventions (auth, pagination, error envelope): `conventions/celsius-network-conventions.yml`
- Data model: `data-model/celsius-network-data-model.yml`
