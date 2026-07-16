---
name: search-console
description: >-
  Query Google Search Console from the terminal via its REST API — no MCP server,
  no dependencies, just curl + a service-account JWT. Submit and check sitemaps,
  list verified properties, inspect a URL's index status, and run the search-analytics
  queries that answer real questions: which pages/queries/countries/locales actually
  get impressions, what is stuck on page 2 ("striking distance"), which pages
  cannibalise each other, what traffic was lost, and where CTR is underperforming
  its position. Use when the user asks about Search Console, GSC, Google indexing,
  "is my sitemap submitted", "is X indexed", impressions/clicks/CTR/position data,
  which locale or page is ranking, or whether an SEO change actually worked.
when_to_use: >-
  Use when the user mentions Google Search Console / GSC / Webmaster Tools, asks
  whether pages are indexed, wants sitemap status or submission, wants ranking or
  impression data, or asks whether an SEO change paid off. Examples: "is my sitemap
  submitted?", "is /maps/de_dust2 indexed?", "did the Turkish locale get any
  traffic?", "what's stuck on page 2?", "which pages are cannibalising each other?",
  "check Search Console". Do NOT use for Bing (that is IndexNow — see
  IndexNow — a separate API) or for on-page audits (use the
  seo-audit skills).
allowed-tools: Bash(curl:*) Bash(python3:*) Read Write
---

# Google Search Console (direct REST API)

Everything here is `curl` + a service-account JWT. **No MCP server, no npm install.**
It follows the same convention as every other credential in this setup: one 0600
file in `$HOME`, outside any repo, never committed, read at call time by curl.

**Why not an MCP server:** the popular ones (e.g. `saurabhsharma2u/search-console-mcp`,
216★, MIT — audited 2026-07-16, genuinely safe) wrap ~34 "tools" over **four**
REST endpoints. Their setup burden is identical (you still create the GCP project
and service account), they add ~10 npm dependencies, and the audited one declares
only `webmasters.readonly` while shipping a `sitemaps.submit` tool — which 403s at
runtime. Curl does the same job with no trust surface. If you ever DO want a server,
that one is the pick, and read its source first.

## Setup

You need one thing: a **service-account JSON key** at `~/.gsc_service_account.json`
(0600), whose email has been added to your Search Console property.

1. [console.cloud.google.com](https://console.cloud.google.com) → project → **APIs &
   Services → Library** → enable **"Google Search Console API"**
   (`searchconsole.googleapis.com`).
2. **IAM & Admin → Service Accounts → Create.** Skip the "grant access" step — GCP
   roles are irrelevant here. Then **Keys → Add key → JSON**.
3. **Search Console → Settings → Users and permissions → Add user** → the service
   account's email → **Full**.
   **This is the step everyone misses.** Nothing in GCP grants Search Console access;
   only this does. And it must be **Full** — `Restricted` is read-only and sitemap
   submission will 403.
4. `mv <key>.json ~/.gsc_service_account.json && chmod 600 ~/.gsc_service_account.json`

**The API cannot verify a property, and cannot add one.** Both are UI-only (the Site
Verification API needs an already-verified owner to call it). They're ~3 clicks. If
verification asks for a DNS TXT and your zone is on Cloudflare, add it via the
Cloudflare API rather than by hand.

**Confirm it works before anything else** — `GET /sites` (below) tests auth *and*
access in one call, and its failure mode tells you which of the two is broken.

## Auth: service-account JWT → access token

Service accounts use JWT-bearer, not the OAuth device flow. Access tokens last 3600s;
mint one per session. This helper is the basis of every call below.

```bash
gsc_token() {
  python3 - <<'PY'
import json, time, base64, urllib.request, pathlib
k = json.loads(pathlib.Path.home().joinpath('.gsc_service_account.json').read_text())
def b64(d): return base64.urlsafe_b64encode(json.dumps(d, separators=(',', ':')).encode()).rstrip(b'=')
now = int(time.time())
hdr = b64({"alg": "RS256", "typ": "JWT"})
# webmasters (full) covers sitemap submit; drop to .readonly if the SA is Restricted.
pay = b64({"iss": k["client_email"], "scope": "https://www.googleapis.com/auth/webmasters",
           "aud": "https://oauth2.googleapis.com/token", "iat": now, "exp": now + 3600})
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding
key = serialization.load_pem_private_key(k["private_key"].encode(), password=None)
sig = base64.urlsafe_b64encode(key.sign(hdr + b"." + pay, padding.PKCS1v15(), hashes.SHA256())).rstrip(b'=')
req = urllib.request.Request("https://oauth2.googleapis.com/token",
    data=urllib.parse.urlencode({"grant_type": "urn:ietf:params:oauth:grant-type:jwt-bearer",
                                 "assertion": (hdr + b"." + pay + b"." + sig).decode()}).encode())
print(json.load(urllib.request.urlopen(req))["access_token"])
PY
}
TOKEN=$(gsc_token)
```
Needs `python3 -m pip install cryptography` (usually present). If it isn't and you
can't install, fall back to `pip install google-auth` and mint via
`google.oauth2.service_account.Credentials.from_service_account_file(...).token`.

**siteUrl format matters and is the #1 source of 403s:**
- Domain property → `sc-domain:example.com`
- URL-prefix property → `https://example.com/` (**trailing slash required**)

URL-encode it in paths: `sc-domain%3Aexample.com`.

## The four endpoints (everything else is post-processing)

```bash
API=https://www.googleapis.com/webmasters/v3
S=sc-domain%3Aexample.com
AUTH="Authorization: Bearer $TOKEN"

# 1. Which properties can this service account see? (run FIRST — proves auth + access)
curl -s -H "$AUTH" "$API/sites"

# 2. Sitemaps: list / status / submit / delete
curl -s -H "$AUTH" "$API/sites/$S/sitemaps"
curl -s -H "$AUTH" "$API/sites/$S/sitemaps/https%3A%2F%2Fexample.com%2Fsitemap.xml"
curl -s -X PUT -H "$AUTH" "$API/sites/$S/sitemaps/https%3A%2F%2Fexample.com%2Fsitemap.xml"   # submit
curl -s -X DELETE -H "$AUTH" "$API/sites/$S/sitemaps/<encoded>"

# 3. Search analytics — THE workhorse. Everything interesting is a filter over this.
curl -s -X POST -H "$AUTH" -H 'Content-Type: application/json' \
  "$API/sites/$S/searchAnalytics/query" -d '{
    "startDate":"2026-06-01","endDate":"2026-06-30",
    "dimensions":["page","query"],
    "rowLimit":25000,"dataState":"final"
  }'

# 4. URL inspection (different host + version!)
curl -s -X POST -H "$AUTH" -H 'Content-Type: application/json' \
  https://searchconsole.googleapis.com/v1/urlInspection/index:inspect \
  -d '{"inspectionUrl":"https://example.com/some/page",
       "siteUrl":"sc-domain:example.com"}'
```

`searchAnalytics` dimensions: `date`, `page`, `query`, `country`, `device`,
`searchAppearance`. Combine up to ~4. `type`: `web`|`image`|`video`|`news`|`discover`.
`dataState`: `final` (settled) or `all` (includes fresh, incomplete).

## Analysis recipes (what the 34 MCP "tools" actually are)

All of these are one `searchAnalytics` call + post-processing. Nothing more.

| Recipe | How |
|---|---|
| **Striking distance** | dimensions `["query","page"]`, keep rows with `position` 11–20. These rank page 2 — the cheapest wins on the site. |
| **Low-hanging fruit / quick wins** | position 5–15 **and** high impressions. Small rank gains here move real traffic. |
| **Low CTR for position** | position ≤ 10 but `ctr` well under the expected curve (~28% @1, ~15% @2, ~10% @3, ~5% @5, ~2% @10). A miss here is a **title/description** problem, not a ranking one. |
| **Cannibalisation** | dimensions `["query","page"]`, group by query, flag queries where **2+ pages** have impressions. Two of your pages competing for one query. |
| **Lost queries** | run two windows, diff by query. Present in the old window, absent/collapsed in the new = regression. |
| **Brand vs non-brand** | partition queries on the brand term (e.g. your brand name). Non-brand growth is the real SEO signal; brand growth is just awareness. |
| **Per-locale ROI** | dimensions `["page"]`, bucket by `/tr/`, `/pl/`, `/de/`… prefix. **This answers whether a translated locale earns its keep.** |
| **Country reality-check** | dimensions `["country"]` — does the locale you invested in match where demand actually is? |
| **Anomaly / drop attribution** | dimensions `["date"]` for the series; when it drops, re-query with `["page"]` or `["query"]` across the two windows to attribute it. |

**Expected-CTR figures above are rules of thumb, not measurements.** Use them to rank
candidates, never to state "we are underperforming by X%" as fact.

## Gotchas that will bite you

- **Data lags 2–3 days.** Today's numbers are always empty. `dataState:"final"` omits
  the freshest ~3 days; `"all"` includes them but they will change under you.
- **Row limits + sampling.** `rowLimit` max 25000; page with `startRow`. Long-tail
  queries are omitted entirely for privacy — **totals will NOT equal the sum of rows.**
  Never present a summed total as authoritative.
- **16-month retention.** Older data is gone. Export if you need a baseline.
- **`Restricted` user = read-only.** Sitemap submit returns 403. Needs **Full**.
- **403 `User does not have sufficient permission`** almost always means the *service
  account email* was never added under Settings → Users, or the `siteUrl` form is wrong
  (`sc-domain:` vs `https://…/`), not that the key is bad.
- **URL inspection is a different host** (`searchconsole.googleapis.com/v1`) and is
  quota-limited (~2k/day, 600/min). Do not loop it over thousands of URLs.
- **"Discovered – currently not indexed" is normal** on a large new site. It is not a
  bug and not something to fix by resubmitting. Google indexes selectively.
- **Google ignores IndexNow entirely.** That is Bing/DDG/Yandex/Seznam only. Sitemap +
  crawl is the only Google path. (IndexNow is a separate, trivial POST — worth having alongside this.)
- **The Indexing API is not a shortcut.** Google restricts it to `JobPosting` /
  `BroadcastEvent`. Using it for general pages is outside policy — do not suggest it.

## Worked example: a multi-locale content silo

A concrete shape to copy — a games site with ~2,900 URLs, 12 locales, and several
content silos. The point isn't the site; it's the questions worth asking of one.

- **Locales** at `/`, `/es/`, `/de/`, `/tr/`, … The per-locale ROI recipe is the
  reason this skill exists. Translating a silo by hand costs ~3,000 words per
  language; that's a **hypothesis**, and search analytics is the only thing that
  settles it. A locale indexed but showing ~zero impressions after 4+ weeks is
  evidence to cut — and locales are usually independently removable.
- **Two content tiers.** Curated pages (hand-written, all locales) vs an auto tier
  (templated prose + one genuinely unique asset per page, English only). **Watch the
  auto tier for "Duplicate without user-selected canonical" or mass "Discovered –
  currently not indexed."** That is the honest verdict on whether templated pages
  earned their place, and it is the number one thing to check on any large generated
  silo.
- **Segment by silo prefix** (`/guides/`, `/weapons/`, landing pages, trust pages)
  before drawing conclusions. Sitewide impressions rising while one silo is flat is a
  completely different story from both rising.
- **Check `www` never appears** in the `page` dimension if you 301 it to the apex. If
  it does, your redirect broke and you are splitting signals across two hosts.

## Site-specific context (optional local overlay)

**If `LOCAL.md` exists in this skill's directory, read it first.** It holds the
property ID, the live setup state, and the site-specific structure/caveats for
whatever site this install is pointed at — the things that make the recipes above
concrete instead of theoretical.

It is deliberately **gitignored**: it is per-install, and it tends to accumulate
details (property IDs, internal structure, candid notes about which pages are weak)
that belong on your disk and not in a public repo. Absent it, everything above still
works — you just supply the property yourself.

## Reporting rules

- **Lead with the number, then the caveat.** "`/tr/` 412 impressions, 3 clicks over
  30d" beats "Turkish is doing okay".
- **Never state a total as exact** — sampling makes sums wrong. Say "at least".
- **Under ~4 weeks of data on a new locale/silo, say so and refuse the verdict.**
  Sparse data is not evidence of failure; it is absence of evidence.
- Segment before concluding. Sitewide impressions rising while one locale is flat is
  a completely different story from both rising.
