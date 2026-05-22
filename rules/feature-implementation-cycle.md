# Feature implementation cycle (Rex playbook)

The end-to-end process used to ship F.1 (candidate email) from brief to production-verified.
Follow this for any non-trivial feature.

---

## 0. Session setup — verify cwd FIRST

Per `feedback_primary_cwd_parent_dir.md` + `feedback_secret_hygiene_vercel_argv_leak.md`: Vercel CLI, `pnpm`, `supabase`, and most tooling silently target the wrong project when invoked from the parent dir.

**The Rex monorepo trap:** `/Users/peeyush/projects/rex-support/` contains `rex-ats/`, `rex-support-os/`, `rex-project-tracker-os/`. Launching Claude Code from the parent means `vercel env add ...` targets whichever project Vercel decides — often the wrong one. `pnpm dev` picks up the wrong lockfile.

**Permanent fix:** launch Claude Code from inside the target repo. Use the `claude-ats` alias (or equivalent per-project alias) so the cwd is the repo root from the start.

**Mid-session fix (if you launched from the parent):** every Bash call must `cd` to the project first, OR use an absolute path. Default bashes without `cd` ARE wrong even if they seem to work — `vercel` in particular reads `.vercel/project.json` from cwd to decide which project to target.

**Verify at session start:**
```bash
pwd                          # should be the project root, not the parent
ls .vercel/project.json 2>/dev/null && cat .vercel/project.json | head -3  # confirms Vercel project link
git rev-parse --show-toplevel  # confirms git root matches
```

If cwd is wrong: STOP and either restart Claude Code from the right dir, or commit to `cd` on every Bash call for the rest of the session.

**Also verify working-tree state** (per `feedback_vercel_deploy_bypasses_git.md`):
```bash
git status                              # any uncommitted feature work?
git log origin/main..HEAD --oneline     # any unpushed commits?
```
Uncommitted files that look like real feature work (not auto-appended noise like AGENTS.md from claude-mem) may have been deployed to prod via `vercel deploy --prod` from a previous session. **Do not discard them.** Ask the user: "These look like prod-deployed changes — should I commit them so main catches up to prod?" Discovered 2026-05-21 with F.1 inbound webhook fixes that were live on prod but uncommitted locally.

---

## 1. Brief

Write the brief BEFORE writing any code. Brief lives in `docs/features/<X>-<slug>.md`.

**Required sections** (per `feedback_brief_authoring_process.md`):
- Repo traceability — cite every file, function, table, RPC by exact path. No vague references.
- Decomposed task list — Task 1, Task 2, … Task N. Each task is small enough that one subagent + one PR can close it.
- Per-task acceptance criteria. No "and it should work."
- Owner-to-subagent mapping — Backend Architect for schema, Frontend Developer for UI, AI Engineer for prompts, etc. Specific match, not "use an agent."
- Two-phase review — Codex review of the brief BEFORE handoff. Bake in the feedback. Then dispatch.

**Anti-patterns to avoid:**
- Calendar weeks / day estimates / "what I need from you next" ceremony. Sequencing is dependency order.
- PM-altitude drift into engineering hygiene. Focus on user value and persona impact.

---

## 2. Per-subtask implementation

Per `feedback_test_sizing.md`: each subtask ships with **smoke spec + typecheck + lint**, NEVER a full e2e run.

**Per-task DoD:**
- Code changes (minimum-viable diff per `minimum-change-engineer` discipline).
- Unit test if the logic has branches (vitest).
- Smoke spec if there's UI (Playwright, no Send/network calls — those land in the epic e2e).
- `pnpm exec tsc --noEmit` clean.
- `pnpm exec eslint <files>` clean.
- Persona-named entities translated to scope tiers in code (`property`/`regional`/`corporate`, not Maria/Sarah/Connor).

**Deploy bundling** (per `feedback_bundled_prod_deploys.md`): if multiple small fixes are queued for the same window, bundle them into one prod deploy. Don't push 5 deploys for 5 one-line changes.

**Deploy path = `git push`, NEVER `vercel deploy --prod` from local** (per `feedback_vercel_deploy_bypasses_git.md`):
- `vercel deploy --prod` uploads the working tree directly, bypassing git entirely. Prod silently drifts ahead of `main`, the audit trail breaks, and a stray `git reset --hard` wipes the only copy of the fix.
- The correct path: `git commit` → `git push origin main` → Vercel-on-push handles the deploy. Git log and prod stay in lockstep.
- If you must hotfix without committing (rare — only when CI is broken), commit IMMEDIATELY after verifying so the audit trail catches up within the same session. Do not walk away with uncommitted prod-deployed code.

---

## 3. Production debugging cycle

When something breaks in prod after deploy, follow this order:

### 3.1 Read code first, don't grep blindly
Per `claude-md` global instructions: prefer dedicated tools. Read the suspect file before guessing.

### 3.2 Check prior art in sibling repos
The Rex monorepo has 9 products. Before reinventing, search for the pattern:
```bash
find /Users/peeyush/projects/rex-support -type f -name "*.ts" \
  -not -path "*/node_modules/*" -not -path "*/.next/*" \
  | xargs grep -l "<pattern>"
```
Real win in F.1: `rex-support-os/app/api/email/inbound/route.ts` already had the Resend plus-addressing pattern. Cribbed it directly — saved hours.

### 3.3 Use context7 for SDK shapes
Training data lags SDK changes. For any third-party library / API question, use context7 MCP:
```
mcp__context7__resolve-library-id → mcp__context7__query-docs
```
F.1 example: training data said Resend webhooks include body+headers. Context7 + the installed `.d.mts` types proved otherwise — payload is metadata-only, must call `resend.emails.receiving.get(email_id)`.

### 3.4 Direct API calls to isolate root cause
When a route returns 200 + orphan, it could be (a) our route logic, (b) their API. Hit their API directly with Python:
```bash
python3 -c "
import http.client, ssl
ctx = ssl.create_default_context()
conn = http.client.HTTPSConnection('api.example.com', context=ctx, timeout=15)
conn.request('GET', '/path', headers={'Authorization': 'Bearer ' + api_key})
r = conn.getresponse()
print('status:', r.status, r.read().decode())
"
```
F.1 example: this revealed the API key was scoped `send-only` (401 `restricted_api_key`) — not our bug at all.

### 3.5 Vercel logs in `--json` mode
The default formatter truncates warnings. Always use `--json`:
```bash
vercel logs <domain> --since 5m --json 2>&1 | grep -A 1 "<pattern>" | tail -20
```
NOT `--output raw` (broken in current CLI).

### 3.6 Supabase CLI for DB queries
Faster + cleaner than the management API:
```bash
supabase db query "SELECT … FROM …;" --linked
```
Falls back: `mcp__plugin_supabase_supabase__execute_sql` if CLI isn't available.

### 3.7 Loud-fail vs silent-orphan
When integrating with external APIs that return `{data, error}` (not throwing), check `error` and log loudly. F.1 example: Resend SDK returns `{data: null, error}` on 401 — original code only logged on `throw`, so the 401 was invisible until we added explicit logging. Mirror this for any SDK with the same shape.

---

## 4. End-of-epic e2e

One spec, all four corners. Lives at `tests/e2e/<feature>-<slug>.spec.ts`. Per `feedback_testing_layer_strategy.md`: deterministic E2E for daily; AI-agent walks (Reality Checker / browser-use) for milestone gates only.

**Structure:**
1. `test.skip(missingEnv.length > 0, ...)` — skip cleanly in environments missing secrets. Never throw at file collection time.
2. `beforeAll`: create whatever rows the test needs, idempotently (delete any leftover from prior runs first).
3. `afterAll`: cascade cleanup (children first, then parents).
4. The test itself: assert DB state after every UI action, not just the rendered UI.
5. For external services, use their test address (`delivered@resend.dev` for Resend) so the test doesn't pollute real recipients or cost money.
6. Bypass third-party callbacks by directly signing + POSTing webhooks where applicable (Svix `wh.sign()` pattern).

**Run with:** `pnpm test:e2e tests/e2e/<file>.spec.ts --reporter=list`

---

## 5. Tracker hygiene (rex-tracker)

After the e2e passes:
1. `mcp__rex-tracker__search_cards` with broad keywords (`composer`, `unsubscribe`, `<feature-name>`, `<vendor-name>`). Don't trust your memory of card IDs.
2. Move each card to `done` via `update_card_status`.
3. Do NOT promote the parent epic if other subtasks are still backlog.
4. Internal task decomposition (Tasks 1-10 within an epic) usually lives in the feature brief, not as separate tracker cards.

---

## 6. Memory hygiene

Per global CLAUDE.md auto-memory rules. Only save:
- Surprising / non-obvious findings (e.g. "Resend webhook payload is metadata-only by design — must call receiving.get").
- Feedback / corrections from the user.
- Project state that won't be obvious from `git log`.

Don't save: code patterns derivable by reading source, debugging recipes (commit message has the context), things already in CLAUDE.md.

Convert relative dates to absolute (`Thursday` → `2026-03-05`).

---

## 7. Secret hygiene

Per `feedback_secret_hygiene.md` + `feedback_secret_hygiene_vercel_argv_leak.md`:
- **Never echo secrets to stdout.** No `echo $SECRET`, no `cat .env`, no unmasked grep output.
- **Generate-into-file** for new secrets: `openssl rand -hex 32 > /tmp/secret` then `cat /tmp/secret | vercel env add ...`.
- **Mask values when inspecting** — show length + first/last 4 chars max.
- **Vercel CLI**: NEVER use `--value` for secrets (leaks via argv). Use stdin-redirect from a tmpfile with empty-string git-branch arg + `--yes` for preview pushes.
- **Vercel `env pull`** masks encrypted vars as `""` — they cannot be auto-synced. Pull from source (vendor dashboard) instead.
- **Rotate on leak.** If a secret ends up in stdout / git history / shared context, treat it as compromised.

---

## 8. Answer before escalating

Per `feedback_answer_before_escalating.md`: factual questions get 3-line answers with citations, not multi-option AskUserQuestion ceremonies. Ask only when truly ambiguous. Otherwise just answer.

---

## Quick checklist

```
[ ] cwd verified — Claude launched from project root, not parent
[ ] Working tree clean OR uncommitted feature-work reconciled with user
[ ] Brief written, Codex-reviewed, dispatched
[ ] Per-task: code + smoke + typecheck + lint clean
[ ] Deploys go via `git push`, NOT `vercel deploy --prod` from local
[ ] Bundle deploys when same window
[ ] End-of-epic e2e written and PASSING locally
[ ] Tracker cards moved to done (broad search, not just the obvious one)
[ ] Memory updated if anything surprising surfaced
[ ] No secrets leaked; rotate if they did
```
