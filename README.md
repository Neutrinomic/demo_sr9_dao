# Sector9 DAO Demo

This is a standalone DAO demo written in Sector9.

## Executive Summary

This DAO holds real ICRC governance tokens in the DAO canister account and
tracks each user's deposited balance locally. Users deposit through ICRC-2,
stake deposited tokens, wait 7 days for voting power to mature, and then create
or vote on governance proposals. Up to 32 proposals can be created in this
bounded demo, and they can be open or passed at the same time.
Creating a proposal reserves a bond equal to the current proposal threshold from
the proposer's active stake; failed proposals slash that bond from local DAO
accounting, and passed proposals return it when executed. Voting uses mature
active stake, and stake used to vote remains locked from unstaking while any
voted proposal is open.

Withdrawals send real tokens back through ICRC-1. The DAO moves `amount + fee`
into pending withdrawal state before the ledger transfer, finalizes on success,
refunds only deterministic non-execution errors, and leaves ambiguous outcomes
pending for retry or reconciliation. A separate `WithdrawalOps` module stores
the retry operation data so the same memo and `created_at_time` can be
resubmitted.

The checked core preserves local accounting conservation, stake and unstake
movement, voting-lock behavior, proposal deadlines, config bounds, bounded
proposal storage, and withdrawal retry bookkeeping. Scalar proposal lifecycle
rules, the DEX-style multi-proposal book, the DAO accounting module, and the
withdrawal-operation module verify independently. The
remaining production assumptions are operational: the configured governance
ledger is trusted, users deposit through the DAO flow, direct token transfers
are unsupported, and deployment/upgrade/reconciliation procedures are outside
this demo.

## How It Works

`DaoActorDemo` is deployed with:

- `governanceLedger`
- `initialQuorumVotes`
- `initialProposalThreshold`

There is no initial token allocation. The DAO starts with zero accounted tokens.
Users bring real governance tokens into the DAO with `deposit(amount)`.

Deposits use ICRC-2:

- The user first approves the DAO canister on the `governanceLedger`.
- `deposit(amount)` calls `icrc2_transfer_from`.
- Each deposit call includes a DAO operation memo and `created_at_time`.
- Tokens are pulled from the caller's default account into the DAO canister's
  default account.
- The caller's DAO liquid balance is credited only after the ledger returns
  `#Ok(txIndex)`.
- A duplicate ledger response is not credited without a matching local deposit
  operation record.
- Ledger errors or rejects do not credit local DAO balance.

Withdrawals use ICRC-1:

- `withdraw(amount)` can spend only liquid DAO balance.
- The actor queries `icrc1_fee()`.
- The DAO debits `amount + fee` into a pending withdrawal before the transfer.
- The in-flight withdrawal call includes a DAO operation memo and
  `created_at_time`.
- The actor calls `icrc1_transfer` to the caller's default account.
- On success, the pending debit is finalized.
- A duplicate ledger response is treated as success using the duplicate
  transaction index.
- Ledger errors that prove the transfer did not execute restore the full
  pending debit to liquid.
- Ambiguous ledger errors or call rejects leave the debit pending and return a
  reconciliation-required error instead of refunding locally.
- `WithdrawalOps` stores the withdrawal operation id, memo timestamp, amount,
  fee, and debit separately from the main DAO state.
- `pending_withdrawal(user)` exposes the pending operation, and
  `retry_withdrawal()` resubmits the same memo and `created_at_time`.
- A user can have only one pending withdrawal operation at a time.

Voting power still requires staking:

- `stake(amount)` moves liquid deposited tokens into active stake.
- Staked tokens cannot vote immediately. `stake(amount)` sets the caller's
  `votingPowerUnlockAt` to `now + 7 days`.
- `create_proposal(action)` and `vote(id, choice)` both require the caller's
  active stake lock to be mature.
- `voting_power(user)` returns mature active stake and reports `0` while the
  7-day voting lock is active; `stake_info(user)` includes
  `votingPowerUnlockAt`.
- `request_unstake(amount)` removes voting power immediately and starts the
  7-day cooldown, except stake already used to vote stays locked while any
  voted proposal is open.
- `claim_unstaked()` moves matured pending unstake back to liquid balance.
- Only claimed liquid balance can be withdrawn.

The 7-day cooldown is:

```sector9
604_800_000_000_000
```

Quorum and proposal thresholds are absolute token amounts. Initial zero values
are normalized to `1`, and governance config actions must keep both values
nonzero and no larger than the DAO's current accounted token supply.

Proposals have a fixed 3-day voting period. `create_proposal(action)` stores
the creation time and deadline, `vote(id, choice)` rejects once that deadline
has been reached, and `close(id)` rejects attempts before the deadline. A
proposal passes only when yes votes exceed no votes, configured quorum is met,
and participation is strictly more than 3% of the proposal's snapshot active
stake. Otherwise close marks it failed and burns the reserved bond locally.

## Public Actor API

`DaoActorDemo.sr9` exposes:

- `governance_ledger()`
- `proposal_config()`
- `dao_totals()`
- `next_proposal_id()`
- `max_proposals()`
- `proposal_window()`
- `voting_power(user)`
- `stake_info(user)`
- `pending_withdrawal(user)`
- `proposal(id)`
- `vote_info(id, user)`
- `deposit(amount)`
- `withdraw(amount)`
- `retry_withdrawal()`
- `stake(amount)`
- `request_unstake(amount)`
- `claim_unstaked()`
- `create_proposal(action)`
- `vote(id, choice)`
- `close(id)`
- `execute(id)`

State-changing methods delegate to `lib/Dao.sr9`; scalar proposal lifecycle
rules live in `lib/Proposal.sr9`, bounded multi-proposal storage lives in
`lib/ProposalBook.sr9`, and withdrawal recovery delegates to
`lib/WithdrawalOps.sr9`. The proposal book follows the DEX registry pattern:
`Dao.State` owns an immutable opaque proposal-book handle whose internals mutate
through verified module functions.

## What Was Verified And Proven

The DAO state is opaque. Its public model tracks:

- governance ledger
- local ledger-backed accounted token total
- liquid balance total
- active stake total
- pending unstake total
- pending withdrawal total
- reserved proposal-bond total
- next proposal id
- quorum and proposal-threshold config

The main accounting invariant is:

```text
totalLiquid + totalActiveStake + totalPendingUnstake + totalPendingWithdraw
  + totalProposalBonds == totalSupply
```

Here `totalSupply` means the DAO's local ledger-backed accounting total, not
the external ledger's global token supply.

The verified and checked contracts cover:

- initialization starts with zero local token allocation
- withdrawal operation initialization starts with zero pending withdrawal amount
- successful deposits increase local accounted tokens and liquid balance
- deposit ledger arguments are constructed so `icrc2_transfer_from` pulls from
  the caller's default account into the DAO canister's default account and
  includes a non-null memo and `created_at_time`
- deposit ledger errors/rejects, including duplicate responses, preserve local
  accounting
- withdrawal begin moves `amount + fee` from liquid into pending withdrawal
- withdrawal ledger arguments are constructed so `icrc1_transfer` sends to the
  caller's default account, with no source subaccount, and includes the pending
  withdrawal memo and `created_at_time`
- withdrawal success settles the pending debit and reduces accounted tokens
- deterministic withdrawal errors restore the pending debit to liquid
- ambiguous withdrawal errors/rejects preserve the pending debit instead of
  refunding locally
- withdrawal retry starts are read-only and return the stored operation id,
  amount, fee, debit, and `created_at_time`
- clearing a withdrawal operation decreases the pending withdrawal-operation
  total by exactly that debit
- withdrawal never decreases active stake or pending unstake, so locked voting
  tokens cannot be withdrawn around the 7-day lock
- staking preserves the local accounted token total
- successful staking decreases the caller's liquid balance by exactly `amount`
  and increases active stake by exactly `amount`
- successful staking sets the caller's voting unlock time to exactly
  `now + 7 days`
- eligible voting power is zero before the caller's voting unlock time and
  equals active stake after it
- successful proposal creation and successful voting prove
  `now >= votingPowerUnlockAt`
- unstake requests remove active voting power and move tokens into pending
  unstake
- successful unstake requests set the unlock time to exactly `now + 7 days`
- claims move matured pending unstake back to liquid
- successful claims move exactly the matured pending-unstake amount into liquid
  and clear the caller's pending unstake
- scalar proposal creation stores the creation time and a deadline exactly
  3 days later
- voting rejects at or after the stored proposal deadline
- scalar proposal close enforces the deadline, quorum, yes > no, and >3%
  participation rule
- proposal creation reserves the current proposal-threshold bond from active
  stake into proposal-bond accounting
- failed proposal close burns the reserved bond locally by reducing
  `totalProposalBonds` and `totalSupply`
- passed proposal execution returns the reserved bond to the proposer's active
  stake
- the proposal book has a fixed lifetime capacity of 32 proposals
- multiple proposals can be open or passed at the same time
- successful voting proves the voter had not already voted on that proposal,
  marks that voter/proposal pair as voted, and changes proposal totals by
  exactly the receipt weight once
- successful voting locks the exact vote weight against unstaking while that
  proposal is open; with multiple open proposals, the reusable-stake lock is the
  maximum open voted weight for the user
- successful deposits, withdrawal staging/success, staking, unstaking,
  claiming, and the shared withdrawal-failure restore step preserve every other
  account key's liquid, active-stake, voting-lock, pending-unstake, and
  pending-withdraw state
- voting and close preserve token balances except failed close burns the
  proposal bond; creation and execution move the proposer's stake between active
  stake and proposal-bond accounting
- config validity keeps quorum and proposal-threshold values nonzero
- governance config actions cannot set quorum or proposal-threshold values
  above the DAO's current accounted token supply

`Dao.sr9` keeps rich per-user contracts on private implementation functions and
uses public wrappers with import-safe aggregate contracts. The actor follows the
DEX example's boundary style: deep DAO accounting invariants stay in the DAO
module, while the actor keeps only the lightweight withdrawal-operation
invariant across ledger awaits.

`Types.sr9`, `WithdrawalOps.sr9`, `Proposal.sr9`, `ProposalBook.sr9`, and
`Dao.sr9` verify independently. `proofs/DaoObservers.sr9` verifies the external
observer proof attempts for deposit, withdraw begin/success/reject, staking,
unstaking, claiming, voting, and execution preservation properties.
`DaoActorDemo.sr9` also verifies and compiles to Wasm.

## Verification Commands

The latest published image was pulled before verification:

```bash
docker pull ghcr.io/neutrinomic/sr9:latest
```

The image digest used was:

```text
sha256:f5cef482c5ad738582453f1e7f3a1096bbbf1b7b7e5da947f8f468e48c2df03c
```

Commands used:

```bash
SR9_IMAGE='ghcr.io/neutrinomic/sr9@sha256:f5cef482c5ad738582453f1e7f3a1096bbbf1b7b7e5da947f8f468e48c2df03c'
SR9=(docker run --rm -e XDG_CACHE_HOME=/tmp/sector9 --user "$(id -u):$(id -g)" -v "$PWD:/work" -w /work "$SR9_IMAGE")

"${SR9[@]}" --check lib/Types.sr9 lib/WithdrawalOps.sr9 lib/Proposal.sr9 lib/ProposalBook.sr9 lib/Dao.sr9 proofs/DaoObservers.sr9 DaoActorDemo.sr9
"${SR9[@]}" --verify --deterministic --cores 1 --verify-timeout-ms 600000 lib/Types.sr9
"${SR9[@]}" --verify --deterministic --cores 1 --verify-timeout-ms 900000 lib/WithdrawalOps.sr9
"${SR9[@]}" --verify --deterministic --cores 1 --verify-timeout-ms 600000 lib/Proposal.sr9
"${SR9[@]}" --verify --deterministic --cores 1 --verify-timeout-ms 600000 lib/ProposalBook.sr9
"${SR9[@]}" --verify --deterministic --cores 1 --verify-timeout-ms 600000 lib/Dao.sr9
"${SR9[@]}" --verify --deterministic --cores 1 --verify-timeout-ms 1200000 proofs/DaoObservers.sr9
"${SR9[@]}" --verify --deterministic --cores 1 --verify-timeout-ms 1200000 DaoActorDemo.sr9
```

The actor compiles with:

```bash
"${SR9[@]}" -c DaoActorDemo.sr9 -o /tmp/DaoActorDemo.wasm
```

Source scan:

```bash
rg -n "trusted" . --glob '*.sr9'
```

No trusted Sector9 source was found.

## Current Limits

- One governance ledger per DAO instance.
- Default accounts only; no subaccount selection.
- No token transfers between DAO users.
- Adding more stake resets the caller's active-stake voting unlock time.
- No abstain vote and no vote replacement.
- No early close when an outcome is mathematically final; close is deadline-only.
- Proposal capacity is fixed at 32 lifetime proposals; the 33rd creation
  rejects.
- No execution actions beyond DAO config changes.
