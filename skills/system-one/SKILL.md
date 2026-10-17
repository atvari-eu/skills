---
name: system-one
description: "How to call Jev decision endpoints from the terminal — default OpenCode Console System One, others via env overrides — via curl or the `jev` CLI: request/response shapes for noul / choice / score questions, models, thresholds, and privacy guardrails. Use when writing anything that invokes `jev`, POSTs to a System One endpoint, or wires Jev decisions into a script; also when System One or decision models come up. Not for when-to-call judgment (`auto-system-one`) or app-level design (`typesafe-ai`)."
---

# System One (Jev) decisions from the terminal

System One models return **typed decisions with calibrated probabilities**, not
text. One call: `state` (any messy context) + typed `questions`. Answers are
numbers/labels your code branches on; you keep policy, thresholds, and side
effects in code.

This doc targets OpenCode Console's System One endpoint by default; every
endpoint-specific value is env-overridable, so the same recipe runs against any
Jev-compatible endpoint and from any agent. When to reach for a judgment call
versus your own reading is the `auto-system-one` skill's job (when installed);
this document carries the mechanics.

On OpenCode, the
[opencode-system-one](https://github.com/atvari-eu/opencode-system-one) plugin
makes this skill redundant: it exposes the same contract as native
`jev_decide` / `jev_judge_many` tools with caching and built-in call
discipline, so there is no request to write by hand — use the plugin instead
of installing this skill. Everything below still applies verbatim for other
agents, non-Console endpoints, and for writing integrations.

## Endpoint, auth, model — defaults and overrides

| Variable | Purpose | Default |
| --- | --- | --- |
| `JEV_ENDPOINT` | decision endpoint URL | `https://opencode.ai/inference/systemone/v1/systemone` (OpenCode Console) |
| `JEV_MODEL` | model id | see Models below |

Key and endpoint must come from the same deployment. The standard key name is
`TYPESAFE_API_KEY` — the one omp, the official SDKs, and the agent-skill
runtime use for the official TypeSafe API (current endpoint URL per
[docs.typesafe.ai](https://docs.typesafe.ai)). OpenCode Console-billed keys go
in `OPENCODE_API_KEY` or, on hosts with OpenCode installed, opencode's own
auth store (`~/.local/share/opencode/auth.json`, field `.opencode.key`) — that
last fallback exists nowhere else; other agents export their key env instead
of hardcoding it.

## Calling it

POST JSON to the endpoint with `Authorization: Bearer $key`. Never export the
key or write it anywhere — read it at call time:

```bash
endpoint="${JEV_ENDPOINT:-https://opencode.ai/inference/systemone/v1/systemone}"
key="${TYPESAFE_API_KEY:-${OPENCODE_API_KEY:-$(jq -r '.opencode.key // empty' ~/.local/share/opencode/auth.json)}}"
curl -sS -H "Authorization: Bearer $key" -H 'Content-Type: application/json' \
  -d @request.json "$endpoint"
```

A `jev` CLI, when installed, wraps the same stdin contract for the default
endpoint: it reads a request body **without `model`/auth** on stdin, adds them
(`JEV_MODEL` selects the model), posts to the endpoint, and prints
`{answers, usage}`:

```bash
jq -n --arg state "..." '{state: $state, questions: {priority: {type: "noul", instructions: "..."}}}' | jev
```

Pointing such a CLI at a different endpoint is CLI-specific; the raw recipe
above is the portable path.

## Requests and responses

```jsonc
// request — multiple questions in one request; they run in parallel
{
  "state": "My order arrived late and the item is damaged.",
  "questions": {
    "department": {
      "type": "choice",
      "instructions": "Which team should handle this request?",
      "criteria": {"returns": "...", "shipping": "...", "billing": "..."}
    },
    "urgency": {"type": "noul", "instructions": "Does this need urgent attention?"}
  }
}
// response — answers quoted per question id, usage counts input tokens
{"answers": {"department": {"type": "choice", "choice": "returns", "confidence": 0.93,
  "probabilities": {...}}, "urgency": {"type": "noul", "noul": 0.86}},
 "usage": {"input_tokens": 339, "output_tokens": 50}}
```

| Type | Question | Answer | Notes |
| --- | --- | --- | --- |
| `noul` | yes/no | probability 0..1 | no separate confidence; one per label when several may apply |
| `choice` | one of up to 255 fixed options | winner + per-option distribution + confidence | define criteria precisely |
| `score` | degree on 2–10 ordered levels you write | probability-weighted mean (may land between levels) | levels must describe concrete situations |

Put the judgment in `instructions` and the options in `criteria` with complete
meaning (what belongs, what does not, precedence, example). Question IDs never
reach the model; include full meaning per question. Include a no-match or
manual-review option when nothing may fit instead of forcing a pick.

## Models

- `jev-1.13-free` — free on OpenCode Console for a limited time; fine for
  experiments; the CLI's default via `JEV_MODEL`
- `jev-1.13` — pinned stable; reproducible
- OpenCode Console pricing: `$0.042` per 1M **input** tokens, output free — a
  typical judgment is a fraction of a cent. Other Jev-compatible endpoints
  price independently; check their docs.

## Policy and uncertainty

- Choice/Score `confidence` is distribution peakedness, not accuracy. Use it
  for auto/review routing, tuned against observed outcomes in *your* domain.
- Workable defaults: `auto` at confidence ≥ 0.85 (plus a large winner margin
  for choices), `review` between, escalate to a person or a reasoning model
  below. A noul near 0.5 means yes/no are equally likely, not medium
  intensity.
- Fail closed: a malformed or missing answer must route to review, never to
  auto. Retry idempotent GET-like judgments only after 429/5xx; a retried
  paid call is a new call.
- 401/403 right after an endpoint or key change is a key↔endpoint pairing
  problem, not rate limiting — the two must come from the same deployment.
  Fix the pair; do not retry blindly.
- Keep questions and thresholds as named constants in one place in the
  consuming script; review those, not the plumbing.

## Long states and repeated calls

- Reruns and triage loops often hit the same judgment again. Cache answers
  keyed on the (normalized) `state` plus `questions` and serve identical
  judgments from the cache; new or differing states call as usual. Caches go
  through the same threshold checks as fresh answers — never force auto.
- A handful of small items fit one `state` with per-item question ids. There
  is no multi-state batch endpoint: for larger sets, one request per item
  (same questions, parallel requests); for one long item, chunk it and judge
  each chunk with the unchanged questions. Chunking is the sanctioned
  exception to one-call-per-decision — each chunk is its own judgment.
- Aggregate piecewise results in code, not by another model call:
  probability-weighted mean, ordering, or reference pick across the answer
  series; flag chunks with low confidence for review.

## Guardrails (hard rules)

- **Never send personal data as `state`**: no health, financial, credential,
  contact, or location data from the user's own notes or files. Decide
  triage/salvage/routing categories without quoting sensitive content.
- Never include API keys, tokens, or contents of `.env` and other secret
  files in `state` or `criteria` — even partially, even "just as context".
- Judgments are advisory: the calling code (or the agent's approval flow)
  enforces every action. Jev does not execute anything.

## Deeper design help

Use the companion `typesafe-ai` skill (and its live docs:
[docs.typesafe.ai](https://docs.typesafe.ai)) for question design,
composition patterns, cookbooks, and SDK usage beyond one-off terminal calls.
