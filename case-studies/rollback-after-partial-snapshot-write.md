# Closing a failed transaction before the next caller uses it

A snapshot replacement began a SQLite transaction and applied several writes, but a later failure could leave that transaction open. The next operation sharing the connection could then run inside the failed replacement's transaction. The July correction added rollback before propagating the failure, preserving the old snapshot and returning the connection to a clean boundary.

The reviewed correction belongs to the historical Python backend; the current application uses Rust.

## An exception is not a transaction boundary

The replacement operation had a sensible happy path: prepare valid rows, begin a transaction, insert or update each row, delete rows absent from the new snapshot and commit. Its missing path was what happened after BEGIN if the loop or a later statement raised an exception.

Consider an existing snapshot containing record A. A replacement contains valid record B followed by record C with an unsupported value. B can be written successfully inside the open transaction, while binding C raises an error before the operation reaches cleanup or commit. Returning that error to the caller does not, by itself, close the transaction holding B's change.

That creates a connection-lifecycle problem beyond the failed request. If a later caller uses the same connection, its work can share the unfinished transaction. A later commit could make earlier partial writes durable, while a later rollback could discard unrelated work. The request boundary and database transaction boundary have diverged.

Python exposes these as explicit connection operations: rollback reverses an open transaction, and `in_transaction` reports whether a transaction is active. Raising a Python exception is not a replacement for invoking the rollback owner. [Python SQLite connection documentation](https://docs.python.org/3.13/library/sqlite3.html#sqlite3.Connection.rollback).

## What the correction changed

The patch kept the explicit BEGIN and wrapped the write loop, stale-row deletion and COMMIT in one failure path. If an exception escaped that body, the code rolled back and then re-raised it. It did not convert failure into a success response, retry the operation or manufacture a replacement snapshot.

```text
replace snapshot:
    prepare rows and the identifiers to retain
    begin transaction

    try:
        upsert each prepared row
        remove rows absent from the replacement
        commit
    on escaping failure:
        roll back the open transaction
        propagate the original failure
```

The rollback belongs after BEGIN has succeeded and before the operation releases its failure to the next caller. A validation failure before starting a transaction has a different cleanup requirement. This distinction avoids treating every error as if it had acquired database state.

The catch covered escaping `BaseException` values in the historical implementation. Its purpose was cleanup, not suppression: the original exception continued outward after rollback. The existing regression exercised an ordinary binding failure; it did not demonstrate every interruption path covered by that broader catch.

## Why native statement behavior is insufficient

SQLite can cancel a failing statement while leaving earlier statements in the same transaction intact. Its ABORT conflict behavior illustrates that distinction: the statement fails without necessarily rolling back the transaction's previous changes. The reviewed regression failed during Python parameter binding, so it was not a demonstration of SQLite's ABORT conflict algorithm; both mechanisms nevertheless show why the application must own the whole replacement's failure boundary. [SQLite conflict-resolution documentation](https://www.sqlite.org/lang_conflict.html).

SQLite's transaction documentation also distinguishes statement completion, explicit commit and rollback, and error responses that do not uniformly roll back an entire transaction. A caller cannot infer a clean connection merely from receiving an error. The operational question is whether the transaction owner has resolved the unit it began. [SQLite transaction error handling](https://www.sqlite.org/lang_transaction.html#response_to_errors_within_a_transaction).

This is an application-integrity issue rather than evidence of a malicious caller reaching another user's records. It matters because stored state can stop representing either the old snapshot or a completed new snapshot, and subsequent operations can inherit an unfinished unit without asking to participate in it.

## The existing failure-injection check

The corrective commit added a focused synthetic regression. It first stored one valid snapshot. It then requested a replacement with a valid first row and a later value represented by an unsupported Python object. That object could not be bound as the database value, causing the operation to fail after entering the write path.

The test required an exception, checked that the connection was no longer in a transaction, and compared the remaining stored rows with the original snapshot. It therefore exercised more than early request validation or a happy-path update.

| Observation | What it establishes |
|---|---|
| A valid row precedes the failing value | Failure occurs after the operation has entered its multi-write path. |
| The exception still reaches the test | Cleanup does not silently convert failure into success. |
| `in_transaction` is false | The failed operation leaves no open transaction on that connection. |
| Stored rows equal the previous snapshot | Partial replacement did not survive the rollback. |

The exception assertion was broad; it did not pin a particular error class or message. The regression did not execute a second writer, force a disk failure, test a failed COMMIT or establish durability after a process crash. It verified the demonstrated mid-write failure and two essential invariants: old state survives, and the connection is no longer inside the failed unit.

## Distinct from concurrency ownership

Rollback on failure fixes transaction cleanup, but it does not by itself serialize all requests using a shared connection. An operation-wide lock and a policy for transaction ownership are separate concerns. The later [atomic-merge correction](atomic-merge-integrity.md) illustrates that additional boundary; it should not be retroactively credited to this July patch.

The reusable lesson is to review an operation's exit states, not just whether it starts a transaction. Successful completion must commit the complete replacement. Failure after writes begin must preserve the prior state and finish the transaction before the connection is reused. Source and the existing synthetic regression were inspected for this retrospective; no tests, database operations or live service checks were run.
