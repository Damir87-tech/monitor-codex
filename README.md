<div align="center">

# monitor-codex

**Two AIs argue about your code until it's clean. You get pinged when it's ready to merge.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude_Code-slash_command-d97757)](https://claude.com/claude-code)
[![Works with Codex](https://img.shields.io/badge/works_with-OpenAI_Codex-10a37f)](https://openai.com/codex/)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

*A fire-and-forget review loop for [Claude Code](https://claude.com/claude-code): Codex reviews the PR, Claude triages and fixes the findings with reproduce-first TDD, re-triggers Codex, and repeats until the review is clean and CI is green — then notifies you and **stops**. It never merges.*

</div>

---

## The problem

AI code review bots are great at *finding* issues and terrible at *finishing* them. You post `@codex review`, walk away, come back to 7 inline findings, fix them by hand, re-trigger, wait again, fix again... The loop is mechanical — but somebody has to babysit it.

**monitor-codex** makes Claude Code the babysitter. You keep the only two jobs that matter: reviewing the final diff and pressing merge.

## How it works

```
        you (or your workflow) post `@codex review` on the PR
                              │
                              ▼
              ┌───────────────────────────────┐
              │  Phase 1 · Poll for Codex      │◄─────────────┐
              │  (15-min cap, HEAD-matched)    │              │
              └───────┬───────────────┬───────┘              │
                CLEAN │               │ FINDINGS             │
                      │               ▼                      │
                      │   ┌───────────────────────────┐      │
                      │   │  Phase 2 · Triage          │      │
                      │   │  accept / reject / defer   │      │
                      │   │  (advisory, not a work     │      │
                      │   │   order — verify the code) │      │
                      │   └───────────┬───────────────┘      │
                      │               ▼                      │
                      │   ┌───────────────────────────┐      │
                      │   │  Phase 3 · Fix batch       │      │
                      │   │  reproduce-first TDD       │      │
                      │   │  ONE BUG = ONE COMMIT      │      │
                      │   └───────────┬───────────────┘      │
                      │               ▼                      │
                      │   ┌───────────────────────────┐      │
                      │   │  Phase 4 · Re-trigger once │──────┘
                      │   │  (after the WHOLE batch)   │
                      │   └───────────────────────────┘
                      ▼
        ┌───────────────────────────────┐
        │  Phase 5 · CI-green gate       │──fail──► fix → push → re-trigger
        └───────────────┬───────────────┘
                        ▼
        ┌───────────────────────────────┐
        │  Phase 6 · Notify + STOP       │
        │  "clean + green, ready for     │
        │   your review"  — NEVER MERGES │
        └───────────────────────────────┘
```

## What makes it different

Most "AI fixes the review comments" setups fail in one of three ways: they treat every bot finding as gospel, they pile all fixes into one unreviewable commit, or they quietly merge. monitor-codex is built around the opposite instincts:

- **Triage before touching code.** Codex findings are advisory input, not a work order. Every finding is verified against the real code and its callers, then explicitly accepted, rejected (with a logged reason), or deferred to the issue tracker.
- **One bug = one commit.** Each accepted fix lands as its own conventional commit — independently reviewable, independently revertable.
- **Reproduce-first TDD.** No fix without a failing test that proves the bug first. If the bug can't be reproduced, the finding is rejected — the test stays as a regression guard.
- **Batch, then re-trigger once.** Pushing doesn't re-trigger Codex; a single `@codex review` after the whole batch does. No half-batch reviews, no review spam.
- **Hard safety rails.** Never merges. Never force-pushes. Never `git reset --hard`, never stashes, never pushes to `main`. When it's genuinely unsure — product decision, new dependency, anything unexpected — it notifies you and **waits indefinitely** instead of guessing.

## Install

```bash
# project-level (this repo only)
mkdir -p .claude/commands
curl -o .claude/commands/monitor-codex.md \
  https://raw.githubusercontent.com/Damir87-tech/monitor-codex/main/monitor-codex.md

# or user-level (all your repos)
mkdir -p ~/.claude/commands
curl -o ~/.claude/commands/monitor-codex.md \
  https://raw.githubusercontent.com/Damir87-tech/monitor-codex/main/monitor-codex.md
```

Then open the file and fill in the four `CUSTOMIZE` markers:

1. **Test commands** — your repo's test / typecheck / lint invocations (Phase 3).
2. **Notification channel** — Telegram example included; swap for Slack, ntfy, email, or plain terminal (Phase 6).
3. **Issue tracker** — where deferred out-of-scope findings become tickets (Phase 2).
4. **Triage rules** *(optional)* — a repo-specific accept/reject policy file.

### Prerequisites

- [Claude Code](https://claude.com/claude-code) with the [`gh` CLI](https://cli.github.com/) authenticated
- [OpenAI Codex](https://openai.com/codex/) connected to your GitHub repo (the `chatgpt-codex-connector[bot]` responds to `@codex review` comments)
- A PR on a feature branch (the loop refuses to run on `main`/`master`)

## Quick start

```
/monitor-codex          # monitors the current branch's PR
/monitor-codex 42       # monitors PR #42
```

Or wire it into your workflow so it auto-enters every time `@codex review` is posted — that's the intended mode: post the trigger, and the loop runs to completion instead of ending the turn.

Then close the laptop. When you get the notification, the PR is Codex-clean and CI-green — your only jobs left are the human review and the merge button.

## Safety guarantees

| It will never | Because |
|---|---|
| Merge the PR | Merging is a human decision. Always. |
| Force-push, `reset --hard`, or stash | Destructive history operations are banned outright |
| Push to `main`/`master` | Feature-branch PRs only |
| `git add -A` | Files are staged by explicit name; secrets and local config never land in a commit |
| Weaken a test to make it pass | If a fix breaks a test, the fix is presumed wrong — not the test |
| Guess on ambiguity | Product decisions, new dependencies, and anything unexpected → notify + stop + wait |

## FAQ

**Why Codex as the reviewer and Claude as the fixer?**
Different models make different mistakes. A cross-vendor adversarial loop catches more than either model reviewing its own work — and the human stays as the final gate.

**What if Codex flags something that isn't a bug?**
That's what triage is for. Findings that contradict intended behaviour get escalated to you; unreproducible findings get rejected with a logged reason; valid-but-out-of-scope findings become tracker tickets instead of PR scope creep.

**What if it loops forever?**
Each round strictly shrinks the finding set (fixed, rejected-with-reason, or deferred — rejections are documented so they aren't re-litigated). There's no round cap because convergence comes from triage discipline, not from a counter.

**Does it work outside of Codex?**
The loop structure (poll → triage → TDD fix → single re-trigger → CI gate → notify) is reviewer-agnostic. Adapting Phase 1's polling to another review bot is a small edit — PRs welcome.

## Star history

[![Star History Chart](https://api.star-history.com/svg?repos=Damir87-tech/monitor-codex&type=Date)](https://star-history.com/#Damir87-tech/monitor-codex&Date)

## License

[MIT](LICENSE)
