---
description: "Monitor Codex PR review, triage + fix findings (one bug = one commit), re-trigger until clean, gate on CI green, then notify. Never merges. Auto-entered after posting @codex review."
argument-hint: "[optional: PR number — defaults to the current branch's PR]"
allowed-tools: Bash(gh *), Bash(git *), Bash(sleep *), Bash(curl *), Bash(cd *), Bash(ls *), Bash(grep *), Bash(cat *), Bash(cut *), Bash(sed *), Bash(jq *), Read, Edit, Write, Glob, Grep, Agent, TodoWrite, Skill
---

<!--
  CUSTOMIZE BEFORE USE — search for "CUSTOMIZE" markers below and fill in:
    1. Your test/lint commands (Phase 3, step 4)
    2. Your notification channel (Phase 6 — Telegram example provided)
    3. Your issue tracker for deferred findings (Phase 2)
    4. Optional: a triage-rules file for your repo's accept/reject policy
-->

# Monitor Codex — Review → Triage → Fix → Re-trigger → CI-green → Notify

Fire-and-forget Codex review loop. Poll for Codex's PR review, triage every finding,
fix the accepted ones (reproduce-first TDD, **one bug = one commit**), re-trigger Codex once
the whole batch is pushed, repeat until Codex is clean, then wait for CI to go green and
notify the maintainer. **Never merges.**

This command is **monitor-only** and is normally **auto-entered** — every time you post
`@codex review` as part of the standard workflow, immediately run this procedure to
completion instead of ending the turn. The maintainer does not run it by hand.

## Critical Rules

- **NEVER merge.** Open/leave the PR for review and stop. Only the maintainer merges. No `gh pr merge`, no `git merge`, no suggestion to merge.
- **NEVER force-push, never `git reset --hard`, never `git stash`.** If the working tree is unexpectedly dirty, stop and report.
- **NEVER push to `main` / `master`.** This loop only runs on a feature branch's PR.
- **NEVER `git add -A` / `git add .`.** Stage files by explicit name. Never commit: `.env*`, `settings.json`, `settings.local.json`, editor/IDE state, plans, or secrets.
- **One finding = one commit.** Each accepted fix is its own commit, independently reviewable and revertable.
- **A plain `git push` does NOT trigger Codex** — only an `@codex review` comment does. So push each fix commit freely, and post `@codex review` **exactly once**, after the entire batch is pushed. Never re-trigger Codex on a half-finished batch.
- **Reproduce-first TDD for every accepted P0–P2** (and any accepted P3): write a failing test that proves the bug *and* its trigger against the real code, confirm red for the right reason, then fix. Keep the test as a regression guard.
- **Never edit a test to make it pass.** If a fix breaks an existing test, the default assumption is the fix is wrong — not the test.
- **When in doubt, stop, notify the maintainer — then WAIT.** Do not guess, do not improvise a workaround, do not take any further action until they reply. If they do not respond, do nothing and stay stopped. This is not an error state; it is the correct state.
- **Triage is mandatory before touching code.** Codex findings are advisory input, not a work order.

## Inputs you must capture up front

Run these (in parallel where possible) and hold the values for the whole loop:

```bash
git branch --show-current                                   # must NOT be main/master — if it is, STOP
PR=${ARGUMENTS:-$(gh pr view --json number -q .number 2>/dev/null)}   # PR number
gh repo view --json nameWithOwner -q .nameWithOwner         # owner/repo
git rev-parse HEAD                                          # current HEAD sha (the commit Codex must review)
```

- If there is no PR for this branch → tell the maintainer there's nothing to monitor and stop.
- If on `main`/`master` → stop.
- Record the **trigger time** (`date -u +%Y-%m-%dT%H:%M:%SZ`) so you only consider Codex responses created *after* the latest `@codex review`.

Build a TodoWrite list: Poll for Codex → Triage findings → Fix batch (one commit each) → Re-trigger → (loop) → CI-green gate → Notify.

## Phase 0: Entry contract (monitor-only)

The standard workflow posts `@codex review` right before this procedure is entered, so **assume the trigger is already in flight** — do not post a second one (it would double the review). Only post `@codex review` yourself if **both**:
1. No Codex response (clean comment or review) exists whose `Reviewed commit` matches the current HEAD, **and**
2. No `@codex review` comment exists newer than the current HEAD's push.

If you do post it, record a fresh trigger time. Then go to Phase 1.

## Phase 1: Poll for Codex's response (15-minute cap)

Codex posts as **`chatgpt-codex-connector[bot]`**, usually within 4–5 min (10 max). Loop, `sleep 60` between iterations, **hard cap 15 minutes** from the trigger time.

Each iteration, look for a response that reviewed **the current HEAD** (ignore stale responses from earlier commits — match the short sha in `**Reviewed commit:** <sha>` as a prefix of `git rev-parse HEAD`):

1. **Clean signal** — an *issue comment*:
   ```bash
   gh api repos/{owner}/{repo}/issues/$PR/comments \
     --jq '.[] | select(.user.login=="chatgpt-codex-connector[bot]") | {created_at, body}'
   ```
   Body contains `Didn't find any major issues` and (when present) a `Reviewed commit` matching HEAD, created after the trigger time → **CLEAN → go to Phase 5 (CI gate).**

2. **Findings** — a PR *review*:
   ```bash
   gh api repos/{owner}/{repo}/pulls/$PR/reviews \
     --jq '.[] | select(.user.login=="chatgpt-codex-connector[bot]") | {id, submitted_at, body}'
   ```
   Latest review whose body starts with `### 💡 Codex Review` and whose `Reviewed commit` matches HEAD → **FINDINGS → go to Phase 2.**

3. **Pending** → log `"Polling Codex review on PR #$PR... (attempt N, elapsed Xm)"` and `sleep 60`.

4. **15-min timeout with no response for HEAD** → notify: `PR #$PR — Codex did not respond within 15 min. Please check manually.` → **STOP and WAIT.** Do not act further.

## Phase 2: Triage findings (advisory — decide BEFORE touching code)

Fetch the inline findings for the latest Codex review:

```bash
gh api repos/{owner}/{repo}/pulls/$PR/comments \
  --jq '.[] | select(.user.login=="chatgpt-codex-connector[bot]") | {path, line, body}'
```

Parse each finding:
- **Severity** from the badge: `badge/P0` / `badge/P1` / `badge/P2` / `badge/P3`.
- **Title** from the leading `**...**`, plus the description, `path`, and `line`.

For each finding decide **accept / reject / defer**:

<!-- CUSTOMIZE: if you keep a repo-specific triage-rules file, reference it here, e.g.
     "per @.claude/artifacts/codex-bugfix-rules.md" -->

- **Real bug vs. intended behaviour.** If "fixing" it would change intended product design/core functionality → it's a product decision, not a bug → **STOP, notify the maintainer, WAIT.**
- **Severity.** Accept **P0/P1/P2**. **P3** only if cheap and clearly worth it; otherwise reject (logged) or defer.
  - P0 = crash / infinite loop / structural break / exploitable security hole.
  - P1 = major functional flaw (missing validation, broken reactivity, silently swallowed errors).
  - P2 = code smell / future liability (dup helpers, redundant queries, missing loading/error states).
  - P3 = minor debt / edge cases / style.
- **Real-world probability.** Negligible-probability theoretical issues may not warrant a fix — log the reasoning and move on. **Exception:** security / data-loss / auth get fixed regardless of likelihood.
- **Already-planned work?** If it's covered by the active plan/roadmap → defer to that plan, don't fix ad hoc.
- **Verify against the real code.** Read the cited file *and its callers/adjacent files*. Do not trust the finding's description of what the code does.
- **When genuinely unsure → STOP, notify, WAIT.** Don't guess.

Produce a triage table: `{severity, file, line, decision (accept/reject/defer), one-line reason}`. Log it to the terminal. **Document rejected findings** briefly so they aren't re-litigated next round. **Defer (out-of-scope but valid)** → create a ticket in your issue tracker <!-- CUSTOMIZE: Linear/Jira/GitHub Issues team + project --> instead of expanding the PR.

If everything is reject/defer (nothing to fix) → re-trigger is unnecessary; the code is unchanged, so go straight to **Phase 5 (CI gate)** if CI hasn't been confirmed, else Phase 6.

## Phase 3: Fix the accepted batch — ONE BUG = ONE COMMIT

Work accepted findings in severity order (P0 → P1 → P2 → P3). For **each** accepted finding:

1. **Reproduce (red).** Write a failing test that demonstrates the bug *and* its trigger against the real code path. Run it; confirm it fails for the bug, not a setup error. If you cannot make it fail, the finding is **unconfirmed → reject it** (log) and move on.
2. **Smallest correct fix** at the right ownership boundary — no refactor/rewrite unless it cleanly removes the whole bug class. **Sweep sibling instances** of the same bug class **within the PR's touched surfaces** and fix them in the same commit.
3. **Treat the root cause, not the symptom.** A guard that suppresses an effect is a symptom patch — trace upstream.
4. **Blast-radius check (green).** Re-run the new test **plus** adjacent/golden tests for the touched area:
   <!-- CUSTOMIZE: replace with your repo's test/typecheck/lint commands, e.g. -->
   - Backend: `cd apps/api && <your test runner> <relevant paths>`
   - Frontend: `cd apps/web && <your test runner> <relevant>` and `<your typecheck command>`
   - If a fix breaks an existing test → assume the fix is wrong; fix the fix, don't weaken the test.
5. **Commit this single finding** (stage by explicit name), conventional prefix, body referencing the finding:
   ```
   fix: <short subject — the Codex finding title>

   Codex <Pn> finding on <path>:<line> — <one-line description>.
   Reproduced with <test name> (now green); root cause: <...>.
   ```
6. **`git push`** (never force-push). **Do NOT post `@codex review` yet.**

Repeat for the next accepted finding (next commit, next push). The whole batch lands as N separate commits, none of which re-triggers Codex.

## Phase 4: Re-trigger Codex once — then loop

Only after **every** accepted fix is committed + pushed and the working tree is clean:

1. Confirm tests green and `git status` clean.
2. Capture the new HEAD sha and a fresh trigger time.
3. Post **one** trigger:
   ```bash
   gh pr comment $PR --body "@codex review"
   ```
4. Go back to **Phase 1** to poll for Codex's next review against the new HEAD. **No cap on the number of rounds** — keep going until Codex is clean.

## Phase 5: CI-green gate (after Codex is CLEAN)

Once Codex returns CLEAN for the current HEAD, poll the **branch** CI:

```bash
gh pr checks $PR --watch        # or poll: gh pr checks $PR
```

- All required checks pass → **go to Phase 6.**
- A check **fails** → this is in-scope. Read the failure, fix it (reproduce-first if it's a real defect), commit (one logical fix = one commit), push, then **re-trigger Codex (Phase 4)** since the code changed — back through the loop.
- Pre-existing lint/format drift on files outside your diff is noise, not your regression.

## Phase 6: Notify + STOP (never merge)

Codex CLEAN **and** CI GREEN → notify the maintainer and stop.

Example — Telegram Bot HTTP API (**ASCII only** — emoji can break UTF-8 encoding in some Windows shells; no temp files):

<!-- CUSTOMIZE: swap for your channel — Slack webhook, ntfy, email, or plain terminal output.
     Store secrets outside the repo, e.g. in ~/.claude/channels/telegram/.env -->

```bash
TOKEN=$(grep TELEGRAM_BOT_TOKEN ~/.claude/channels/telegram/.env | cut -d'=' -f2)
CHAT_ID=$(grep TELEGRAM_CHAT_ID ~/.claude/channels/telegram/.env | cut -d'=' -f2)
MSG="PR #$PR clean - Codex found no issues and CI is green. Ready for your review + smoke test. Fixed <K> findings across <R> round(s). <pr_url>"
curl -s -X POST "https://api.telegram.org/bot${TOKEN}/sendMessage" \
  -d "chat_id=${CHAT_ID}" \
  --data-urlencode "text=${MSG}"
```

- Also print the same summary to the terminal (terminal output is canonical).
- If the curl fails → log it and keep the terminal summary; do not retry destructively.
- **The workflow ends here. NEVER merge.** Only the maintainer merges, after their review + smoke test.

## When to stop and wait for the maintainer (do not panic, do not act)

Send a notification and then **stop and wait indefinitely** — taking no further action until the maintainer replies — in any of these cases:
- A finding would change intended product design / core functionality (product decision).
- You are genuinely unsure whether a finding is a real, in-scope defect.
- A new dependency would be needed to fix a finding (supply-chain decision).
- Codex did not respond within the 15-minute cap.
- Anything unexpected breaks (push rejected, a classifier blocks an action, an agent race) — lay out the state in the ping and wait.

If the maintainer does not respond, that is fine — remain stopped. Do not improvise a workaround.

## Error handling

| Situation | Action |
|-----------|--------|
| No PR for the branch | Tell the maintainer, stop |
| On `main`/`master` | Stop — this loop never runs on the default branch |
| Codex silent for 15 min | Notify "did not respond", STOP + WAIT |
| Finding is a product decision / ambiguous | Notify, STOP + WAIT (no guessing) |
| Can't reproduce a finding as a failing test | Reject it (log reason), move on |
| A fix breaks an existing test | Assume the fix is wrong; fix the fix, never weaken the test |
| Out-of-scope but valid finding | Ticket in issue tracker, do not expand the PR |
| New dependency needed | Notify, STOP + WAIT (supply-chain decision) |
| Working tree unexpectedly dirty | Stop, report (no stash, no reset) |
| Notification send fails | Log, keep terminal summary; do not retry destructively |
| Any step suggests merge / force-push / reset --hard / stash | Refuse |

## Execution flow

```
(after standard `@codex review` is posted)
  -> Phase 0: confirm trigger in flight (post only if HEAD unreviewed)
  -> Phase 1: poll Codex (<=15 min, match Reviewed commit == HEAD)
        |- CLEAN  -> Phase 5
        |- FINDINGS -> Phase 2
        '- 15-min silent -> Notify + STOP/WAIT
  -> Phase 2: triage (accept/reject/defer)
  -> Phase 3: fix accepted -> reproduce-first TDD -> ONE COMMIT EACH -> push (no re-trigger)
  -> Phase 4: whole batch pushed -> ONE `@codex review` -> back to Phase 1 (no round cap)
  -> Phase 5: CI-green gate on the BRANCH (fail -> fix -> push -> re-trigger -> loop)
  -> Phase 6: Notify (ASCII) "clean + green, ready for review" -> STOP. NEVER MERGE.
```
