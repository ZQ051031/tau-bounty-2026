# Programmable payments upstream audit

Existing open PRs collectively implement recurring payments, receive splits, Tau-computed payouts, and combined grant conditions. The audit found differences in how the proposals implement those features. In particular, the branch with native evidence for grant conditions on one Tau rule defers some receive splits, while the same-transaction routing branch implements grants through a grant-specific Python path. Tau's critical-bounty process calls for approval before building, and the maintainer's acceptance criterion is that new policies and combinations can be expressed in Tau rules and configuration without a new feature-specific Python execution path. The public threads reviewed did not establish approval for a design that resolves these differences. No implementation changes were made to the fork and no new upstream PR was opened. [Bounty program](https://tau.net/bounty-program) · [Maintainer review](https://github.com/IDNI/tau-testnet/pull/146#pullrequestreview-5995504067) · [Named acceptance cases](https://github.com/IDNI/tau-testnet/issues/34#issuecomment-7025966250)

## What the code implements

| Requirement | Evidence |
| --- | --- |
| Recurring payments ([#34](https://github.com/IDNI/tau-testnet/issues/34)) | #167 has a standing-payment book, a deterministic block sweep, and explicit execution. #160 and #177 schedule the same payout evaluator on consensus time. #160's native tests cover Jan 31 to the last day of February and back to Mar 31, missed periods, insufficient balance and retries. [#167 design](https://github.com/IDNI/tau-testnet/blob/7dea6b3d9d81ec87287fd975d639a29afbe1eccd/docs/programmable_payments.md) · [#160 native trigger tests](https://github.com/IDNI/tau-testnet/blob/cb8c42d2d999c913be3e48aaf5504dd5e7dfd17c/tests/test_payout_triggers_native.py) |
| Receive split ([#35](https://github.com/IDNI/tau-testnet/issues/35)) | #171 implements a dedicated split at the credit seam, one original fee, floor rounding, and terminal routed credits. #177 and #179 stage successful Tau payout legs with the crediting transaction. #177 tests a gross credit of 101, a share of 20 and a net credit of 81 with native Tau. [#171 route tests](https://github.com/IDNI/tau-testnet/blob/97128d11344b2e48d9dd570d4fee348a8fa92667/tests/test_receive_routes_bounty01.py) · [#177 native trigger tests](https://github.com/IDNI/tau-testnet/blob/70bd5a9db2da7dc96fa249b853528d9b4b4800ae/tests/test_payout_triggers_native.py) · [#179 arrival apply path](https://github.com/IDNI/tau-testnet/blob/8c99969a2640de3a868a60177bcae50327e8e72f/consensus/engine.py#L3748-L3790) |
| Rule-computed payouts ([#63](https://github.com/IDNI/tau-testnet/issues/63)) | #178's native cases test a 5% royalty on 101 paying five once and multi-leg escrow under signed caps. #179 adds eight rule slots per account, in-place updates that preserve settled IDs and period spend, bounded due accounting, sealed terms and explicit-execution fee caps. #178's head also contains a payout-update parser and update tests. [#178 native payout tests](https://github.com/IDNI/tau-testnet/blob/47a110e0a489b3e3c177941b16316b36a9b82a57/tests/test_payout_native.py) · [#178 update tests](https://github.com/IDNI/tau-testnet/blob/47a110e0a489b3e3c177941b16316b36a9b82a57/tests/test_payout_multirule.py) · [#179 payout book](https://github.com/IDNI/tau-testnet/blob/8c99969a2640de3a868a60177bcae50327e8e72f/docs/payout_book.md) |
| Limits, time locks and availability | #172 adds per-grant releases and a shared funder cap. #173 adds a lock, draw window and inactivity deadline. #158 and its descendant #160 express amount, executor, date, weekday and hour constraints in Tau; the rule reads the amount already spent in the period. #145 implements a hold measured from funding. [#172](https://github.com/IDNI/tau-testnet/pull/172) · [#173](https://github.com/IDNI/tau-testnet/pull/173) · [#158](https://github.com/IDNI/tau-testnet/pull/158) · [#160 native condition tests](https://github.com/IDNI/tau-testnet/blob/cb8c42d2d999c913be3e48aaf5504dd5e7dfd17c/tests/test_payout_conditions_native.py) · [#145 allowance code](https://github.com/IDNI/tau-testnet/blob/a060e5a472502df8ec5faddc90c1f7e760db7bc0/consensus/payments.py#L565-L581) |
| Automatic refund and combined conditions | #145 implements an idle or absolute deadline, whichever comes first. After either deadline, the draw fails and the sweep returns the full remainder. #176 tests a signed 5,000-unit grant with three weekly draws of 500 and a 3,500-unit refund on two local nodes. #160 checks the equivalent fixed-deadline payout using a native Tau rule. [#145 design](https://github.com/IDNI/tau-testnet/blob/a060e5a472502df8ec5faddc90c1f7e760db7bc0/docs/programmable_payments.md) · [#176 two-node test](https://github.com/IDNI/tau-testnet/blob/2c6b34693f7d809bb4ddf5208058012fded8182e/tests/test_grants_twonode_native.py#L529) · [#160 native deadline test](https://github.com/IDNI/tau-testnet/blob/cb8c42d2d999c913be3e48aaf5504dd5e7dfd17c/tests/test_payout_options_native.py#L100-L135) |

## Design questions still open for an upstream implementation

- **Same-transaction routing.** #160 runs arrival policies after ordinary transactions and can defer work past the originating block. Its route tests establish floor rounding and payout caps, while #35 also asks for the split to commit with the credit. #177 and #179 stage successful legs within the original transfer and suppress triggers on the resulting legs. #178 takes a third approach. It accepts scheduled payout credits as arrivals but does not re-enqueue the legs of an arrival run. One-hop termination and same-transaction accounting are separate properties. [#160 arrival semantics](https://github.com/IDNI/tau-testnet/blob/cb8c42d2d999c913be3e48aaf5504dd5e7dfd17c/docs/payout_rules.md#L575-L620) · [#177 trigger contract](https://github.com/IDNI/tau-testnet/blob/70bd5a9db2da7dc96fa249b853528d9b4b4800ae/docs/payouts-and-grants.md#L36-L64) · [#178 arrival engine](https://github.com/IDNI/tau-testnet/blob/47a110e0a489b3e3c177941b16316b36a9b82a57/consensus/engine.py#L348-L405)
- **Standing payment policy.** #34 asks that execution respect the current payer `o5` policy unless it has an explicit protected exception. #160 treats the payout rule itself as authority and bypasses `o5`. #145's original standing-payment path rechecks `o5`, and its rules-form design amendment proposes doing so for unsealed accounts. [#34 policy requirements](https://github.com/IDNI/tau-testnet/issues/34) · [#160 payout authority](https://github.com/IDNI/tau-testnet/blob/cb8c42d2d999c913be3e48aaf5504dd5e7dfd17c/docs/payout_rules.md#L857-L879) · [#145 design amendment](https://github.com/IDNI/tau-testnet/blob/a060e5a472502df8ec5faddc90c1f7e760db7bc0/docs/programmable_payments_amendment.md)
- **Grant clocks.** #160 has a fixed deadline that triggers automatically, and its native tests separately exercise rule-install and last-paid clocks for holds and explicit idle settlement. Those tests do not prove a hold anchored to a later funding arrival or an automatic inactivity trigger that follows the last draw. The feature-specific allowance in #145 implements those clocks. #145's amendment proposes persistent rule memory for funding and periodic block-clock evaluation for inactivity. [#160 clock examples](https://github.com/IDNI/tau-testnet/blob/cb8c42d2d999c913be3e48aaf5504dd5e7dfd17c/docs/payout_rules.md#L419-L474) · [#145 design amendment](https://github.com/IDNI/tau-testnet/blob/a060e5a472502df8ec5faddc90c1f7e760db7bc0/docs/programmable_payments_amendment.md)
- **Fees, periods and precedence.** #160 uses the executor's fee for explicit runs, the original fee for signed transfers eligible to route, and fee-free clock payouts. In #167, standing execution can charge against a signed fee limit. Calendar days and months use UTC boundaries, monthly schedules clamp the original day to the end of shorter months, and a grant's week can be anchored to an agreed start date. Accepted settlements increment the period total once. The rule or grant guard rejects a spend at the refund deadline, so the refund takes precedence over available period allowance. The clock is the applying block timestamp. [#160 payout timing and fees](https://github.com/IDNI/tau-testnet/blob/cb8c42d2d999c913be3e48aaf5504dd5e7dfd17c/docs/payout_rules.md#L418-L490) · [#167 design](https://github.com/IDNI/tau-testnet/blob/7dea6b3d9d81ec87287fd975d639a29afbe1eccd/docs/programmable_payments.md) · [#62 attested inputs](https://github.com/IDNI/tau-testnet/issues/62)

## Test results

| Where | Revision and scope | Result |
| --- | --- | --- |
| Local | #160 at `cb8c42d2d999c913be3e48aaf5504dd5e7dfd17c`, published native Tau v0.8.5 wheel, five payout suites | 78 passed, 0 failed, 0 skipped |
| Local | #178 at `47a110e0a489b3e3c177941b16316b36a9b82a57`, same native wheel, payout suite | 21 passed, 0 failed, 0 skipped |
| Local | #176 at `2c6b34693f7d809bb4ddf5208058012fded8182e`, five grant, time, standing, routing and payout unit files | 63 passed, 0 failed, 0 skipped |
| Candidate author's GitHub CI | #176 at `2c6b34693f7d809bb4ddf5208058012fded8182e` | Mock suite 4,652 passed, 284 skipped, 2 deselected. Native Bounty 01 step 7 passed, including the [signed two-node grant](https://github.com/jr11ikhsan/tau-testnet/actions/runs/44947672268/job/136978461558). |
| Candidate author's GitHub CI | #177 at `70bd5a9db2da7dc96fa249b853528d9b4b4800ae` | Mock suite 4,585 passed, 283 skipped, 2 deselected. Real-Tau suite 120 passed, 9 skipped, 2 deselected. [GitHub run](https://github.com/forkproven/tau-testnet/actions/runs/45127647224) |
| Upstream GitHub CI | #160 at `cb8c42d2d999c913be3e48aaf5504dd5e7dfd17c` | [Mock suite](https://github.com/IDNI/tau-testnet/actions/runs/45256110114/job/137978240235) 5,099 passed, 361 skipped, 2 deselected. The [native runner](https://github.com/IDNI/tau-testnet/actions/runs/45256110114/job/137978239637) received a shutdown signal during its image build, before pytest. |

The local native tests used real Tau evaluation and the payout settlement helpers with synthetic public identifiers. The Alice/Bob assertions cover an early rejection, three 500-unit weekly draws, rejection of an extra unit in the same week, a 3,500-unit refund and no second refund. The #176 CI run is separate signed two-node evidence. The local audit used no private keys, ran no signed node demo on the public testnet, and did not rerun the full repository suite.

### Reproduce the local runs

From an upstream checkout, fetch the PR heads into named refs:

```sh
git fetch origin pull/160/head:refs/remotes/review/160 \
  pull/176/head:refs/remotes/review/176 pull/178/head:refs/remotes/review/178
```

For #160 and #178, use Python 3.14 on Linux x86-64 with glibc 2.39 or newer. Debian trixie with `libcurl4t64` and `libgmp10` installed was used for this run. Install the published [Tau v0.8.5 wheel](https://github.com/IDNI/tau-lang/releases/tag/v0.8.5) and dependencies inside the environment:

```sh
python -m pip install \
  https://github.com/IDNI/tau-lang/releases/download/v0.8.5/tau_nanobind-0.8.5-cp314-cp314-manylinux_2_39_x86_64.whl \
  pytest==9.1.1 pytest-trio==0.8.0 pytest-asyncio==1.4.0 multiaddr==0.2.0 \
  py-ecc==8.0.0 py_arkworks_bls12381==0.5.0 trio==0.34.0 \
  sortedcontainers==2.4.0 blake3==1.0.11

git switch --detach cb8c42d2d999c913be3e48aaf5504dd5e7dfd17c
python -c 'import tau_native; print(tau_native.load_tau_module())'
python -m pytest -p no:cacheprovider -q -rs tests/test_payout_native.py \
  tests/test_payout_conditions_native.py tests/test_payout_options_native.py \
  tests/test_payout_triggers_native.py tests/test_payout_places_native.py

git switch --detach 47a110e0a489b3e3c177941b16316b36a9b82a57
python -m pytest -p no:cacheprovider -q -rs tests/test_payout_native.py

git switch --detach 2c6b34693f7d809bb4ddf5208058012fded8182e
python -m pytest -q tests/test_grants_bounty01.py tests/test_grants_time_bounty01.py \
  tests/test_standing_payments_bounty01.py tests/test_payout_rules_bounty01.py \
  tests/test_receive_routes_bounty01.py
```
