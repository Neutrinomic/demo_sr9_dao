# DAO Security TODO

## Scope

This practical audit reviewed:

- `lib/Dao.sr9`
- `lib/Types.sr9`
- `DaoActorDemo.sr9`
- `README.md`

The review intentionally ignores formal verification claims and focuses on production security, user-funds safety, governance correctness, operational liveness, and user-facing behavior.

## TODO

- [ ] **High: handle ambiguous withdrawal ledger results without refunding.**
  `DaoActorDemo.sr9:179`, `lib/Dao.sr9:783`, `lib/Dao.sr9:851`.
  Any `icrc1_transfer` `#Err` currently restores the pending debit to liquid. Add durable withdrawal operation IDs, memo/timestamp correlation, and a reconciliation-needed state for duplicate or unknown outcomes. Refund only errors that are guaranteed not to have executed.

- [ ] **High: add recovery for stuck pending withdrawals.**
  `DaoActorDemo.sr9:173`, `DaoActorDemo.sr9:179`, `lib/Dao.sr9:697`.
  Pending withdrawals are cleared only by the suspended ledger call returning. Persist full pending withdrawal operation records and add retry, finalize, cancel, or reconciliation flows so users cannot remain stuck forever.

- [ ] **High: prevent immediate proposal close griefing.**
  `DaoActorDemo.sr9:260`, `lib/Dao.sr9:1277`, `lib/Dao.sr9:1303`.
  `close(id)` has no caller restriction and no voting period. Store proposal creation time and voting deadline. Allow close only after the deadline, or only when the outcome is mathematically final.

- [ ] **High: make vote weight safe against stake recycling.**
  `lib/Dao.sr9:1173`, `lib/Dao.sr9:1248`, `lib/Dao.sr9:978`.
  `proposalSnapshotStake` is recorded but not used to compute votes. Use proposal-time snapshots or lock the exact stake used for voting until proposal finalization so the same tokens cannot vote again through another principal while a proposal is open.

- [x] **High: implement real config validation.**
  `lib/Dao.sr9:107`, `lib/Dao.sr9:475`, `lib/Dao.sr9:1357`.
  `configValid` and `actionValid` always return true. Validate constructor config and governance config actions. Reject zero, trivializing, or permanently unreachable quorum and proposal-threshold settings.

- [ ] **Medium: add idempotency and correlation data to ledger calls.**
  `DaoActorDemo.sr9:60`, `DaoActorDemo.sr9:61`, `DaoActorDemo.sr9:83`, `DaoActorDemo.sr9:85`.
  Deposits and withdrawals use `memo = null` and `created_at_time = null`. Create per-operation IDs, persist them where needed, include them in memo/timestamp fields, and handle duplicate ledger responses explicitly.

- [ ] **Medium: prune or bound storage growth.**
  `lib/Dao.sr9:765`, `lib/Dao.sr9:1262`.
  Zero balances and cleared pending-withdrawal fields are written back instead of removed. Vote keys are never cleared. Delete zero entries, prune finalized proposal vote records, and bound or archive proposal-related storage.

- [ ] **Low: make `voting_power` report eligible voting power or rename it.**
  `DaoActorDemo.sr9:112`, `lib/Dao.sr9:1252`.
  `voting_power(user)` returns active stake even when it is still locked. Rename it to `active_stake`, or make it time-aware and return only currently eligible voting power.

## Assumptions

- Users are expected to deposit through the DAO `deposit(amount)` flow. Direct token transfers to the DAO account are unsupported and are outside this security TODO list.
- The deployed `governanceLedger` principal is assumed to be the intended trusted ICRC ledger. This DAO does not plan to validate ledger module hash, token metadata, or supported standards as part of the current hardening work.

## Overall Assessment

The current DAO is coherent as a verified demo, but it is not production-ready for real-token use. The most important hardening work is operational: robust ledger idempotency and reconciliation, safe withdrawal recovery, real proposal deadlines, snapshot or vote-lock semantics, and meaningful governance config validation.
