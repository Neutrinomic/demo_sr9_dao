# Sector9 DAO Demo

This is a standalone verified DAO demo written in Sector9.

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
- Tokens are pulled from the caller's default account into the DAO canister's
  default account.
- The caller's DAO liquid balance is credited only after the ledger returns
  `#Ok(txIndex)`.
- Ledger errors or rejects do not credit local DAO balance.

Withdrawals use ICRC-1:

- `withdraw(amount)` can spend only liquid DAO balance.
- The actor queries `icrc1_fee()`.
- The DAO debits `amount + fee` into a pending withdrawal before the transfer.
- The actor calls `icrc1_transfer` to the caller's default account.
- On success, the pending debit is finalized.
- On ledger error or reject, the full pending debit is restored to liquid.
- A user can have only one pending withdrawal at a time.

Voting power still requires staking:

- `stake(amount)` moves liquid deposited tokens into active stake.
- Staked tokens cannot vote immediately. `stake(amount)` sets the caller's
  `votingPowerUnlockAt` to `now + 7 days`.
- `create_proposal(action)` and `vote(id, choice)` both require the caller's
  active stake lock to be mature.
- `voting_power(user)` returns active stake; `stake_info(user)` includes
  `votingPowerUnlockAt` so callers can tell when active stake becomes eligible.
- `request_unstake(amount)` removes voting power immediately and starts the
  7-day cooldown.
- `claim_unstaked()` moves matured pending unstake back to liquid balance.
- Only claimed liquid balance can be withdrawn.

The 7-day cooldown is:

```sector9
604_800_000_000_000
```

Quorum and proposal thresholds are absolute token amounts. Initial zero values
are normalized to `1`, and governance config actions must keep both values
nonzero and no larger than the DAO's current accounted token supply.

Proposals have a fixed 7-day voting period. `create_proposal(action)` stores
the creation time and deadline, and `close(id)` rejects attempts before the
deadline.

## Public Actor API

`DaoActorDemo.sr9` exposes:

- `governance_ledger()`
- `proposal_config()`
- `voting_power(user)`
- `stake_info(user)`
- `proposal(id)`
- `deposit(amount)`
- `withdraw(amount)`
- `stake(amount)`
- `request_unstake(amount)`
- `claim_unstaked()`
- `create_proposal(action)`
- `vote(id, choice)`
- `close(id)`
- `execute(id)`

All state-changing methods delegate to verified transitions in `lib/Dao.sr9`.

## What Was Verified And Proven

The DAO state is opaque. Its public model tracks:

- governance ledger
- local ledger-backed accounted token total
- liquid balance total
- active stake total
- pending unstake total
- pending withdrawal total
- next proposal id
- quorum and proposal-threshold config

The main accounting invariant is:

```text
totalLiquid + totalActiveStake + totalPendingUnstake + totalPendingWithdraw
  == totalSupply
```

Here `totalSupply` means the DAO's local ledger-backed accounting total, not
the external ledger's global token supply.

The verified contracts prove:

- initialization starts with zero local token allocation
- successful deposits increase local accounted tokens and liquid balance
- deposit ledger arguments are constructed so `icrc2_transfer_from` pulls from
  the caller's default account into the DAO canister's default account
- deposit ledger errors/rejects preserve local accounting
- withdrawal begin moves `amount + fee` from liquid into pending withdrawal
- withdrawal ledger arguments are constructed so `icrc1_transfer` sends to the
  caller's default account, with no source subaccount
- withdrawal success settles the pending debit and reduces accounted tokens
- withdrawal error/reject restores the pending debit to liquid
- withdrawal never decreases active stake or pending unstake, so locked voting
  tokens cannot be withdrawn around the 7-day lock
- after a withdrawal has been debited into pending state, a ledger error or
  reject restores the exact pending debit to the caller's liquid balance
- staking preserves the local accounted token total
- successful staking decreases the caller's liquid balance by exactly `amount`
  and increases active stake by exactly `amount`
- successful staking sets the caller's voting unlock time to exactly
  `now + 7 days`
- successful proposal creation and successful voting prove
  `now >= votingPowerUnlockAt`
- unstake requests remove active voting power and move tokens into pending
  unstake
- successful unstake requests set the unlock time to exactly `now + 7 days`
- claims move matured pending unstake back to liquid
- successful claims move exactly the matured pending-unstake amount into liquid
  and clear the caller's pending unstake
- voting and proposal lifecycle transitions preserve token accounting
- successful proposal creation stores the creation time and a deadline exactly
  7 days later
- successful voting proves the voter had not already voted on that proposal,
  marks that voter/proposal pair as voted, and changes proposal totals by
  exactly the receipt weight once
- successful deposits, withdrawal staging/success, staking, unstaking,
  claiming, and the shared withdrawal-failure restore step preserve every other
  account key's liquid, active-stake, voting-lock, pending-unstake, and
  pending-withdraw state
- proposal creation, voting, closing, and execution preserve every account key's
  liquid, active-stake, pending-unstake, and pending-withdraw amounts
- config validity keeps quorum and proposal-threshold values nonzero
- governance config actions cannot set quorum or proposal-threshold values
  above the DAO's current accounted token supply

`Dao.sr9` keeps rich per-user contracts on private implementation functions and
uses public wrappers with import-safe aggregate contracts. This avoids a current
Sector9 limitation where external modules cannot unfold mutable maps inside an
opaque DAO state through public pure accessors.

The persistent actor proves `Dao.supplyBalanced(dao)` and `Dao.configValid(dao)`
across public methods and across ledger awaits.

`proofs/DaoObservers.sr9` adds external observer proofs for deposit,
withdraw begin/success/reject, staking, unstaking, claiming, voting, and
execution preservation properties.

## Verification Commands

The latest published image was pulled before verification:

```bash
docker pull ghcr.io/neutrinomic/sr9:latest
```

The image digest used was:

```text
sha256:f5cef482c5ad738582453f1e7f3a1096bbbf1b7b7e5da947f8f468e48c2df03c
```

Verification succeeded with:

```bash
SR9_IMAGE='ghcr.io/neutrinomic/sr9@sha256:f5cef482c5ad738582453f1e7f3a1096bbbf1b7b7e5da947f8f468e48c2df03c'
SR9=(docker run --rm -e XDG_CACHE_HOME=/tmp/sector9 --user "$(id -u):$(id -g)" -v "$PWD:/work" -w /work "$SR9_IMAGE")

"${SR9[@]}" --verify --deterministic --cores 1 --verify-timeout-ms 600000 lib/Types.sr9
"${SR9[@]}" --verify --deterministic --cores 2 --verify-timeout-ms 1200000 lib/Dao.sr9
"${SR9[@]}" --verify --deterministic --cores 1 --verify-timeout-ms 700000 proofs/DaoObservers.sr9
"${SR9[@]}" --verify --deterministic --cores 2 --verify-timeout-ms 1200000 DaoActorDemo.sr9
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
- No proposal archive beyond the current proposal slot.
- No execution actions beyond DAO config changes.
