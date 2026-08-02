---
name: Withdraw from a Celsius wallet and reconcile the transaction
description: Check the balance and withdrawal scheme, submit a withdrawal, and drive the resulting transaction to a final state without an idempotency key.
api: openapi/celsius-network-partner-api-openapi.yml
operations: [getCoinBalance, getWithdrawalAddresses, getWithdrawalAddressForCoin, withdraw, getTransactionStatus, getCoinTransactions, getStatistics]
generated: '2026-08-02'
method: generated
source: openapi/celsius-network-partner-api-openapi.yml
status: retired
---

# Withdraw from a Celsius wallet and reconcile the transaction

> **This API is retired.** `wallet-api.celsius.network` no longer resolves. Celsius Network
> froze withdrawals on 2022-06-12 and shut down its platform on 2024-02-29. This skill is a
> historical record of how the flow worked.

This is the money-moving flow and the riskiest one in the API: `withdraw` is irreversible and
**there is no idempotency key**.

## Authentication

`X-Cel-Partner-Token` plus `X-Cel-User-Token` (Segmented Integration) or `X-Cel-Api-Key`
(Omnibus / Omnibus Treasury).

## Steps

1. **Check funds.** Call `getCoinBalance` for the coin. `amount` is a numeric string; parse it
   as a decimal. Withdrawing more returns `422` with slug `INSUFFICIENT_FUNDS`.
2. **Check the withdrawal scheme.** The partner account is configured with one of four
   schemes: no address (withdrawals disabled), a list of pre-approved addresses, the origin
   address of the first deposit, or any address. Call `getWithdrawalAddresses` for every
   permitted address, or `getWithdrawalAddressForCoin` for the one that applies to this coin.
   Submitting an address the scheme does not allow returns `422` with slug
   `INVALID_ADDRESS_FOR_WITHDRAWAL_SCHEME`.
3. **Submit the withdrawal.** Call `withdraw` with the coin in the path and
   `multipart/form-data` carrying `address` and `amount`. The response carries
   `transaction_id` — persist it **before** doing anything else. It is your only handle on
   this transfer.
4. **Drive it to a final state.** Poll `getTransactionStatus` with that `transaction_id`. The
   `state` moves `processing` → `unconfirmed` → `confirmed`, or lands on `rejected`. `tx_id`
   is `null` until the transaction reaches the chain, then carries the blockchain hash.
   `confirmed` and `rejected` are the only final states.
5. **Reconcile.** Cross-check against `getCoinTransactions` for that coin, matching on the
   transaction `id` and looking for `nature: withdrawal`. For partner-level totals, call
   `getStatistics`, optionally with a `timestamp` lower bound, which returns
   `withdrawal_count` and `withdrawal_amount` rolled up in USD and broken down per coin.

## Rules

- **Never blindly retry `withdraw`.** There is no idempotency key, so a retry after a timeout
  can send funds twice. On any ambiguous failure, list `getCoinTransactions` for the coin and
  look for a `withdrawal` record at the expected amount and time before re-issuing.
- **Persist `transaction_id` before acknowledging success upstream.** If you lose it, the only
  recovery is scanning the paged transaction list.
- Minimum withdrawal is one US dollar. Below it you get `400` with slug `DUST_CHECK_FAILED`.
  Not retryable unchanged.
- `400`/`422` with slug `WRONG_TRANSACTION_ID` or `BAD_TRANSACTION_ID_FORMAT` means the id is
  unknown or malformed — do not treat it as "transaction not yet visible" and keep polling.
- A `401` with slug `COMPLIANCE_ERROR` on `withdraw` means withdrawals are suspended for the
  user's jurisdiction. Terminal, not an auth failure.
- Branch on `slug`, never on `message` — messages interpolate country and state names.
- Verify the `X-Signature` response header before acting on any response; on a money-moving
  call an unverified response must be treated as a failure of unknown outcome, which means
  reconciling against the transaction list rather than retrying.

## Artifacts

- Error catalog: `errors/celsius-network-problem-types.yml`
- Conventions (no idempotency, page/per_page pagination, error envelope): `conventions/celsius-network-conventions.yml`
- Data model: `data-model/celsius-network-data-model.yml`
