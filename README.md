# chappie-actions

Public companion repo for cross-org Chappie consumers. Contains **only** reusable GitHub Actions workflows and this README.

Private source, agent packs, and runtime logic live in the [`BuildCircle/chappie`](https://github.com/BuildCircle/chappie) monorepo and ship as the public GHCR image `ghcr.io/buildcircle/chappie-cli:v1`.

## Pinning policy

Consumer stub workflows must pin a **version tag**, not `@main`:

```yaml
uses: BuildCircle/chappie-actions/.github/workflows/agent.yml@v1
```

When a new major runtime is published, bump the tag in your stub deliberately.

## Required inputs

| Input | Description |
|---|---|
| `prompt` | Natural language task for the agent |
| `client` | Client slug registered in the runtime pack |
| `project` | Project slug registered in the runtime pack |
| `profile` | Agent profile slug (default `develop`) |
| `vendor` | Agent vendor: `cursor`, `claude`, `codex`, `openai`, or `gemini` |
| `triggerRef` | Optional issue number or external ticket reference |
| `branch` | Optional existing branch for PR-steering mode |
| `clientOverlayPath` | Optional path in your repo to a private overlay tree |

## Required secrets

Store these as org-level or environment-scoped Actions secrets on the consumer repo:

| Secret | When |
|---|---|
| `CHAPPIE_APP_ID` | Always. chappie GitHub App id |
| `CHAPPIE_APP_PRIVATE_KEY` | Always. Stub mints a short-lived installation token per run |
| `CURSOR_API_KEY` | When `vendor: cursor` |
| `ANTHROPIC_API_KEY` | When `vendor: claude` |
| `OPENAI_API_KEY` | When `vendor: openai` or `vendor: codex` |
| `GOOGLE_API_KEY` | When `vendor: gemini` |

Optional Journix secrets (`JOURNIX_API_URL`, `JOURNIX_BASIC_USER`, `JOURNIX_BASIC_PASSWORD`) enable best-effort journal push after successful runs.

The reusable `agent.yml` contract still accepts `CHAPPIE_PUSH_TOKEN`. Caller stubs mint an installation token from the App credentials and pass it into that slot. Do not store a long-lived installation token or human PAT as `CHAPPIE_PUSH_TOKEN`.

App permissions and the bot login slug are documented in [`BuildCircle/chappie` docs/auth.md](https://github.com/BuildCircle/chappie/blob/main/docs/auth.md).

## Stub example (GitHub Issues)

```yaml
name: Agent
on:
  issue_comment:
    types: [created]
jobs:
  mint:
    if: |
      github.event.issue.pull_request == null
      && startsWith(github.event.comment.body, '/agent')
    runs-on: ubuntu-latest
    timeout-minutes: 2
    outputs:
      token: ${{ steps.app.outputs.token }}
    steps:
      - id: app
        uses: actions/create-github-app-token@v2
        with:
          app-id: ${{ secrets.CHAPPIE_APP_ID }}
          private-key: ${{ secrets.CHAPPIE_APP_PRIVATE_KEY }}

  agent:
    needs: mint
    if: |
      github.event.issue.pull_request == null
      && startsWith(github.event.comment.body, '/agent')
    uses: BuildCircle/chappie-actions/.github/workflows/agent.yml@v1
    with:
      prompt: ${{ github.event.comment.body }}
      triggerRef: ${{ github.event.issue.number }}
      vendor: claude
      client: your-client-slug
      project: your-project-slug
      # profile: develop  # optional; defaults to develop
    secrets:
      CHAPPIE_PUSH_TOKEN: ${{ needs.mint.outputs.token }}
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

## Registration

Your repository must be registered in the runtime pack before bootstrap succeeds. Contact BuildCircle to add your repo; unregistered repos fail fast during bootstrap with a clear error.

## What is not in this repo

- No application source code
- No agent rules, skills, hooks, or profiles
- No customer names or org allowlists
- No pack-read tokens required for MVP

Runtime work executes inside the pinned GHCR image built from private source.
