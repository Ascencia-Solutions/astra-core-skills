# Code Verify — task-class evidence templates

> Shared with Astra AI via the `core/` submodule. CoS skill wrapper lives at
> `~/.claude/skills/code-verify/SKILL.md`.

## Purpose

The `code-verify-gate.sh` Stop hook is the floor — it blocks the turn when
a code/config edit lands without runtime evidence. This document is the
**form**: what shape evidence should take per task class, so the proof you
log is actually convincing rather than just gate-clearing.

## When this loads

The skill wrapper auto-loads when an `Edit` or `Write` tool call touches a
file matching the gate's extension list (`.py .ts .tsx .js .jsx .sh .sql
.cs .bicep .yml .yaml .json .toml .env`, plus `Dockerfile / tsconfig.json
/ package.json`). Don't rely on natural-language triggers — they're
unreliable per the architecture doc's known concerns.

## Evidence templates by class

Pick the template that matches the artifact you changed. If multiple
apply, do the first one whose tool you can run; ladder up only if the
first fails or doesn't fit.

### UI change (.tsx, .jsx, components, CSS)

- **Preferred:** Playwright visual run touching the changed component, or
  a saved screenshot path written from a browser session.
- **Acceptable:** `bun test` / `vitest` / `jest` over the component's test
  file with a passing assertion that exercises the new behavior.
- **Inadequate:** "It compiles." Type-check passing is a precondition,
  not evidence.

### API / endpoint (.py / .ts route handler, FastAPI, Express, Azure Function)

- **Preferred:** `curl -s -o /dev/null -w '%{http_code}\n' <url>` against
  the endpoint with the new behavior, plus body excerpt for a happy-path
  payload.
- **Acceptable:** Integration test that boots the app and hits the route.
- **Inadequate:** Unit-testing the handler in isolation while leaving
  routing/middleware untested.

### DB schema / data (.sql, migrations, seed scripts)

- **Preferred:** Run the migration against a scratch DB; `SELECT count(*)`
  before/after and a sample row from the new shape.
- **Acceptable:** Dry-run output (e.g. EF Core `migrations script`,
  `alembic upgrade --sql`) plus a row-count sanity check.
- **Inadequate:** Reading the migration file and asserting it "looks
  right."

### CLI / script (.sh, .py, scripts called from cron)

- **Preferred:** Execute the script by basename (`bash foo.sh` /
  `python foo.py`) — the gate matches whole-token basenames in the Bash
  command and clears on success. Capture exit code + stdout/stderr
  excerpt.
- **Acceptable:** Idempotency check — run twice, confirm the second run
  is a no-op (or produces the documented diff).
- **Inadequate:** Just running `--help`.

### Refactor (no behavior change intended)

- **Preferred:** Full test suite green + a `git diff --stat` showing the
  intended scope (no surprise files touched).
- **Acceptable:** Targeted suite over the modified packages.
- **Inadequate:** "I didn't change behavior" without proof the tests
  actually exercised the changed paths.

### Config / infra (.yml, .yaml, .json, .toml, .bicep, Dockerfile)

- **Preferred:** Service health check against the deployed config —
  `curl /healthz`, `kubectl get pods`, `docker ps`, `az resource show`.
- **Acceptable:** Validator pass (`bicep build`, `terraform validate`,
  `docker build`, `yamllint`) + before/after diff of the live state.
- **Inadequate:** "The YAML parses." Schema-valid config can still ship a
  broken service.

### Hooks / shell glue (`~/.claude/hooks/*.sh`)

- **Preferred:** Stand-alone smoke-test script that simulates the JSON
  envelope Claude Code passes and asserts allow/block per the cases the
  hook is meant to cover. (See `/tmp/verify-hook-smoketest.sh` from the
  v1 build for the pattern.)
- **Acceptable:** Manual replay against a captured stdin sample with the
  expected exit code printed.

## The waiver lane

If runtime evidence genuinely doesn't apply (e.g. a comment-only edit
that happens to live in a `.ts` file, or a deliberate
work-in-progress that you intend to verify in a follow-up turn), append
the override sentinel to your final reply:

```
[VERIFY-WAIVED: <one-sentence reason>]
```

Every waiver is appended to `~/.claude/memory/work/verify-waivers.jsonl`
with `{ts, session_id, files_edited, reason, edit_count}`. The weekly
retrospective surfaces the waiver rate per file class. If config-only
or docs-typo waivers are dominating, that's a signal the gate is too
tight and we ratchet back — not that 80% of edits genuinely don't need
verification.

Waiver discipline:
- **One sentence, specific.** "docs-only typo in src/foo.ts comment block"
  not "wip" or "skip".
- **Don't waive to clear the gate so you can declare victory.** The
  retro will catch you. Either run the evidence command or own the
  waiver.

## Anti-patterns the gate ignores by design

- `Read`-only checks ("the file exists, looks right") — the gate doesn't
  count reads as evidence.
- `Bash(echo ok)` or other no-op commands — the matcher requires a test
  runner, basename match, HTTP probe, screenshot, or UI-automation tool.
- Type-checks alone (`tsc --noEmit`) — they confirm the compiler is
  happy, not that the runtime behavior is what you intended. Use them as
  a precondition, not as the verification step.

## Where the gate is documented

- Hook source: `~/.claude/hooks/code-verify-gate.sh` and
  `code-verify-track.sh`.
- Algorithm reference: `v1.0.2 Critical Rule #10`, Phase 6 step 3.
- CLAUDE.md alignment: §Algorithm Instant/Quick paragraph.
