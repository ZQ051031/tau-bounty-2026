# Tau Bounty #01 — Programmable payments

Independent review and verification for Tau's programmable-payments bounty. The upstream PRs collectively implement the requested payment behaviors, including the 5,000-unit Alice/Bob example. Their consensus semantics differ, and the maintainers require design approval before implementation. The [upstream audit](AUDIT.md) maps the code and test evidence, shows where the implementations differ, and explains why this review produced no new implementation PR. The [user's Tau fork](https://github.com/ZQ051031/tau-testnet) was cloned for the review.

## Scope

- [#34 — Standing payments](https://github.com/IDNI/tau-testnet/issues/34). An owner authorizes a fixed payment once. An eligible block or a relayer can trigger the installment, depending on the candidate.
- [#35 — Receive routing](https://github.com/IDNI/tau-testnet/issues/35). A recipient authorizes a split once. At 20%, a gross transfer of 101 sends 20 to savings and 81 to the recipient, with a single transfer fee.
- [#63 — Rule-computed payouts](https://github.com/IDNI/tau-testnet/issues/63). A Tau clause computes the amounts from its inputs. The node enforces the signed recipient set, payment caps and available balance before settling the legs atomically.
- Conditional grants reserve principal and allow draws under a combined schedule, spending cap and refund deadline.

[Design notes](DESIGN.md) cover clock choice, trigger boundaries, fees, period accounting, temporary holds and refund precedence.

## Run the verified unit suites

Use Python 3.14. The following commands install only the dependencies needed by these selected tests. No wallet, account setup or transaction signing is required.

```sh
git clone https://github.com/IDNI/tau-testnet.git
cd tau-testnet
python3.14 -m venv .venv
. .venv/bin/activate
python -m pip install pytest==9.1.1 pytest-asyncio==1.4.0 multiaddr==0.2.0 \
  trio==0.34.0 py-ecc==8.0.0 py_arkworks_bls12381==0.5.0 blake3==1.0.11
git fetch origin pull/128/head pull/146/head pull/167/head

# PR #128
git switch --detach 007301046a495e6ae0da0ced5691cec88b0d1bd4
python -m pytest -q tests/test_periods.py tests/test_grant_book.py tests/test_standing_book.py

# PR #146
git switch --detach c1cb7f2f98a77f0bb05242f86acff5627b65ca30
python -m pytest -q tests/test_payments_periods.py tests/test_payments_grants.py \
  tests/test_payments_payouts.py tests/test_payments_routes.py tests/test_payments_standing.py

# PR #167
git switch --detach 7dea6b3d9d81ec87287fd975d639a29afbe1eccd
python -m pytest -q tests/test_periods.py tests/test_sweep.py
```

| Candidate | Verified scope | Passed | Failed | Skipped |
| --- | --- | ---: | ---: | ---: |
| [#128](https://github.com/IDNI/tau-testnet/pull/128) | Calendar arithmetic, grant conditions and standing authorizations | 385 | 0 | 0 |
| [#146](https://github.com/IDNI/tau-testnet/pull/146) | Periods, grants, payout caps and atomicity, receive routes and standing payments | 45 | 0 | 0 |
| [#167](https://github.com/IDNI/tau-testnet/pull/167) | Calendar periods and deterministic sweep | 18 | 0 | 0 |

The first attempt with repository fixtures stopped because `multiaddr` was missing. The final runs used the dependency list above and loaded the repository fixtures. Payout unit tests use a callable to supply rule outputs. The [upstream audit](AUDIT.md) records the subsequent native Tau runs, the signed two-node CI evidence, and the verification limits.

## Alice and Bob

The example uses 5,000 test units and sets the refund deadline to 2027-03-01 00:00 UTC. The local check invoked PR #128's production `TauConsensusEngine.apply_block` for seven block timestamps. It used synthetic public points and scripted signature verification, chain identity and Tau evaluation. Its funding and draw fees were set to 3 test units.

| Step | Outcome | Escrow |
| --- | --- | ---: |
| Alice opens the grant | 5,000 locked for Bob | 5,000 |
| 1 second before Dec 1 | Bob's draw rejected | 5,000 |
| Dec 1 at 00:00 UTC | Bob draws 300 and 200 to the shop | 4,500 |
| 1 second before Monday, Dec 7 | An extra unit rejected at the weekly cap | 4,500 |
| Dec 7 at 00:00 UTC | The weekly cap resets and Bob draws 500 | 4,000 |
| Mar 1 at 00:00 UTC | A same-block draw rejected and 4,000 refunded to Alice without a fee | 0 |
| Next block | No second refund | 0 |

The check asserted exact balance and escrow totals at each step. Final balances were Alice 8,997, Bob 41, shop 1,000 and proposer 12, from initial balances of 10,000 for Alice and 50 for Bob.

Save the independent `alice_bob_block_apply.py` attachment locally. From the Tau checkout, run it with the same environment as the unit tests.

```sh
git switch --detach 007301046a495e6ae0da0ced5691cec88b0d1bd4
PYTHONPATH=. TAU_ENV=test TAU_FORCE_TEST=1 python /path/to/alice_bob_block_apply.py
```

For a self-contained grant unit check with the January 1 deadline and 3,800 expected refund, run PR #146's named case.

```sh
git switch --detach c1cb7f2f98a77f0bb05242f86acff5627b65ca30
python -m pytest -q tests/test_payments_grants.py::test_grant_5000_returns_3800_after_1200_drawn
```

The candidate's [signed local-network demo](https://github.com/IDNI/tau-testnet/blob/007301046a495e6ae0da0ced5691cec88b0d1bd4/scripts/e2e_programmable_payments.py) requires private fixture keys and the native Tau engine. The verification above stayed within the key-free scope.
