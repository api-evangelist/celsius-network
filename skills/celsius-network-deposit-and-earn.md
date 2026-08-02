---
name: Fund a Celsius wallet and track accrued interest
description: Get a per-coin deposit address for a user, confirm the deposit landed, and read the interest accruing on the balance.
api: openapi/celsius-network-partner-api-openapi.yml
operations: [getSupportedCurrencies, getInterestRates, getDeposit, getCoinTransactions, getTransactionSummary, getCoinBalance, getBalanceSummary, getInterestSummary]
generated: '2026-08-02'
method: generated
source: openapi/celsius-network-partner-api-openapi.yml
status: retired
---

# Fund a Celsius wallet and track accrued interest

> **This API is retired.** `wallet-api.celsius.network` no longer resolves. Celsius Network
> shut down its platform on 2024-02-29 as part of its Chapter 11 wind-down. This skill is a
> historical record of how the flow worked.

The core earning loop: pick a coin, deposit to it, watch the deposit confirm, then read the
interest it accrues. Works for all three partnership types.

## Authentication

`X-Cel-Partner-Token` on every request, plus `X-Cel-User-Token` (Segmented Integration) or
`X-Cel-Api-Key` (Omnibus / Omnibus Treasury). `getInterestRates` needs no user credential.

## Steps

1. **Confirm the coin is supported.** Call `getSupportedCurrencies` and check the symbol
   appears in `currencies[]`. An unsupported symbol returns `422` with slug `COIN_NOT_FOUND`
   from every coin-scoped operation.
2. **Read the rate before committing.** Call `getInterestRates`. Each entry carries `coin` and
   `rate` as a numeric string — `"0.0200"` is 2%. Rates were per-coin and changed over time.
3. **Get the deposit address.** Call `getDeposit` with the coin in the path. The response
   carries `address`. **For BTC, `address` is the segwit address and `addressSecondary` is the
   legacy address** — pick the one that matches the sending wallet's capability. A `503`
   ("No free addresses available") is the one transient error in this API and is safe to retry.
4. **Send the funds on-chain**, outside this API, to that address.
5. **Watch the deposit confirm.** Poll `getCoinTransactions` for that coin. Look for a record
   with `nature: deposit`. Its `state` moves `processing` → `unconfirmed` → `confirmed`
   (or `rejected`). Only `confirmed` and `rejected` are final. Use `page` and `per_page` and
   read the `pagination` wrapper — the newest transactions are first, sorted by `time`
   descending.
6. **Read the balance.** `getCoinBalance` for one coin (returns `amount` plus `amount_in_usd`)
   or `getBalanceSummary` for every coin at once. Note that `getBalanceSummary` keys its
   `balance` map in **lowercase** symbols while the rest of the API uses uppercase.
7. **Read accrued interest.** Call `getInterestSummary`. Each coin's entry splits interest into
   `amount` (paid In Kind, in the coin itself) and `amount_cel` (paid In CEL, the CEL token),
   with `amount_usd` covering both. `total_amount_usd` is the sum across all coins.

## Rules

- **All amounts are numeric strings, not floats.** Parse them as decimals. On a transaction
  record, `amount` is rounded down to the last significant decimal place and `amount_precise`
  carries the unrounded value — reconcile against `amount_precise`.
- USD figures are converted at the transaction's `time`, not at read time.
- A `401` with slug `COMPLIANCE_ERROR` on `getDeposit` means deposits are suspended in the
  user's jurisdiction even though the account is otherwise usable — withdrawals typically
  remained available. Do not treat it as an auth failure and do not retry.
- Interest transactions carry `original_interest_coin` and
  `interest_amount_in_original_coin` when the interest was accrued in one coin and paid in
  another; both are absent unless `nature` is `interest`.
- The paged records field is documented as `record`; some responses use `records`. Accept both.

## Artifacts

- Error catalog: `errors/celsius-network-problem-types.yml`
- Conventions: `conventions/celsius-network-conventions.yml`
- Data model: `data-model/celsius-network-data-model.yml`
