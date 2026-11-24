# Programmable payments design notes

## Existing work

The upstream tracker had four open submissions covering the full bounty at review time and several staged submissions. A useful next code change would target a specific gap in the candidate maintainers choose and preserve that candidate's consensus behavior.

| Candidate | Execution model | Distinction |
| --- | --- | --- |
| [#128](https://github.com/IDNI/tau-testnet/pull/128) | Feature-specific books for standing authorizations, receive routes, Tau payouts and escrowed grants | Signed standing instructions are executed by an open relayer. Grant draws have their own fee. |
| [#145](https://github.com/IDNI/tau-testnet/pull/145) | A programmable-payments book with a post-block sweep | An allowance can set a hold relative to its funding block and an inactivity refund. |
| [#146](https://github.com/IDNI/tau-testnet/pull/146) | Feature books and a unified Tau runner for clock, arrival and explicit execution | Signed configuration bounds clause output and adds settlement keys and shared caps. |
| [#167](https://github.com/IDNI/tau-testnet/pull/167) | Deterministically ordered sweep with explicit standing-payment fallback | The payer funds a bounded standing execution fee. The standing authorization can be versioned by an update transaction. |

## Design decisions

### Triggers and fees

An owner signs the payment terms once. Regular wallet transfers can trigger a receive split. A block's timestamp can make a standing installment or refund due. An explicit transaction can trigger an authorized settlement or draw from escrow. A rule-generated credit should be terminal for routing to keep a single payment from causing a cycle. Make credit eligibility explicit and persist deterministic settlement identifiers so retries and replay cannot pay the same occurrence twice.

For a fee-bearing explicit trigger, charge the submitting account under its signed fee limit. For a block-triggered installment, the owner can authorize a maximum fee with the terms, and the installment settles only when its principal and fee are covered. A receive split shares the original transfer's fee. Fund a grant's principal independently of fees. Charge the grantor for funding and the grantee from their ordinary balance for a draw. Refund the whole unspent principal without a fee or a route deduction. PR #128 uses executor-funded relays for standing payments. The block-triggered standing fee is a separate choice implemented in PR #167.

### Time, availability and accounting

Use the canonical block timestamp in UTC. For a calendar day the boundary is 00:00 UTC, for a week it is Monday at 00:00 UTC, and for a calendar month it is the first day of the month at 00:00 UTC. Clamp an installment scheduled for the 29th, 30th or 31st to the last day when needed, with the original day restored in the next eligible month. A fixed-length window may use an explicit anchor. Skip missed windows by default.

Keep `(period index, gross spent)` with each grant, or with each recipient in a multi-recipient authorization. Reset the counter logically when the period index changes. An accepted settlement increments it once. Keep ordinary spendable balances and reserved principal separate, and include both in supply accounting. Derive a temporary hold from the funding block's timestamp. Combine it with the absolute start time by taking the later boundary. Apply hour and weekday masks in UTC.

[Issue #62](https://github.com/IDNI/tau-testnet/issues/62) describes third-party signed claims. If a payout depends on delivery or a price, its rule can trust an explicitly configured attester once that feature exists. The node should verify the attestation's signature, subject, chain and freshness before exposing it to Tau. Periods and expiry can use block time independently of that feature.

### Refund precedence and atomicity

Check expiry and any configured inactivity deadline before draw eligibility. A draw may proceed only while the grant is open, after the effective unlock time, within the availability window and under both the per-draw and period caps. The remaining allowance is the smallest of the signed limits and unspent principal. A draw at the deadline loses to the refund. A successful draw can reset an inactivity timer while leaving the absolute expiry fixed. At the first eligible block, refund the remainder and close the grant. If the block caps sweep work, the expired grant remains unavailable while its refund waits in the deterministic queue.

Verify rule outputs against the authorized recipient set and aggregate caps. Commit all resulting debits, credits, counters and receipts together. Clear no occurrence marker and advance no spending counter on a failed attempt. Bind authorizations to the network's genesis hash and reuse the admission checks at block application.

## Verification boundaries

The measured tests cover unsigned period and authorization logic, fee and routing arithmetic, scripted payout outputs, and a local Alice/Bob block-apply run for PR #128. Absolute expiry and periods were exercised in that run. Relative holds, inactivity expiry, native Tau execution and the signed multi-node path were outside that run. See the [README](README.md) for exact revisions, commands and results.
