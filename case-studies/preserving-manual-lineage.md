# Preserving manual ownership during an automatic rebuild

An automatic rebuild correctly preserved manually edited groups but still deleted the relationship edges belonging to their records. The top-level object survived while part of the user's edit disappeared. The June correction narrowed deletion to the automatic subsystem's ownership and strengthened a regression that could previously pass without exercising competing automatic output.

The reviewed correction belongs to the historical Python backend; the current application uses Rust.

## Object ownership was not enough

The application represented groups, their record membership and directed lineage edges in separate tables. A lineage edge connected a child record to a parent record. Groups could be automatically derived or manually owned; a rebuild was supposed to replace automatic derivations while preserving manual decisions.

The original teardown already selected automatically owned groups before deleting their membership and group rows. Its separate lineage cleanup, however, deleted every edge. That made the ownership boundary inconsistent across related data: group rows were scoped, but their relationship data was treated as globally disposable.

This is an integrity boundary between an automatic process and explicit user edits. It is not evidence of one account accessing another account's records. The failure arose because the rebuild claimed broader mutation authority than its task required, even when every database reference remained structurally valid.

SQLite's DELETE semantics make the distinction concrete: a WHERE expression selects which rows are removed, while an unrestricted DELETE removes the table's contents. A correctly scoped first deletion does not constrain a later deletion in another table. [SQLite DELETE documentation](https://www.sqlite.org/lang_delete.html).

## Preserving an edge if either endpoint is protected

The corrected lineage deletion selected edges only when neither endpoint belonged to a manually owned group. Let `M` be the set of record identifiers belonging to those groups. An edge could be deleted only if both its child and its parent were outside `M`.

| Child belongs to a manual group | Parent belongs to a manual group | Cleanup decision |
|---|---|---|
| No | No | Eligible for deletion and regeneration. |
| Yes | No | Preserve the edge. |
| No | Yes | Preserve the edge. |
| Yes | Yes | Preserve the edge. |

The conjunction is important. Using “child is unprotected **or** parent is unprotected” would still delete the two mixed cases. Those edges touch protected user-owned data even though their other endpoint is outside a manual group.

The patch did not add a new per-edge owner field. It derived protection through current group membership. That is a precise, narrower boundary with a dependency: membership and the group's provenance label must correctly represent the protected records. It does not independently prove that every possible future ownership model is covered.

## The complete narrow rebuild pattern

The public pattern below expresses the historical operation without copying its database statements:

```text
automatic rebuild:
    remove membership and group rows owned by automatic generation
    protected = record identifiers in remaining manually owned groups

    delete a lineage edge only when:
        its child is outside protected
        AND its parent is outside protected

    candidate_records = records not already assigned to a surviving group
    derive automatic groups and edges from candidate_records
    persist a derived edge only if both endpoints are candidate records
    assign records only from that candidate set
```

The boundary applies during teardown and reconstruction. Protecting deletion but feeding manual records back into the automatic derivation could still create competing membership. Conversely, filtering reconstruction would not recover a manual edge already erased by cleanup. The reviewed function addressed those phases separately.

The historical query used exclusion sets. When transferring this pattern to another schema, SQL's null behavior needs attention: `NOT IN` can yield NULL rather than true when its operands include NULL. Treat the example as an ownership predicate to implement against the actual key contract, not a query to transplant without considering schema semantics. [SQLite IN and NOT IN semantics](https://www.sqlite.org/lang_expr.html#the_in_and_not_in_operators).

## Making the regression do real work

The existing manual-preservation test was weakened by an escape condition. It could succeed when the rebuild assigned no records, or when all inspected automatic output excluded the manual record. With no automatic output to inspect, the negative membership check did not establish that protection worked while the builder was active.

Python's `all` returns true for an empty iterable. That is appropriate language behavior, but an exclusion assertion over an empty result can become a vacuous regression. [Python `all` documentation](https://docs.python.org/3/library/functions.html#all).

The correction created two protected manual groups and a separate free pair that the rebuild could actually group. It then required positive automatic assignment and a nonempty automatic-group collection before checking exclusion. Finally, it asserted that both manual groups retained their manual provenance and records, and that neither protected record appeared in any generated group.

This checks two independent things: the automatic process ran, and its output respected the protected membership. A test that confirms only “manual rows still exist” would miss double assignment; a test that confirms only “no automatic duplicate exists” could pass because generation did nothing.

## Evidence boundaries

The patch directly shows the narrowed lineage-deletion predicate. The strengthened regression checks protected group membership under active generation; it does not directly assert persistence of every kind of manual lineage edge. Those are different evidence levels and should not be collapsed into a claim of exhaustive edge verification.

The inspected revision also retained intermediate commits and individually committing helpers during rebuilding. This ownership correction was not a proof of operation-wide rollback or concurrent rebuild safety. Later transaction work addressed a different seam, as described in [the atomic-merge case](atomic-merge-integrity.md).

Source and existing regression changes were inspected; no test suite or rebuild was run for this write-up. The result is a useful review pattern: inventory every related table touched by an automatic reset, define protection across relationship endpoints, and ensure preservation checks execute alongside nonempty automatic output. Preserving a visible object is insufficient if the automation silently erases the relationships that give it meaning.
