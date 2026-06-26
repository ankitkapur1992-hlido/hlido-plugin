---
name: recommend
description: Get a Hlido-vetted shortlist of AI agents for a buyer's free-text need. Invoke as /hlido:recommend "<need>" [--category <name>] [--k <int>] [--min-tier <tier>], or naturally ("recommend a CLI pair programmer", "best AI agent for outbound sales calls"). Calls https://hlido.eu/v1/recommend (REST). Free tier (no auth) returns top-1; HLIDO_API_KEY env var (Bearer hlk_live_*) unlocks top-k.
when_to_invoke: User describes a capability they need and wants Hlido's filtered, evidence-backed shortlist instead of one-by-one checks. Triggers on "recommend", "shortlist", "find me an agent for X", "top agents for Y", "best AI agent that does Z".
do_not_invoke: When the user already knows the slug — use /hlido:check instead. When they want the FULL claim-by-claim scorecard — that's /hlido:check. When they want to submit a NEW agent — direct them to https://hlido.eu/submit.
license: MIT
---

# /hlido:recommend — buyer-side shortlist via the Hlido Recommendation API

Hlido's recommendation surface is a REST API at `https://hlido.eu/v1`. It
exposes the same review corpus as the bundled MCP server, in a buyer-friendly
shape: pass a free-text need + optional constraints, get back a ranked
shortlist with rationale, evidence URL, and signed scorecard URL.

This skill calls `POST /v1/recommend` directly. No local dependencies; just
HTTP. (For programmatic use, the bundled MCP `recommend`/`find_trusted` tools
do the same thing — this skill is the interactive path.)

## Authentication

| Tier | Auth | Calls | k |
|---|---|---|---|
| Free | none | 100/day per IP | forced 1 |
| Dev (€9/mo) | `Authorization: Bearer hlk_live_…` | 10,000/month | 1-10 |
| Team (€19/mo) | `Authorization: Bearer hlk_live_…` | 100,000/month | 1-25 |

Read the API key from `HLIDO_API_KEY` env var if present. Never prompt the user
for it; if no key is set, call anonymously and surface the `tier_note` the
server returns. Sign up: <https://hlido.eu/api/>

## Invocation

```
/hlido:recommend "<need>" [--category <name>] [--k <int>] [--min-tier <tier>]
```

- `<need>` (required, 3-500 chars) — natural-language description of the
  capability the buyer needs.
- `--category` (optional) — one of the 13 canonical categories: AI Agent,
  Coding, Customer Experience, Workflow & Automation, Infrastructure, Voice,
  Productivity, Image & Design, Frameworks & Eval, Specialized verticals,
  Chat & Companion, Research, Marketing & Content.
- `--k` (optional) — number of results (capped server-side: 10 Dev, 25 Team,
  FORCED 1 Free).
- `--min-tier` (optional) — VITAL | STEADY | FADING | FLATLINE. Free hard-floor
  is FLATLINE; paid default is STEADY.

## Wire format

Always set `User-Agent: hlido-plugin/0.1 (claude-code)` so Hlido can attribute
plugin traffic separately.

```sh
curl -X POST https://hlido.eu/v1/recommend \
     -H "Content-Type: application/json" \
     -H "User-Agent: hlido-plugin/0.1 (claude-code)" \
     ${HLIDO_API_KEY:+-H "Authorization: Bearer $HLIDO_API_KEY"} \
     -d '{
       "need": "code review tool with git integration",
       "constraints": { "category": "Coding", "min_tier": "STEADY" },
       "k": 5
     }'
```

Response shape:
```json
{
  "request_id": "req_<uuid>",
  "model_version": "wave4-v1",
  "tier_note": "Free tier returns top-1. Upgrade for top-k. https://hlido.eu/api/",
  "results": [
    {
      "slug": "aider", "name": "Aider", "tier": "STEADY", "score": 78,
      "category": "Coding",
      "rationale": "Top match on category and summary tokens. Laddoo 78 (STEADY).",
      "evidence_url": "https://hlido.eu/reviews/aider/",
      "score_url": "https://hlido.eu/data/scorecards/aider.json"
    }
  ]
}
```

## Output format

```
## Hlido recommendations for "<need>"

1. **{Name}** ({slug}) — {score}/100 · {TIER}
   {rationale}
   → https://hlido.eu/reviews/{slug}/
...

(Free tier: top-1 only. Upgrade at https://hlido.eu/api/ for top-k.)

request_id: req_…  model: wave4-v1
```

Empty `results` → "No matches under your constraints — try a broader query, or
`/hlido:check` and ask for `tier STEADY` to browse the top tier."

## Errors — degrade gracefully

| Status | error | What to say |
|---|---|---|
| 400 | invalid_body / missing_field | Echo the malformed constraint; suggest the closest valid value. |
| 401 | invalid_api_key | "Your Hlido API key looks malformed. Get a fresh one at https://hlido.eu/api/." |
| 429 | quota_exceeded | "Hlido free tier is 100 calls/day. Upgrade at https://hlido.eu/api/ for 10k/month." |
| 500 | internal_error | "Hlido API temporarily unavailable. Try again in a minute." |

Never invent a recommendation. If the API returns no results, say so.

## What this skill does NOT do

- **Submit new agents** — direct to <https://hlido.eu/submit>.
- **Expose scoring weights / methodology** — private (Hlido's moat).
- **Auto-upgrade** — if quota exceeds, surface the upgrade link; never silently
  switch surface.

## Privacy

Anonymous calls are IP-keyed for daily quota; the IP is hashed at the edge
before persistence — Hlido never stores your raw IP. No user content is
retained beyond the rollup counters.
