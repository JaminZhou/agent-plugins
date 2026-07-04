# claude-plugins

Claude Code plugins by [@JaminZhou](https://github.com/JaminZhou).

## Install

```
/plugin marketplace add JaminZhou/claude-plugins
/plugin install watch-pr@jamin-plugins
```

## Plugins

### watch-pr

Watch a PR for automated bot review rounds (Codex etc.) and drive each round
to resolution — verify and fix each finding (or reject with a rationale),
push, reply with the fixing SHA, resolve the bot threads, then watch for the
next round in the background — until a round comes back clean.

Key empirics baked in: Codex reviews every push ~4–7 minutes later; a clean
round is a head-commit review with zero inline comments, not the absence of a
review; never merge before the head commit's review lands.

Requires the [GitHub CLI](https://cli.github.com) (`gh`) authenticated for
the target repo.
