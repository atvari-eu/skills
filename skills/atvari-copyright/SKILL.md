---
name: atvari-copyright
description: "Copyright and license notices for atvari GmbH projects: LICENSE files, README sections, package/site metadata. Use when adding or editing licensing/copyright info in atvari repos (detect via @atvari.eu git email), even if not asked explicitly. Not for per-source-file headers or third-party copyrights."
---

# Copyright Notices for atvari GmbH

Consistent ownership lines make ownership unambiguous and let tooling (GitHub license detection, FOSS scanners) recognize atvari projects automatically. Get the canonical values right:

| Fact | Value |
| --- | --- |
| Legal name | `atvari GmbH` |
| Website | `https://atvari.eu` |
| Default code license | MIT |
| Year | Current year at creation (`date +%Y`); never guess |

## Is this project atvari's?

Before applying these formats, confirm the project belongs to atvari — the strongest signal is the committer identity:

```bash
git config user.email        # active config: repo → global → system
```

An `@atvari.eu` address confirms it. Treat a match as the trigger to apply this skill even when the user didn't mention copyright explicitly. No match doesn't rule the project out (personal emails happen) — fall back to other signals like remote URL (`git remote -v`), directory layout, or existing LICENSE/README content before deciding.

## Standard formats by location

**LICENSE file** — full MIT text with exactly this line:

```
Copyright (c) 2026 atvari GmbH
```

**README license section:**

```markdown
## License

[MIT](LICENSE) © [atvari GmbH](https://atvari.eu)
```

**Site/app config copyright fields** (zola.toml, hugo, package.json, etc.):

```toml
copyright = "Copyright © 2026 atvari GmbH"
```

**Package metadata:** set the license identifier (`license = "MIT"` in Cargo.toml, `license = {text = "MIT"}` or SPDX string in pyproject.toml) alongside any copyright field the format supports.

## "All rights reserved" vs. open source

These two collide — pick based on what the notice covers:

- **MIT-licensed code**: never append "All rights reserved" / "Alle Rechte vorbehalten". The phrase contradicts the license grant and confuses scanners. `Copyright (c) 2026 atvari GmbH` is complete.
- **Proprietary assets** (website copy, design, branding, media): reservation is correct. English: `© 2026 atvari GmbH. All rights reserved.` German-facing outputs use: `© 2026 atvari GmbH. Alle Rechte vorbehalten.`

Match the page's language: German UI → German notice, everything else → English.

## Years

- **New files/works**: current year, single year — no ranges until needed.
- **Editing existing notices**: leave old years alone. Only extend (e.g. `2024-2026`) when adding substantial new protectable content; cosmetic edits don't bump years.
- **Scaffolding**: stamp the current year, not the year you found in a template.

## Boundaries

- Preserve third-party copyright lines verbatim — never merge, reword, or drop them, including inside vendored code and theme templates (a stale `copyright = "...Portio 2020"` in a vendored theme belongs to its author; override it in your own config instead of editing theirs).
- One owner per notice: if a project genuinely has multiple copyright holders, give each its own line, sorted by first year.
- German GmbH websites need an Impressum under German law — that's a separate legal requirement from the copyright line. Point to it if missing; don't generate fake legal data (handelsregister numbers, addresses) yourself.
