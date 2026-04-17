# Astra — three-repo architecture

> Single source of truth for how the Astra AI Product, Astra CoS, and Astra Core Skills repos relate, what each owns, and the hard-wall rules that keep them from contaminating each other. If something in a consumer repo contradicts this file, this file wins.

## The three identities

| Identity | Brand name | Code slug | Banner | Owner tag | Who it's for |
|---|---|---|---|---|---|
| **Customer product** | Astra AI | `astra-product` | 🌐 `ASTRA AI (PRODUCT)` | `owner=astra-ai` | Paying CEO customers (Maxime, Hector, future clients) |
| **Personal agent** | Astra CoS | `astra-cos` | 🏠 `ASTRA COS (PERSONAL)` | `owner=astra-cos` | Jeremie Duplessis-Savard, and only Jeremie |
| **Shared skills** | Astra Core | `astra-core` | 🔗 `ASTRA CORE (SHARED)` | `owner=astra-core` | Both of the above, as a library |

**Normative rule:** never say or write "Astra" unqualified in any durable artifact (commits, docs, memory entries, resource names). Always attach the qualifier.

"Brain" is banned — it reads as "the smart core of Astra AI Product," which it is not.

## Repos

| Repo | Purpose | Checked out at (on vm-paperclip) |
|---|---|---|
| `Ascencia-Solutions/astra-ai` | Astra AI Product source | `~/astra-ai/` |
| `Ascencia-Solutions/astra-cos` | Astra CoS source (renamed from `astra-brain`, 2026-04-17) | `~/astra/` |
| `Ascencia-Solutions/astra-core-skills` | Astra Core shared behavior | consumed as git submodule (see below) |

## Submodule layout

Both consumers mount `astra-core-skills` at the **same relative path name** on each side, so skill files have identical paths within their tree:

- Product: `~/astra-ai/api/skills/core/`
- CoS: `~/astra/skills/core/`

Inside: `time-management.md`, `ARCHITECTURE.md` (this file), `README.md`. Add new core skills at the top level of that repo.

## What each repo owns — and must never touch

| | Astra AI (product) | Astra CoS (personal) | Astra Core (shared) |
|---|---|---|---|
| **Runtime location** | Azure: Function App `astra-mcp-api`, SWA `astra-admin`, Postgres `astra-product-db` | vm-paperclip | n/a (text only) |
| **Memory store** | Postgres `astra-product-db` (canadacentral) with tenant-isolated schema | SQLite `~/astra/memory.db` with sqlite-vec | n/a |
| **Memory verb** | `api/lib/memory.ts` inside the backend only | `astra-remember` CLI (absolute path `~/bin/astra-remember`) | n/a |
| **Write path** | Through the Function App, per-tenant, per-user | Bash scripts + dream cycle | Edit + commit to this repo |
| **May read product memory?** | ✅ yes, that's its job | ❌ NEVER | ❌ |
| **May read CoS memory?** | ❌ NEVER | ✅ yes, that's its job | ❌ |
| **May run the other's verbs?** | ❌ no `astra-remember` in product code | ❌ no direct Postgres calls in CoS code | ❌ |
| **Jeremie-specific facts** (Sofia, Sunday sacred, Laval commute) | ❌ NEVER in system prompt or memory | ✅ first-class | ❌ not in core skills — they're universal |

## Decision rules — where does a new thing go?

When you're about to write a new rule, skill, script, or resource, run this checklist:

1. **Is this a behavior rule that any exec would benefit from?** (e.g. "ask before creating a multi-day block without daily hours") → **Astra Core** (`astra-core-skills/<skill>.md`). Bump submodule pointer in each consumer afterward.
2. **Is it implementation detail — tool schema, bash recipe, SQL, JSON payload?** → the relevant consumer's execution layer. Product: `api/lib/`, `api/skills/core/` is off-limits for implementation. CoS: `~/astra/skills/time-management/SKILL.md` for wrappers, `~/.claude/scripts/` for execution.
3. **Is it personal to Jeremie?** → Astra CoS only (`~/astra/skills/*/SKILL.md` or CoS memory). Never anywhere in Astra Core or Astra AI Product.
4. **Is it client-specific (Équipement Capital, Construction BA)?** → `~/astra-ai/api/clients/<client-dir>/` only. Never in core or CoS.

## Azure naming conventions

Going forward (applies to any new resource):

| Resource scope | Naming | Resource group |
|---|---|---|
| Product-only | `astra-product-<role>` | `rg-astra-product` |
| CoS-only | `astra-cos-<role>` | `rg-astra-cos` (create when needed) |
| Shared infrastructure | `ascencia-<role>` (existing convention) | `rg-ascencia-shared-ai` |

Historical exceptions (grandfathered, tagged not renamed):
- `astra-mcp-api` (Function App for Product) — stays in `rg-ascencia-shared-ai`, tagged `owner=astra-ai`
- `astra-admin` (SWA for Product) — same
- `stastramcpapi` (Storage for Product) — same
- `vm-paperclip` + `rg-paperclip-prod` (CoS VM + RG) — stays, tagged `owner=astra-cos`
- `ascencia-openai`, `kv-ascencia-astra` (shared) — tagged `owner=astra-core`

## Azure resource tags

Every resource gets two tags:

```
purpose = product | personal | shared
owner   = astra-ai | astra-cos | astra-core
```

Filter by tag for billing attribution and ops filtering:

```bash
az resource list --query "[?tags.owner=='astra-ai'].name" -o tsv
```

## Key Vault secret naming (going forward)

- Product: `PRODUCT-<role>` (e.g., `PRODUCT-PG-CONNECTION-STRING`)
- CoS: `COS-<role>` (e.g., `COS-GRAPH-CLIENT-SECRET`)
- Shared: unprefixed (e.g., `AZURE-OPENAI-KEY`), but set the KV secret tag `owner=astra-core`.

Existing unprefixed product/CoS secrets stay. Rename during the next rotation of each.

## Commit message conventions

| Repo | Prefix format |
|---|---|
| `astra-ai` | `product: <what changed>` or conventional-commit with `product` scope |
| `astra-cos` | `cos: <what changed>` or conventional-commit with `cos` scope |
| `astra-core-skills` | `core: <what changed>` or conventional-commit with `core` scope |

Rationale: grep-able git log, instant identification in cross-repo notifications.

## How to propagate a behavioral rule change

When a rule in `astra-core-skills` changes, both consumers must explicitly pick it up — the pointer bump is intentional, so an edit here cannot silently change production or personal behavior:

```bash
# 1. Edit the rule in its canonical home
cd astra-core-skills
$EDITOR time-management.md
git commit -am "core: <summary>" && git push

# 2. Bump submodule pointer in the product
cd ~/astra-ai
git -C api/skills/core pull origin main
git add api/skills/core
git commit -m "product: bump astra-core pointer (<summary>)"
git push

# 3. Bump submodule pointer in the CoS
cd ~/astra
git -C skills/core pull origin main
git add skills/core
git commit -m "cos: bump astra-core pointer (<summary>)"
git push
```

## Hard-wall check — if in doubt

If you're about to write a commit, a config, or a line of code and you're unsure which repo it belongs in, ask:

1. *Would this help paying customers?* → product
2. *Would this help only Jeremie?* → CoS
3. *Would this help any exec using an AI chief of staff?* → core
4. *Does it reference Sofia, Sunday, Laval, Ville de Laval, or any household detail?* → CoS, never core or product

When the answer is unclear, default to the **consumer** you're in — don't pollute the shared core with specifics you're not sure about.

## Session-start identity

On vm-paperclip, Claude Code's `~/.claude/hooks/session-start.sh` emits a banner based on the current working directory. The three possible banners:

- `$PWD` inside `~/astra-ai/*` → 🌐 ASTRA AI (PRODUCT)
- `$PWD` inside `~/astra/*` → 🏠 ASTRA COS (PERSONAL)
- Otherwise → no identity banner (operator is at the home dir or elsewhere)

The banner appears in the session's opening context so an AI agent cannot mistake identity.

## What is NOT in this document

- Client-specific knowledge (Équipement Capital's tables, Construction BA's workflows) — that's `~/astra-ai/api/clients/<client>/knowledge.md`.
- The time-management playbook itself — `time-management.md` in this repo.
- Jeremie's personal operating rules — `~/astra/CLAUDE.md` and `~/astra/telos/`.
- Deploy procedures — GitHub Actions workflows in each consumer repo.
