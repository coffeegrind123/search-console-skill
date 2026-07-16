# search-console — an Agent Skill for Google Search Console

A [Claude Code](https://claude.com/claude-code) Agent Skill that queries Google
Search Console **directly over its REST API** — `curl` + a service-account JWT.
No MCP server, no npm install, no third-party code holding your credentials.

## Why not an MCP server?

There are ~12 Google Search Console MCP servers on GitHub. The most popular is
genuinely well-built and safe (MIT, no install hooks, no telemetry, credentials
encrypted at rest). I audited it and still didn't install it:

- It wraps **~34 "tools" over four REST endpoints.** Nearly every analytics and
  "SEO insight" tool is one `searchAnalytics/query` call plus post-processing.
  Striking-distance is *"keep rows with position 11–20."* That's the whole trick.
- **The setup burden is identical.** You still create the GCP project, enable the
  API, mint the service account, and add it in Search Console. A server saves the
  agent writing curl. It saves you nothing.
- It adds ~10 npm dependencies and hands a Google credential to third-party code.

So this skill keeps the genuinely valuable part — **the analysis recipes and the
gotchas** — and drops the packaging.

## What's in it

- **Working service-account JWT auth** (the fiddly bit — RS256, `jwt-bearer` grant)
- **The four endpoints:** `sites`, `sitemaps` (list/submit/delete), `searchAnalytics`,
  and URL inspection (which lives on a *different host*)
- **Nine analysis recipes:** striking distance, low-hanging fruit, CTR-vs-position,
  cannibalisation, lost queries, brand vs non-brand, per-locale ROI, country
  reality-check, anomaly attribution
- **The gotchas that cost hours:** 2–3 day data lag, sampling means **totals ≠ the
  sum of rows**, 16-month retention, `Restricted` ≠ `Full`, quota limits, why
  *"Discovered – currently not indexed"* is normal and not a bug, why Google ignores
  IndexNow entirely, and why the Indexing API is not a shortcut
  (it's `JobPosting`/`BroadcastEvent` only — general pages are outside policy)
- **Reporting rules**, including: refuse the verdict on under ~4 weeks of data.
  Sparse data is not evidence of failure; it's absence of evidence.

## Install

```bash
git clone https://github.com/<you>/search-console-skill.git \
  ~/.claude/skills/search-console
```

Then follow the Setup section in `SKILL.md`. It auto-loads when you mention Search
Console, GSC, indexing, sitemaps, or ranking data — or invoke it with
`/search-console`.

## Credentials

The skill reads `~/.gsc_service_account.json` (0600) at call time. **No credential
ever lives in this repo** — `.gitignore` blocks `*.json`, `*.pem`, `*.key` and
`*service_account*` as a backstop.

## License

MIT — see [LICENSE](LICENSE).
