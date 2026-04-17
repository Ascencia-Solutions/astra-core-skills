# astra-core-skills

Canonical behavioral rules for Astra agents. Shared between the Astra AI product and the personal Chief of Staff.

## What lives here

**Core skills** — universal behavior playbooks that define *how* Astra should handle a class of tasks, independent of where she runs or who's talking to her. These are pure markdown; they describe behavior, not implementation.

Currently:
- `time-management.md` — unified calendar + to-do playbook (multi-day scheduling, occurrence grouping, reminder rules, prioritization, cross-domain intelligence, anti-patterns).

## What does NOT live here

- **Execution layer** — tool schemas, API recipes, bash commands. Each consumer (product vs CoS) implements execution differently. Execution stays in the consumer repo.
- **Personal preferences** — "Jeremie protects Sundays", "default task list is 'Astra'". CoS-only; never leaks into the product.
- **Client-specific knowledge** — Équipement Capital's tables, Construction BA's workflows. Those belong to per-client skill files inside the product.

## How it's used

### Astra AI product (`Ascencia-Solutions/astra-ai`)
Mounted as a git submodule at `api/skills/core/`. The Function App always loads every file in that directory and appends to the system prompt.

### Astra Chief of Staff (`Ascencia-Solutions/astra-brain`)
Mounted as a git submodule at `~/astra/skills/core/`. The CoS execution-layer skills (e.g. `calendar-manager/SKILL.md`) reference the core skill for behavior and layer bash recipes + personal context on top.

## Editing rules

1. **This is the source of truth for behavior.** Fix a rule once, both agents inherit it.
2. **No execution details.** If you find yourself writing a bash one-liner, a tool schema, or a list ID — that's the wrong file.
3. **No user-specific memory.** If the rule says "Jeremie's partner is Sofia" — wrong file.
4. **Use `### skill-name` headers** for the first line of each skill file. That's how both agents discover skill identifiers.

## Releasing changes

Both consumer repos pin to a specific commit via submodule. To propagate a change:
1. Edit here, commit + push to `main`
2. In `astra-ai`: `cd api/skills/core && git pull origin main`, commit the new pointer, push
3. In `astra-brain`: `cd ~/astra/skills/core && git pull origin main`, commit the pointer, push

This intentionally requires an explicit bump per consumer — so an edit here can't silently change production behavior without a commit on the consuming side.
