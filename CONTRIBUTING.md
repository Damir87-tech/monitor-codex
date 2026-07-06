# Contributing

Thanks for your interest in improving monitor-codex!

## Ways to contribute

- **Adapt the loop to another review bot** (GitHub Copilot review, Gemini Code Assist, CodeRabbit...) — Phase 1's polling and the bot login name are the only Codex-specific parts.
- **Add notification channel examples** — Slack webhook, ntfy, Discord, email. Keep the Telegram example as the reference.
- **Sharpen the triage rules** — better severity heuristics, better accept/reject criteria.
- **Report what broke** — if the loop stalled, double-triggered, or mis-triaged in your repo, open an issue with the PR timeline (trigger times, Codex response times, HEAD shas).

## Ground rules

The safety rails are non-negotiable. PRs that make the loop merge, force-push, reset, stash, or guess on ambiguity will not be accepted — "notify + stop + wait" is a feature, not a limitation.

## How

1. Fork, branch, make your change to `monitor-codex.md` (or docs).
2. Test it on a real PR in one of your own repos — this is a prompt artifact, so the only meaningful test is a live run. Include a short summary of that run in the PR description.
3. Open a PR.
