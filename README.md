# agent-plugins

Agent plugins for Claude Code and Codex by [@JaminZhou](https://github.com/JaminZhou).

## Install

### Claude Code

```
/plugin marketplace add JaminZhou/agent-plugins
/plugin install watch-pr@agent-plugins
```

### Codex

Add this repository as a Codex plugin marketplace:

```bash
codex plugin marketplace add JaminZhou/agent-plugins
codex
```

Then open `/plugins`, switch to the `JaminZhou Plugins` marketplace, and install
`watch-pr`. Start a new Codex thread after installing, then ask Codex to use
`watch-pr` on the target PR.

### Codex skill-only fallback

If you only want the workflow instructions without installing the Codex plugin
package, install the skill directly:

```bash
mkdir -p ~/.agents/skills/watch-pr
curl -fsSL \
  https://raw.githubusercontent.com/JaminZhou/agent-plugins/main/plugins/watch-pr/skills/watch-pr/SKILL.md \
  -o ~/.agents/skills/watch-pr/SKILL.md
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
