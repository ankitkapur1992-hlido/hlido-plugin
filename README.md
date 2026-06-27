# Hlido — Claude Code plugin

**Independent trust checks for AI agents, inside Claude Code.**

[Hlido](https://hlido.eu) is "Rotten Tomatoes for AI agents" — an independent
verification layer that evaluates AI agents on their public surface and
publishes a score (0-100), a tier (VITAL / STEADY / FADING / FLATLINE), and a
claim-by-claim evidence table (PASS / FAIL / UNVERIFIED). Scoring weights are
private; outcomes and evidence are public.

This plugin puts that corpus one command away — and gives Claude live,
machine-readable access to it via MCP, so you can vet an agent *before* you
install it, delegate to it, or build on it.

## What you get

| Surface | Use it for |
|---|---|
| **`/hlido:check`** | Look up one agent's score, tier, dimensions, and top claim verdicts; search the corpus; compare 2 agents; browse a tier. Reads public JSON, zero auth. |
| **`/hlido:recommend "<need>"`** | A vetted, evidence-backed shortlist for a free-text need. Free anonymously (top-1); `HLIDO_API_KEY` unlocks top-k. |
| **MCP server (`hlido`)** | Agent-to-agent tools — `trust_check`, `find_trusted`, `compare_agents`, `verify_claim`, `get_scorecard`, `recommend`, `request_quick_audit`, and more. Claude calls these automatically when a task needs a trust read. |

Natural language works too — "is Cursor trustworthy?", "compare aider and
claude-code", "recommend a CLI pair programmer with strong git support."

## Install

**Install now — straight from this repo (works today, no waiting):**

```
/plugin marketplace add hlido/hlido-plugin
/plugin install hlido@hlido
```

That is it — `/hlido:check`, `/hlido:recommend`, and the MCP trust tools are
live immediately, zero auth.

*Also submitted to the official community marketplace (in review). Once listed
you will additionally be able to:*

```
/plugin marketplace add anthropics/claude-plugins-community
/plugin install hlido@claude-community
```

Or try it locally from a clone of this repo:

```bash
claude --plugin-dir ./hlido-plugin
```


## Optional configuration

- **`HLIDO_API_KEY`** — a Hlido API key (`hlk_live_…`) unlocks top-k results in
  `/hlido:recommend`. Without it, the plugin works anonymously (top-1). Get one
  at <https://hlido.eu/api/>.

The MCP server and `/hlido:check` need no configuration — they read public,
CDN-cached endpoints on `hlido.eu`.

## Privacy & safety

- All requests go to `hlido.eu` over HTTPS. No user content is stored; Hlido
  keeps no per-user state. Anonymous quota is enforced on a hashed IP.
- Outputs are public-safe: the plugin surfaces outcomes and evidence, never the
  private scoring weights / methodology rubric.
- Read-only by default. The only write paths are explicit, opt-in actions you
  trigger (e.g. submitting a new agent for review or reporting a review issue).

## Links

- Website: <https://hlido.eu>
- Reviews index: <https://hlido.eu/reviews/>
- REST API + OpenAPI: <https://hlido.eu/api/>
- CLI (no Claude Code needed): `npx @hlido/cli check <slug>`
- MCP endpoint: `https://hlido.eu/mcp`

## License

MIT © Hlido
