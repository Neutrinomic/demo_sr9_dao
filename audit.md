# DAO Security Audit

## Scope

This practical audit reviewed:

- `lib/Dao.sr9`
- `lib/Types.sr9`
- `DaoActorDemo.sr9`
- `README.md`

The review intentionally ignores formal verification claims and focuses on production security, user-funds safety, governance correctness, operational liveness, and user-facing behavior.

## Findings

### High: withdrawal errors can refund after ambiguous ledger execution

References:

- `DaoActorDemo.sr9:179`
- `lib/Dao.sr9:783`
- `lib/Dao.sr9:851`

Impact:

Any `icrc1_transfer` `#Err` is routed to `finishWithdrawErr`, which restores the pending debit to liquid. Some ledger errors can be ambiguous, such as duplicate or unknown execution states. If the external ledger already executed the transfer, the user could receive the external payout and also recover local liquid balance.

Recommended fix:

Add durable withdrawal operation IDs, include memo and timestamp correlation data in ledger calls, and treat ambiguous results as reconciliation-needed instead of refundable. Only restore local balance for errors that are guaranteed not to have executed.

### High: pending withdrawals can get permanently stuck

References:

- `DaoActorDemo.sr9:173`
- `DaoActorDemo.sr9:179`
- `lib/Dao.sr9:697`

Impact:

The DAO stages `amount + fee` into pending withdrawal state before awaiting the ledger transfer. The only clearing paths are the suspended call returning success, error, or reject. If a callback is lost, the ledger does not respond, or an upgrade interrupts the flow, the user can remain stuck with `#withdrawInProgress` and no recovery path.

Recommended fix:

Persist full pending withdrawal operation records with recipient, fee, nonce or memo, created-at timestamp, status, and timestamps. Add public or admin-controlled retry, finalize, cancel, and reconciliation flows.

### High: proposals can be closed immediately by anyone

References:

- `DaoActorDemo.sr9:260`
- `lib/Dao.sr9:1277`
- `lib/Dao.sr9:1303`

Impact:

`close(id)` has no caller restriction and no voting period. Any caller can close a newly opened proposal before voters have time to react, causing it to fail unless quorum and yes-majority already exist. A large holder can also vote and close before opposition can respond.

Recommended fix:

Store proposal creation time and voting deadline. Allow close only after the deadline, or allow early close only when the outcome is mathematically final.

### High: vote weight uses current stake, not proposal snapshot stake

References:

- `lib/Dao.sr9:1173`
- `lib/Dao.sr9:1248`
- `lib/Dao.sr9:978`

Impact:

`proposalSnapshotStake` is recorded but not used to compute votes. Voting weight is based on current active stake at vote time. A voter can vote, later unstake and withdraw after cooldown, transfer tokens, restake under another principal, and vote again on the same still-open proposal.

Recommended fix:

Use proposal-time voting snapshots, or lock the exact stake used for voting until proposal finalization. Prevent voted stake from being recycled into another principal's vote while the proposal is open.

### High: config validation is effectively disabled

References:

- `lib/Dao.sr9:107`
- `lib/Dao.sr9:475`
- `lib/Dao.sr9:1357`

Impact:

`configValid` and `actionValid` always return true. Governance can set quorum and proposal threshold to zero, or to unreachable values that permanently break governance. Initial constructor values are also unchecked.

Recommended fix:

Validate constructor config and governance config actions. Require sane nonzero bounds and reject actions that make governance trivial or permanently unusable.

### Medium: ledger calls lack idempotency and correlation data

References:

- `DaoActorDemo.sr9:60`
- `DaoActorDemo.sr9:61`
- `DaoActorDemo.sr9:83`
- `DaoActorDemo.sr9:85`

Impact:

Deposits and withdrawals use `memo = null` and `created_at_time = null`. Frontend retries, duplicate responses, callback failures, and later reconciliation cannot be tied to a specific DAO operation. For deposits, a duplicate transfer-from result may represent a prior successful transfer that is not credited locally.

Recommended fix:

Create per-operation IDs, persist them where needed, and include them in memo and timestamp fields. Add duplicate and reconciliation handling for deposit and withdrawal flows.

### Medium: direct token transfers to the DAO are unclaimable

References:

- `DaoActorDemo.sr9:130`
- `lib/Dao.sr9:559`

Impact:

Only the `deposit(amount)` transfer-from path credits local DAO balances. If a user sends ICRC-1 tokens directly to the DAO account, the DAO has no notify, reconcile, or rescue path to credit or return those tokens.

Recommended fix:

Add a deposit-by-transaction or notify flow, ledger balance reconciliation, or a governed rescue path for unmatched inbound transfers.

### Medium: ledger principal is blindly trusted

References:

- `DaoActorDemo.sr9:7`
- `DaoActorDemo.sr9:142`

Impact:

The actor accepts `governanceLedger` at deployment and casts it with `ICRCLedger.fromPrincipal`. There is no validation of module hash, ledger metadata, supported standards, token identity, or expected canister ID. A wrong deployment argument can make the DAO account worthless balances or interact with a nonstandard ledger.

Recommended fix:

Pin the intended ledger canister in deployment configuration. Validate ledger metadata and supported ICRC standards at initialization or first use, and document the trust assumption.

### Medium: storage grows without pruning

References:

- `lib/Dao.sr9:765`
- `lib/Dao.sr9:1262`

Impact:

Zero balances and cleared pending-withdrawal fields are written back instead of removed. Vote keys are never cleared, while old proposal details are overwritten by the single proposal slot. Over time this can become a storage and cycle-cost DoS vector.

Recommended fix:

Delete zero entries instead of storing them, prune finalized proposal vote records, and either bound or archive proposal-related storage.

### Low: `voting_power` reports active stake, not eligible voting power

References:

- `DaoActorDemo.sr9:112`
- `lib/Dao.sr9:1252`

Impact:

`voting_power(user)` returns active stake even when it is still locked and cannot vote yet. Clients may display locked stake as usable voting power.

Recommended fix:

Rename the method to `active_stake`, or make `voting_power` time-aware and return only currently eligible voting power.

## Overall Assessment

The current DAO is coherent as a verified demo, but it is not production-ready for real-token use. The most important hardening work is operational, not cosmetic: robust ledger idempotency and reconciliation, safe withdrawal recovery, real proposal deadlines, snapshot or vote-lock semantics, and meaningful governance config validation.
