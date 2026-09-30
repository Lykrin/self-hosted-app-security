# Application integrity and security boundaries

Browse these articles at [Lykrin Security Research](https://lykrin.github.io/).

Six reviews of input, ownership, transaction and privilege boundaries in a personal application. Historical Python corrections and the later Rust design are identified separately.

| Case study | Work / review date |
|---|---|
| [A gateway restart without a general root-control API](case-studies/authenticated-ingress-and-fixed-control.md) | 6 September 2026 |
| [Remove database-path authority from browser settings](case-studies/database-configuration-boundaries.md) | 14 June 2026 |
| [A self-merge deleted the record it should preserve](case-studies/atomic-merge-integrity.md) | 5 September 2026 |
| [Browser-origin checks across HTTP and WebSocket entry points](case-studies/browser-origin-and-websocket-boundaries.md) | 12 June 2026 |
| [Preserving manual ownership during an automatic rebuild](case-studies/preserving-manual-lineage.md) | 21 June 2026 |
| [Closing a failed transaction before the next caller uses it](case-studies/rollback-after-partial-snapshot-write.md) | 2 July 2026 |

Dates refer to recorded original work or the actual source review. Articles were prepared on 30 September 2026, with later analysis identified where relevant. Private evidence remains outside this repository. Research dates describe the work, not earlier public publication.
