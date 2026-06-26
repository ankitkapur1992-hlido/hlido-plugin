---
name: check
description: Look up an AI agent's Hlido evaluation — score, tier, claim verdicts, evidence. Invoke as /hlido:check <slug>, or naturally ("is Cursor trustworthy?", "what's Aider's Hlido score?", "compare cursor and claude-code", "show all VITAL agents"). Reads public hlido.eu JSON; no auth, env vars, or local data required.
when_to_invoke: User asks about an AI agent's trust score, Hlido rating, claim verdicts, or wants to compare agents or browse a tier. Triggers on "is X verified", "what's X's hlido score", "find me a trusted code agent", "compare cursor and claude code", "show all VITAL agents", or /hlido:check. Lightweight read-only lookup against public hlido.eu surfaces — safe to invoke anytime.
do_not_invoke: When the user wants to SUBMIT a new agent for review (use the form at https://hlido.eu/submit or the bundled MCP submit_agent tool). When the user wants the methodology / scoring weights (those are private — Hlido's moat). When asked about non-AI software (Hlido only covers AI agents).
license: MIT
---

# /hlido:check — look up an AI agent's Hlido evaluation

Hlido (https://hlido.eu) is the independent verification layer for the AI agent
ecosystem — "Rotten Tomatoes for AI agents". Every agent on Hlido has been put
through a public-surface evaluation that produces a score (0-100), a tier
(VITAL / STEADY / FADING / FLATLINE), per-claim verdicts (PASS / FAIL /
UNVERIFIED) with linked evidence, and four dimension scores (Strategic Alpha,
Craft & Soul, Execution Grit, Value Signal).

This skill pulls that evaluation inline. No auth, no env vars, no local files.
It hits public JSON endpoints on hlido.eu.

> **Two ways to query Hlido in this plugin:** this skill is the fast,
> zero-auth *interactive* path (reads static CDN-cached JSON). For
> *programmatic / agent-to-agent* use the bundled **MCP server** (`hlido`) —
> tools like `trust_check`, `find_trusted`, `compare_agents`, `verify_claim`,
> `get_scorecard`, `recommend`, `request_quick_audit`. Same corpus, different
> surface. Use whichever fits.

## Supported lookups

| Intent | What it does |
|---|---|
| **check** `<slug>` | Full scorecard for one agent: score, tier, summary, top 5 claims with verdict + evidence, links to full review and attestation. |
| **search** `<free-text>` | Top 5 agents whose name, category, or summary matches the query. Returns slug, name, score, tier, one-line summary. |
| **compare** `<slug1> <slug2>` | Side-by-side: scores, tiers, dimension scores, headline differences in claim verdicts. |
| **tier** `<tier-name>` | All agents in a tier band: `VITAL` (≥90), `STEADY` (≥70), `FADING` (≥40), `FLATLINE` (<40). Case-insensitive. |

Invoked explicitly as `/hlido:check <slug>`, or pick the right intent from the
user's natural-language request (e.g. "compare cursor and aider" → compare).
If the request is bare ("/hlido:check" with no args), list the four intents and
ask which they want.

## Where to read data

Hit these public URLs (HTTPS, no auth, JSON responses, `Access-Control-Allow-Origin: *`):

| Purpose | URL pattern | Notes |
|---|---|---|
| Registry index | `https://hlido.eu/data/review-registry.json` | Source of truth for every published review. Top-level `items[]` array. Use for `search`/`tier` and to look up a slug's `score`, `tier`, `category`, `summary`, `publishedAt`, `detailUrl`. |
| Per-slug scorecard | `https://hlido.eu/data/scorecards/{slug}.json` | Detailed scorecard: `dimensions{}` (4 editorial dimensions), `claims[]` (each `id`, `task`, `verdict`, `evidence`), `verdict_summary`, `run_id`, `signed_at`. |
| Per-slug attestation | `https://hlido.eu/data/attestations/{slug}.json` | Canonical signed reference — minimal, cryptographically-verifiable subset. Cite as the proof anchor. May 404 for older slugs; degrade gracefully. |

**Do NOT** scrape `https://hlido.eu/reviews/{slug}/` HTML — the JSON above
contains everything the page renders and is faster + more reliable.

## Resolving a slug

Slugs are kebab-case ASCII (e.g. `cursor`, `claude-code`, `aider`, `gumloop`).
When the user types `Cursor` or `"Claude Code"`, normalize: lowercase → replace
spaces/punctuation with `-` → strip leading/trailing `-`. Then look up in the
registry. If not found, fuzzy-match (see below).

## Output format — check `<slug>`

```
## {Name} — {score}/100 · {TIER}

> {one-sentence summary from registry.summary or scorecard.verdict_summary}

**Category:** {category} · **Published:** {publishedAt} · **Run:** `{run_id}`

### Dimensions
- Strategic Alpha: {sa}/100
- Craft & Soul: {cs}/100
- Execution Grit: {eg}/100
- Value Signal: {vs}/100

### Top claims (5 of {N})
- ✓ **PASS** — {claim.task}. {one-line evidence excerpt}
- ✗ **FAIL** — {claim.task}. {one-line evidence excerpt}
- ? **UNVERIFIED** — {claim.task}. {one-line reason}
...

### Links
- Full review: https://hlido.eu/reviews/{slug}/
- Signed attestation: https://hlido.eu/data/attestations/{slug}.json
- Raw scorecard: https://hlido.eu/data/scorecards/{slug}.json
```

Pick the 5 most-informative claims (prefer FAIL > PASS > UNVERIFIED — FAILs are
most decision-relevant). Fewer than 5 claims → show all. If the attestation URL
404s, drop that line silently.

## Output format — search `<free-text>`

```
## Search: "{query}" — {N} matches

1. **{name}** ({slug}) — {score}/100 · {TIER}
   {one-line summary}
   → https://hlido.eu/reviews/{slug}/
...
```

Match: case-insensitive substring on `name`, `category`, `summary`. Rank:
exact-name > name-starts-with > name-contains > category > summary. Cap at 5.
0 matches → suggest a broader term or `tier STEADY` to browse.

## Output format — compare `<a> <b>`

```
## {Name A} vs {Name B}

|                  | {A}             | {B}             |
|------------------|-----------------|-----------------|
| Score            | {a_score}/100   | {b_score}/100   |
| Tier             | {a_tier}        | {b_tier}        |
| Category         | {a_cat}         | {b_cat}         |
| Strategic Alpha  | {a_sa}          | {b_sa}          |
| Craft & Soul     | {a_cs}          | {b_cs}          |
| Execution Grit   | {a_eg}          | {b_eg}          |
| Value Signal     | {a_vs}          | {b_vs}          |
| Published        | {a_date}        | {b_date}        |

### Headline differences
- {short bullet}
...

Full reviews: {A_url} · {B_url}
```

One slug missing → say which, offer 3 fuzzy matches before bailing.

## Output format — tier `<tier>`

```
## {TIER} agents — {count}

(score ≥ {threshold})

1. **{name}** ({slug}) — {score}/100
   {one-line summary}
   → https://hlido.eu/reviews/{slug}/
...
```

Sort by score desc. Cap at 25 (hint to filter by category if more).

## If a slug is not found

Suggest 3 closest matches by name-token similarity: tokenize input (split on
`-`/spaces, lowercase) → score each item by overlapping tokens against `slug`
and `name` → top 3 by overlap, ties broken by score.

```
No agent found for slug `cursr`. Did you mean:
- **cursor** (cursor) — 87/100, STEADY
- **codex-cli** (codex-cli) — 72/100, STEADY
- **roo-code** (roo-code) — 68/100, FADING

Try: /hlido:check cursor
```

## Tier thresholds (fixed — don't recompute)

```
VITAL    score ≥ 90
STEADY   score ≥ 70
FADING   score ≥ 40
FLATLINE score <  40
```

The registry's `tier` field is authoritative — don't re-derive from score.

## Caching

Fetch fresh each invocation — Cloudflare CDN caches edge-side and users expect
current data. Don't persist across calls.

## Error handling — degrade gracefully

| Failure | Response |
|---|---|
| Registry fetch 5xx / network | "Hlido registry temporarily unavailable. Try again shortly, or browse https://hlido.eu/reviews/." |
| Scorecard 404 for a slug in the registry | Show registry-level info + note "Detailed scorecard not available for this slug." |
| Attestation 404 | Drop the attestation link silently. |
| Malformed JSON | "Unexpected response from Hlido — please report at https://hlido.eu/submit." |

Never invent data. If a field is missing, say so.

## What this skill does NOT do

- **Submit new agents** — point to https://hlido.eu/submit (or the MCP `submit_agent` tool).
- **Expose scoring weights / methodology** — private (Hlido's moat). Shows outcomes + evidence only.
- **Modify data** — strictly read-only.
- **Cache across invocations** — fresh fetch each time.

## Privacy + safety

All requests go to `hlido.eu` over HTTPS. No user data is sent; Hlido has no
per-user state. Registry/scorecard JSON is already public-safe (no weights, no
internal IDs beyond the public `run_id`).

## Related surfaces

- **CLI without Claude Code:** `npx @hlido/cli check <slug>` (also `search`, `compare`, `tier`) wraps the same endpoints.
- **REST API:** `https://hlido.eu/api/` (human landing) · `https://hlido.eu/v1/openapi.json` (spec) — recommendation, incident registry, RSS feed.
- **MCP server:** bundled in this plugin (`hlido`) for agent-to-agent integration.
