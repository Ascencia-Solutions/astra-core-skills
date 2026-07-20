

## Migrated from KG (2026-07-19)

### Memory `463f03052903` (importance 8, created 2026-07-18)

Generated memory views regenerate FROM memory.db — so fixing a stale fact by editing ~/.claude/memory/*.md is worthless, and 'leave it for the dream cycle' is worse than worthless: the cycle faithfully reproduces the staleness from the unchanged DB source. Proven 2026-07-18 — infrastructure.md still named the retired MI as the live Key Vault credential path, and the source entries (e6c7a80eaaa9, 26c257e31990) were themselves stale. Correct rule: when a generated view is wrong, query memory.db for the entries that feed it and fix THOSE, then let the view regenerate.
