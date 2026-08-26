---
name: ci-agents
description: "Run Claude Code and OpenCode (Zen / Go models) in CI pipelines authenticated with flat-rate subscriptions instead of pay-per-token API keys. Use whenever adding or fixing agent automation in GitHub Actions, GitLab CI, Forgejo Actions, or any other CI: writing workflow YAML (.github/workflows, .gitlab-ci.yml), wiring up secrets and CI variables (CLAUDE_CODE_OAUTH_TOKEN, OPENCODE_API_KEY, `gh secret set`, `opencode github install`, `/install-github-app`), comment-triggered bots (@claude, /oc, /opencode, @opencode), scheduled agent jobs, or debugging headless auth / rate-limit / token-expiry failures. Not for local interactive login, MCP server setup, or authoring prompts/skills for the agents themselves."
---

# AI Coding Agents in CI on Subscriptions

Run Claude Code and OpenCode in pipelines without pay-per-token API keys. Flat subscriptions cover the model cost; CI only needs one long-lived credential per tool. Pick the credential that matches the workload:

| Workload | Tool & auth | Credential |
| --- | --- | --- |
| Claude models (PR review, fixes, triage) | Claude Code with a Claude Pro/Max/Team/Enterprise subscription | `CLAUDE_CODE_OAUTH_TOKEN` |
| Open models (GLM, Kimi, MiniMax…) via OpenCode | OpenCode Go ($10/mo) or Zen gateway | `OPENCODE_API_KEY` |

Two ground rules shape everything below:

- **Only Claude Code has a sanctioned subscription path for CI.** Anthropic's ToS (Feb 2026) restricts subscription tokens to official clients, and OpenCode removed built-in Claude Pro/Max OAuth at Anthropic's request. Route Claude-subscription work through Claude Code; run OpenCode against its own Go/Zen keys. See [Claude OAuth inside OpenCode](#claude-oauth-inside-opencode).
- **Subscription tokens are personal.** A `claude setup-token` bills one person's rate limits — every parallel job draws from the same 5-hour/weekly windows. Keep agent jobs few, small, and serialized where possible.

## Claude Code via subscription

### 1. Generate the token (once, locally)

```bash
claude setup-token
```

Opens the browser login flow, then prints a ~1-year OAuth token (`sk-ant-oat01-…`). It is not saved anywhere — copy it straight into your CI secret store. Requires Pro, Max, Team, or Enterprise.

The token can only make model requests: no Remote Control sessions, no claude.ai connectors. Local MCP servers still work.

### 2a. GitHub Actions — quick setup

In a local `claude` session inside the target repo:

```
/install-github-app
```

This installs the Claude GitHub App, stores the secret (`CLAUDE_CODE_OAUTH_TOKEN` when you pick the subscription option), and opens a PR with ready-made workflows. Test by commenting `@claude` on an issue or PR.

### 2b. GitHub Actions — manual

1. Install the app: <https://github.com/apps/claude> (needs Contents, Issues, Pull requests read/write).
2. Add the secret:
   ```bash
   gh secret set CLAUDE_CODE_OAUTH_TOKEN   # paste token when prompted
   ```
3. Add the workflow:

```yaml
# .github/workflows/claude.yml
name: claude
on:
  issue_comment: { types: [created] }
  pull_request_review_comment: { types: [created] }

jobs:
  claude:
    if: contains(github.event.comment.body, '@claude')
    runs-on: ubuntu-latest
    timeout-minutes: 20
    permissions:
      contents: write
      pull-requests: write
      issues: write
      actions: read        # lets Claude read CI results on the PR
      id-token: write      # required for the action's GitHub App auth
    steps:
      - uses: actions/checkout@v6
        with: { fetch-depth: 1 }
      - uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          # prompt: ...           # omit → responds to @claude mentions only
```

For unattended runs (schedule, `pull_request`, `issues`), add a `prompt:` input and matching event triggers.

### 2c. Generic CI / plain CLI job

Any runner that can install Node or download the binary works. The env var alone authenticates it:

```yaml
# works in GitLab CI, Forgejo Actions, Jenkins, …
# expose CLAUDE_CODE_OAUTH_TOKEN from your CI secret store as an env var first
script:
  - curl -fsSL https://claude.ai/install.sh | bash
  - claude -p "Review this diff for security issues" --output-format json
```

For jobs that edit files without human approval, add `--dangerously-skip-permissions` (or `--permission-mode acceptEdits`) — headless runs have no one to answer permission prompts. Caveat: Claude Code's `--bare` mode does **not** read `CLAUDE_CODE_OAUTH_TOKEN`; stick to normal mode.

## OpenCode via Go / Zen

OpenCode Go and Zen are OpenCode's own flat-rate offerings. Both read the same env var; they differ in provider id and endpoint:

| Plan | Provider id | Endpoint | Env var |
| --- | --- | --- | --- |
| Zen (gateway, curated frontier models) | `opencode/<model>` | `https://opencode.ai/zen/v1` | `OPENCODE_API_KEY` |
| Go ($10/mo open models) | `opencode-go/<model>` | `https://opencode.ai/zen/go/v1` | `OPENCODE_API_KEY` |

Get the key from <https://opencode.ai/auth> (subscribe to Go there too). It's a static API key — no token-refresh machinery needed in CI, unlike Claude's OAuth token.

Headless execution is just `opencode run "<prompt>" --model opencode-go/kimi-k2.5` with `OPENCODE_API_KEY` set — it exits when done.

### GitHub Actions

Quick setup — walks through installing the GitHub app, creating the workflow, and setting up secrets:

```bash
opencode github install
```

Manual alternative: install <https://github.com/apps/opencode-agent>, add `OPENCODE_API_KEY` as an Actions secret, then:

```yaml
# .github/workflows/opencode.yml
name: opencode
on:
  issue_comment: { types: [created] }
  pull_request_review_comment: { types: [created] }

jobs:
  opencode:
    if: contains(github.event.comment.body, '/oc') || contains(github.event.comment.body, '/opencode')
    runs-on: ubuntu-latest
    timeout-minutes: 20
    permissions:
      id-token: write      # exchanges OIDC for an OpenCode App installation token
    steps:
      - uses: actions/checkout@v6
        with: { fetch-depth: 1, persist-credentials: false }
      - name: Run OpenCode
        uses: anomalyco/opencode/github@latest
        env:
          OPENCODE_API_KEY: ${{ secrets.OPENCODE_API_KEY }}
        with:
          model: opencode-go/glm-5
          # share: false   # keep session private; defaults true on public repos
```

No OpenCode GitHub App? Set `use_github_token: true` and grant `contents/pull-requests/issues: write` plus `GITHUB_TOKEN` instead — the OIDC exchange (`id-token: write`) is skipped.

Scheduled/automatic events require a `prompt:` input and, if OpenCode should open branches/PRs, `contents: write` + `pull-requests: write`.

### GitLab CI

Either store `OPENCODE_API_KEY` as a masked variable and drive the CLI yourself, or use the community component referenced by the official docs ([nagyv/gitlab-opencode](https://gitlab.com/nagyv/gitlab-opencode)), which takes the full auth JSON as a File-type variable:

```yaml
# .gitlab-ci.yml
include:
  - component: $CI_SERVER_FQDN/nagyv/gitlab-opencode/opencode@2
    inputs:
      auth_json: $OPENCODE_AUTH_JSON   # File-type CI variable, masked
      message: "Review this merge request"
```

Variables live under **Settings → CI/CD → Variables**; mark credentials masked. Or skip the component entirely:

```yaml
opencode-job:
  image: node:22-slim
  script:
    - npm install --global opencode-ai
    - opencode run "Summarize changes vs $CI_MERGE_REQUEST_DIFF_BASE_SHA" --model opencode-go/glm-5
```

### Forgejo / Gitea Actions and other CI

No first-party integration exists (the GitHub action depends on GitHub's OIDC/App infrastructure), so use the generic recipe: put `OPENCODE_API_KEY` in the repo's Actions secrets (Forgejo/Gitea: repo **Settings → Secrets**), install opencode in the job, and call `opencode run`. Same for any system with a shell and a secret store.

```yaml
steps:
  - curl -fsSL https://opencode.ai/install | bash
  - opencode run "Triage failing test in pkg/api" --model opencode-go/kimi-k2.5
```

## Claude OAuth inside OpenCode

Don't build on this. OpenCode dropped built-in Claude Pro/Max OAuth after Anthropic's legal request (March 2026); maintainers state they cannot support it, and Anthropic now validates client identity on subscription traffic. Community plugins and hand-written `auth.json` entries (`{"anthropic":{"type":"oauth",...}}`, injectable via the `OPENCODE_AUTH_CONTENT` env var) exist but violate the spirit of the ToS restriction and break without warning when Anthropic tightens checks. If someone asks for this setup in CI, steer them to the supported split instead: **Claude Code action for Claude-model workloads, OpenCode with its own Go/Zen key for open models** — both covered above.

## Finishing setup checklist

After committing workflow files, finish the pipeline variables — the slash-command flows handle most of this automatically:

**GitHub** (`gh` authenticated):
```bash
gh secret set CLAUDE_CODE_OAUTH_TOKEN     # Claude Code path
gh secret set OPENCODE_API_KEY            # OpenCode path
```
Or rerun `/install-github-app` (Claude Code) / `opencode github install` (OpenCode) — both idempotently install their apps and refresh secrets. Verify the respective GitHub App (github.com/apps/claude, github.com/apps/opencode-agent) is enabled for the repo.

**GitLab**: project **Settings → CI/CD → Variables** → add `OPENCODE_API_KEY` (masked) or the auth JSON (File type, masked). For Duo-style flows also set `GITLAB_TOKEN_OPENCODE`.

**Forgejo/Gitea**: repo **Settings → Actions → Secrets** (or org-level equivalent).

Then trigger once end-to-end before relying on it: comment `@claude` / `/oc`, or dispatch the scheduled job manually via `workflow_dispatch` / **Run pipeline**.

## Gotchas that quietly break pipelines

- **Agent commits don't retrigger CI.** Commits pushed with the default `GITHUB_TOKEN` never start new workflow runs. If the agent's pushes must kick off your test suite, authenticate pushes with the Claude/OpenCode GitHub App tokens or a custom App token (`actions/create-github-app-token@v2`) instead.
- **Never combine `pull_request_target` with agent checkouts.** It checks out base-repo code while exposing secrets to PR-controlled content — a privilege-escalation primitive. Trigger reviews on `pull_request` (forks get no secrets, which is correct) and gate comment triggers on write access (both official actions do this by default).
- **Pin versions and cap runtime**: pin actions to major tags (`@v1`, `@v2`), set `timeout-minutes`, and keep prompts scoped to one task. Subscription rate limits make runaway agent loops expensive in wall-clock time even though they're free in dollars.
- **`share: true` publishes the session transcript.** On public repos the OpenCode action shares by default — disable it when prompts may embed private code or secrets.
- **Token rotation**: `claude setup-token` tokens die after ~1 year. Record the expiry where the team will see it; expired tokens fail every job with auth errors.
