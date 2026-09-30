# Preventing destructive self-merges and partial updates

A merge operation in a personal application could delete the record it was supposed to preserve when the source and destination were the same. A second failure mode left partial changes when deleting a source record failed after its associations had already been copied.

The September 2026 review corrected both in the application's historical Python/FastAPI/SQLite backend. This was an integrity and transaction-boundary defect: an allowed edit could destroy or inconsistently rearrange application history.

## How the failure happened

A chain groups associated records and notes. The merge operation transfers the source's contents to a destination, removes the source and marks the destination as manually edited.

The original implementation did not reject identical identifiers. With source equal to destination, the cleanup step deleted that same chain and its membership. The bug required no unusual input shape; the two identifiers merely had to be equal.

For distinct identifiers, the operation used helpers that committed individually. Associations could be copied before source deletion reached a foreign-key restriction involving notes. The exception stopped later work but could not undo an earlier commit.

| Input or failure | Before | After |
|---|---|---|
| Source and destination are identical | Cleanup could delete the selected chain and its membership. | API rejects with HTTP 422; direct store invocation also rejects before writes. |
| Source contains associations and notes | Notes were not transferred before deleting the source; deletion could fail. | Both associations and notes move within the same transaction. |
| Source deletion fails after earlier writes | Committed intermediate changes could remain. | The entire merge rolls back, including transferred contents and provenance changes. |
| Another request uses the shared connection during a merge | Per-call serialization did not protect the complete operation from interleaving. | One connection lock spans the complete write unit and its rollback. |

## Patch pattern

The API guard provides a clear client error. The store guard protects the invariant for callers that bypass the HTTP route. Putting the condition in both places serves different boundaries; a disabled browser button would not protect either one.

The transaction also had to include every change. Calling a helper that commits from inside a larger transaction would still break atomicity, so the corrected merge used direct database operations under the transaction owner.

This pseudocode expresses the pattern without reproducing the private source:

```text
HTTP merge request:
    if source_id == destination_id:
        reject with 422
    call store.merge(destination_id, source_id)

store.merge(destination_id, source_id):
    if source_id == destination_id:
        reject invalid merge before writing

    hold shared connection lock:
        run one transaction:
            require both records to exist
            copy source associations to destination without duplicates
            reassign source notes to destination
            remove source associations
            delete source record
            mark destination as manually edited
        commit all changes together, or roll back all on failure
```

The reusable principle is to define the edit as one state transition. Its invariant is stronger than “the endpoint returned an error”: rejection or failure must preserve the prior database state.

## Why database safeguards alone were insufficient

SQLite distinguishes a failed statement from a failed application operation. An immediate foreign-key violation reverts the statement that violated it; that does not undo earlier committed association copies. The historical schema enabled foreign keys, but the merge still needed an application-wide transaction. This is the mechanism behind the observed defect, interpreted using [SQLite's foreign-key documentation](https://www.sqlite.org/foreignkeys.html#fk_deferred).

Python's connection context manager commits an open transaction on normal exit and rolls it back when an uncaught exception exits the body. It does not itself open a transaction or close the connection. The reviewed implementation used ordinary implicit-write transaction handling and held its connection lock around the context; the first write began the unit that the context later committed or rolled back. This distinction matters when adapting the pattern to autocommit settings or another connection owner. [Python `sqlite3` context-manager documentation](https://docs.python.org/3/library/sqlite3.html#how-to-use-the-connection-context-manager).

SQLite's single-writer rule applies to database connections; it does not decide which application request owns a shared connection's multi-statement operation. The lesson inferred from this patch is to keep the application lock across the full unit, including rollback, rather than just individual statements. [SQLite transaction documentation](https://www.sqlite.org/lang_transaction.html).

These primary references explain database behavior. They do not establish this application's original failure or test results; that evidence comes from the dated corrective patch and its existing synthetic checks.

## Existing verification evidence

The corrective commit added three synthetic checks:

1. **Self-merge rejection:** direct store invocation raises an error; the HTTP route returns 422; the chain still exists afterward.
2. **Successful transfer:** the source is removed, its association and note belong to the destination, and SQLite's foreign-key check reports no violation.
3. **Failure after writes begin:** a synthetic trigger aborts source deletion. The full database dump after failure equals the dump captured before the merge, and the connection has no open transaction.

The third check is particularly useful: it targets the destructive midpoint rather than testing only a successful operation or an early input rejection. It would reveal copied associations, moved notes or altered metadata surviving a failed merge.

The historical reassessment records passing final checks for the Python backend. These results belong to that September 2026 correction; no tests were rerun while preparing this case study.

The corrected implementation is the retired Python backend; the current application uses Rust.
