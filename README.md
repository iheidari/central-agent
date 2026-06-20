# central-agent

Shared Claude Code automation reused across repos: a **reusable GitHub Actions
workflow** for PR review, backed by a **Claude Code plugin marketplace**.

## Reusable PR-review workflow

[`.github/workflows/claude-review.yml`](.github/workflows/claude-review.yml) runs
two skills against a pull request's changes:

1. **`/simplify`** (built-in) — reuse / simplification / altitude cleanups, posted as comments.
2. **`/thermos:thermos`** (from the `thermos` plugin below) — a combined bug/security
   and code-quality audit that fans out to two subagents and synthesizes the findings.

### Use it from another repo

```yaml
# .github/workflows/pr-review.yml in the consuming repo
name: PR Review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    uses: iheidari/central-agent/.github/workflows/claude-review.yml@main
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

The consuming repo must define a `CLAUDE_CODE_OAUTH_TOKEN` secret (generate with
`claude setup-token`). Optional inputs: `claude_args`, `plugin_marketplaces`, `plugins`.

Full setup, pinning, inputs, and troubleshooting:
[`docs/using-the-review-workflow.md`](docs/using-the-review-workflow.md).

## `thermos` plugin

A marketplace ([`.claude-plugin/marketplace.json`](.claude-plugin/marketplace.json))
publishing the [`thermos`](plugins/thermos) plugin: the `thermos` skill plus its two
rubric skills and two review subagents.

Install it interactively in any session:

```
/plugin marketplace add iheidari/central-agent
/plugin install thermos@central-agent
```

The CI workflow installs it automatically via the action's `plugin_marketplaces` /
`plugins` inputs.
