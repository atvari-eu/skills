# skills

Common [agent skills](https://skills.sh/) used within atvari, published to the open skills ecosystem.

## Skills

| Skill | Description |
| --- | --- |
| — | No skills published yet. |

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
