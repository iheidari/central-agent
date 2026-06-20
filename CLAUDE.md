# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Shared Claude Code automation distributed to other repos **by reference** (git URL), not published as a package. There is no build, test, or lint step — the deliverables are Markdown (skills/agents), JSON (plugin/marketplace manifests), and a YAML GitHub Actions workflow. "Editing the files" is the work; correctness is enforced by schema and by Claude Code loading the plugin, not by a compiler.

Two independent consumption paths ship from here:

1. **Reusable PR-review workflow** — [.github/workflows/claude-review.yml](.github/workflows/claude-review.yml), called from other repos via `uses: iheidari/central-agent/.github/workflows/claude-review.yml@main`. It runs two `anthropics/claude-code-action@v1` steps: `/simplify` (built-in) and `/thermos:thermos` (the plugin below, pulled in via the action's `plugin_marketplaces`/`plugins` inputs).
2. **Plugin marketplace** — [.claude-plugin/marketplace.json](.claude-plugin/marketplace.json) publishes the `thermos` plugin for interactive install (`/plugin marketplace add iheidari/central-agent` → `/plugin install thermos@central-agent`).

## The thermos wiring (read these together)

The plugin is a chain of indirection; changing behavior usually means touching more than one link:

- [.claude-plugin/marketplace.json](.claude-plugin/marketplace.json) — declares the `thermos` plugin and points `source` at [plugins/thermos](plugins/thermos).
- [plugins/thermos/.claude-plugin/plugin.json](plugins/thermos/.claude-plugin/plugin.json) — declares the **two agents** explicitly. It does **not** list skills: skills are auto-discovered from `plugins/thermos/skills/`.
- [plugins/thermos/skills/thermos/SKILL.md](plugins/thermos/skills/thermos/SKILL.md) — the user-facing orchestrator, invoked as `/thermos:thermos` (`plugin:skill`). It has `disable-model-invocation: true`, so it only runs when explicitly called. It launches both subagents in parallel with `run_in_background: true`, then synthesizes/dedupes their findings.
- [plugins/thermos/agents/thermo-nuclear-review-subagent.md](plugins/thermos/agents/thermo-nuclear-review-subagent.md) and [thermo-nuclear-code-quality-review-subagent.md](plugins/thermos/agents/thermo-nuclear-code-quality-review-subagent.md) — the two subagents. Each is a thin wrapper: its job is to **load its matching rubric skill** and follow it exactly, with a self-contained fallback if the skill isn't available.
- [plugins/thermos/skills/thermo-nuclear-review/SKILL.md](plugins/thermos/skills/thermo-nuclear-review/SKILL.md) (bug/security/correctness/devex/feature-leak rubric) and [thermo-nuclear-code-quality-review/SKILL.md](plugins/thermos/skills/thermo-nuclear-code-quality-review/SKILL.md) (maintainability / 1k-line / spaghetti / code-judo rubric) — the actual review logic.

So: `marketplace → plugin → thermos skill → 2 subagents → 2 rubric skills`. The two rubric skills are also usable standalone (they don't disable model invocation), so they have value independent of `thermos`.

## Conventions that bite

- **`version` and `description` are duplicated across two manifests**: [marketplace.json](.claude-plugin/marketplace.json) and [plugin.json](plugins/thermos/.claude-plugin/plugin.json) each carry the plugin's `version` and `description`. The spec does not force them to match, so edit both together or they silently drift.
- **Adding an agent** requires adding it to the `agents` array in `plugin.json`. **Adding a skill** does not — dropping a `skills/<name>/SKILL.md` is enough (auto-discovery). Both must be referenced relative to the plugin root.
- Every skill/agent `.md` needs YAML frontmatter with `name` and `description`; the `description` is what Claude Code uses to decide relevance, so write it as a trigger phrase, not a summary.
- Subagents assume the **parent already gathered the diff/file contents** and passes them in labeled `### Git / diff output` and `### Changed file contents` sections — they don't fetch context themselves by default.
- The workflow runs the skills in **review-only** mode (comment, never commit). Preserve that framing when editing prompts in [claude-review.yml](.github/workflows/claude-review.yml).

## Validating changes

There's no test suite. After editing manifests, sanity-check JSON and try a local install:

```sh
# JSON well-formedness
jq empty .claude-plugin/marketplace.json plugins/thermos/.claude-plugin/plugin.json

# Load this checkout as a marketplace locally to confirm the plugin resolves
# (run inside a Claude Code session):
#   /plugin marketplace add ./
#   /plugin install thermos@central-agent
```

When changing a skill or agent, confirm it still appears (and `/thermos:thermos` still resolves) after reinstalling the local plugin.
