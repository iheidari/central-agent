# Using the Claude PR-review workflow in another repo

This repo ships a **reusable GitHub Actions workflow**
([`.github/workflows/claude-review.yml`](../.github/workflows/claude-review.yml))
that reviews a pull request with two skills:

1. **`/simplify`** (built-in) — reuse / simplification / altitude cleanups.
2. **`/thermos:thermos`** — a combined bug/security + code-quality audit that
   fans out to two subagents and synthesizes their findings. It is packaged as a
   Claude Code plugin in this repo, so it loads automatically on a clean runner.

You consume it by reference — there's nothing to copy. Point a tiny caller
workflow at it and add one secret.

## 1. Quick start

Create `.github/workflows/pr-review.yml` in the **consuming** repo:

```yaml
name: PR Review

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  review:
    uses: iheidari/central-agent/.github/workflows/claude-review.yml@main
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

That's the whole integration. The `uses:` line resolves to the reusable
workflow in this repo at the `@main` ref.

## 2. Add the secret

The consuming repo needs a `CLAUDE_CODE_OAUTH_TOKEN` secret.

1. Generate a token locally:

   ```sh
   claude setup-token
   ```

2. Add it under **Settings → Secrets and variables → Actions → New repository
   secret**, named exactly `CLAUDE_CODE_OAUTH_TOKEN`.

This is the only per-repo requirement, and it's independent of whether either
repo is public or private.

## 3. Pin to a version (recommended)

`@main` always tracks the latest. To insulate a repo from changes, pin to a tag
or commit SHA instead:

```yaml
    uses: iheidari/central-agent/.github/workflows/claude-review.yml@v1.0.0
```

Bump the ref deliberately when you want to pick up new behavior.

## 4. Optional inputs

All inputs have defaults; override only when needed.

| Input | Default | Purpose |
| --- | --- | --- |
| `claude_args` | `--max-turns 25` | Extra CLI args passed to `claude` (e.g. `--model`, `--max-turns`). |
| `plugin_marketplaces` | `https://github.com/iheidari/central-agent.git` | Git URL(s) of marketplaces that provide `/thermos`. |
| `plugins` | `thermos@central-agent` | Plugin(s) to install before the thermos pass. |

Example overriding an input:

```yaml
jobs:
  review:
    uses: iheidari/central-agent/.github/workflows/claude-review.yml@main
    with:
      claude_args: "--max-turns 40 --model claude-opus-4-8"
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

## 5. Permissions

The reusable workflow already declares what it needs:

```yaml
permissions:
  contents: read
  pull-requests: write
```

A caller doesn't need to redeclare these — they apply to the reusable workflow's
job. Just make sure your org/repo settings don't strip `pull-requests: write`
from Actions, or the review can't post comments.

## 6. Public vs private

- **central-agent is public**, so the reusable workflow and the plugin
  marketplace clone with no extra configuration.
- If you ever make central-agent **private**, callers must live in the same
  account/org and you must enable **Settings → Actions → General → Access →
  "Accessible from repositories owned by …"**. The plugin-marketplace clone
  would also need a token with read access, since the runner's `GITHUB_TOKEN`
  can't read a different private repo.

## 7. Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| `Error: ... could not be found` on the `uses:` line | Wrong path/ref, or central-agent is private and Access isn't enabled. |
| Thermos pass runs but `/thermos:thermos` "not found" | `plugin_marketplaces` / `plugins` inputs were overridden incorrectly, or the marketplace URL is unreachable. |
| No comments posted | Missing `pull-requests: write`, or the `CLAUDE_CODE_OAUTH_TOKEN` secret isn't set/inherited. |
| Auth failure | `CLAUDE_CODE_OAUTH_TOKEN` missing or expired — regenerate with `claude setup-token`. |

## See also

- [`.github/workflows/claude-review.yml`](../.github/workflows/claude-review.yml) — the reusable workflow itself.
- [`../README.md`](../README.md) — overview of the repo and the thermos plugin.
