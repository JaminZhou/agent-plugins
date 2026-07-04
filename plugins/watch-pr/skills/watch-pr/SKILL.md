---
name: watch-pr
description: Watch a PR for automated bot review rounds (Codex etc.) and drive each round to resolution (fix → push → reply → resolve → re-watch) until a clean round. Use when the user says "watch the PR" or asks to handle bot review comments.
---

# Watch PR (bot review loop)

Drive a PR through automated bot review rounds until a round comes back clean.
Written for OpenAI's Codex GitHub reviewer (`chatgpt-codex-connector[bot]`);
the loop works for any bot that reviews on push — adjust the login name.

Agent-portable: every step is plain `gh` CLI + shell, no harness-specific
tools. Works the same whether the agent running it is Claude Code, Codex, or
anything else that supports Agent Skills.

## How Codex review behaves (empirical)

- Codex is enabled repo-side (chatgpt.com/codex cloud settings) and reviews
  pushes to an open PR: a `COMMENTED` review lands roughly **4–7 minutes
  after a push**, its body contains `**Reviewed commit:** \`<sha>\``.
- **Clean-round signal: Codex adds a `+1` reaction on the PR body** (the 👍
  on the PR description; `gh api repos/<o>/<r>/issues/<N>/reactions`). It
  reviews every round: findings → a review with inline comments; clean → the
  +1 reaction instead, both ~4–7 min after the push.
- Reactions are never removed, so a +1 from an earlier clean round persists:
  the test is **+1 `created_at` LATER than the last push**, not presence.
- Fallback: if neither a head review nor a fresh +1 arrives in ~15 min,
  treat as clean but report the ambiguity.
- Findings carry P1/P2/P3 badges. The bot is strong on mechanical reasoning
  (constraint math, edge cases, dead paths): treat findings as probably right,
  verify before fixing, and when one is wrong, reject it with a code comment
  explaining why plus a thread reply — don't silently ignore it.

## The loop

1. Identify the PR: `gh pr view --json number,headRefOid` on the current
   branch, or take the PR number from the user.
2. **Wait for the review of HEAD.** Write a small script that polls every
   30s (20 min timeout) for a bot review whose body contains the head SHA
   prefix. If your harness supports background tasks (e.g. Claude Code's
   background Bash — you are re-invoked when it exits), run it there so the
   session stays free; otherwise just run it in the foreground and wait:
   Poll BOTH signals — a head review (findings) and a fresh +1 (clean):
   ```bash
   gh api repos/<owner>/<repo>/pulls/<N>/reviews \
     --jq '[.[] | select(.user.login == "chatgpt-codex-connector[bot]")
            | select(.body | contains("<head-sha-10>"))] | length'
   gh api repos/<owner>/<repo>/issues/<N>/reactions \
     --jq '[.[] | select(.user.login == "chatgpt-codex-connector[bot]")
            | select(.content == "+1")
            | select(.created_at > "<last-push-iso8601>")] | length'
   ```
   Exit on either: review → handle findings; fresh +1 → clean round, report
   ready-to-merge.
   `<last-push-iso8601>` is when you PUSHED, not the commit time (they can
   differ) — capture `date -u +%FT%TZ` right after `git push`, or use the
   head commit's `%cI` only when you commit-and-push in one breath.

3. When the review lands, fetch its inline comments
   (`gh api repos/<owner>/<repo>/pulls/<N>/comments`), filter to the bot, and
   cross-check unresolved threads via GraphQL `reviewThreads`.
   - Zero new comments → clean round: report ready-to-merge and stop.
4. For each finding: verify it against the code, then fix it — or reject it
   with an in-code rationale. Run the project's checks (tests, linters, any
   repo-specific gates) before pushing.
5. Commit following the repo's commit conventions; push.
6. Reply to each comment with the fixing commit SHA:
   `gh api repos/<owner>/<repo>/pulls/<N>/comments/<id>/replies -f body=...`
7. Resolve the **bot** threads (never resolve human review threads unless
   explicitly asked):
   ```bash
   gh api graphql -f query='mutation { resolveReviewThread(input: {threadId: "<id>"}) { thread { isResolved } } }'
   ```
   Thread IDs come from the GraphQL `reviewThreads` query in step 3.
8. Go to 2 — the push you just made triggers the next round.

## Merge (only when the user says so)

Merge per the repo's convention, then clean up: switch to the default branch,
fast-forward it, delete the PR branch locally and remotely, and report the
current branch.
