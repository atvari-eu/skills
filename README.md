# skills

Common [agent skills](https://skills.sh/) used within atvari, published to the open skills ecosystem.

## Skills

| Skill | Description |
| --- | --- |
| [atvari-copyright](skills/atvari-copyright/SKILL.md) | Copyright and license notices for atvari GmbH projects: LICENSE files, README sections, and package/site metadata, detecting atvari repos via the `@atvari.eu` git email. |
| [auto-system-one](skills/auto-system-one/SKILL.md) | Teaches agents to reach for `jev` decisions (Jev-compatible decision endpoints; default OpenCode Console System One) proactively — screen before reading, verify claims, rank by meaning, gate choices, score rubrics — with skip rules and a hard privacy line. |
| [ci-agents](skills/ci-agents/SKILL.md) | Run Claude Code and OpenCode agents in CI pipelines authenticated with flat-rate subscriptions instead of pay-per-token API keys: workflow YAML for GitHub Actions, GitLab CI, and Forgejo Actions, secrets wiring, and comment-triggered bots. |
| [conventional-commits](skills/conventional-commits/SKILL.md) | Default commit messages to Conventional Commits (`type(scope): description`), including breaking-change footers and repo-precedent awareness. |
| [system-one](skills/system-one/SKILL.md) | How to call a Jev decision endpoint from the terminal — default OpenCode Console System One, others via env overrides — with request shapes for noul / choice / score, the `jev` CLI, thresholds, and privacy guardrails. |

## Installation

Install individual skills via the `skills` CLI:

```console
$ skills add atvari-eu/skills@<skill-name> -g
```

Or via home-manager/NixOS:

```nix
programs.opencode.skills."<skill-name>" = "${inputs.skills}/skills/<skill-name>";
```

## Layout

- `skills/<name>/SKILL.md` — one skill per directory
- `skills/<name>/evals/evals.json` — eval cases with fixtures in `evals/files/`

Skill directory paths are load-bearing — don't rename or restructure existing skill dirs without intent.

## License

[MIT](LICENSE)
