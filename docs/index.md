# Spotify for Creators API (Unofficial)

!!! danger "Unofficial API — Use at your own risk"
    This documentation covers **unofficial, reverse-engineered** internal APIs
    used by Spotify for Creators (formerly Anchor.fm). These APIs are
    undocumented and may change or break without notice. The authors are not affiliated
    with Spotify.

This site documents the internal HTTP and GraphQL APIs powering
[Spotify for Creators](https://creators.spotify.com) (formerly Anchor.fm),
discovered through browser traffic analysis, GraphQL schema introspection,
and JavaScript bundle AST analysis.

**Last research: 2026-06-12**

!!! info "Platform change (2026-06-11): Plays redefined"
    On 2026-06-11 Spotify redefined **Plays** as *30+ seconds listened = 1 play*,
    aligning with the AMP industry standard. The S4C dashboard headline metric
    switched from **Streams + Downloads** to **Plays + Downloads** (all platforms);
    the old metric was demoted to "reference" status. New features shipped
    alongside: an **Engagement tab** (consumption time, retention, completion
    rates), **benchmarks** (percentile comparison), and **audience segments**
    (core fans / new / returning). Metric semantics documented before this date
    may reflect the old definitions — see the [Changelog](changelog.md).

---

## Architecture Overview

Spotify for Creators runs on **two distinct backend systems** that are used
together:

```
Spotify for Creators (creators.spotify.com)
         │
         ├─ anchor.fm Internal REST API  (api-v5.anchor.fm)
         │    └─ Episode read/write, show settings, user state
         │       ← Still active post-Anchor acquisition. Callable with Python + requests.
         │
         ├─ GraphQL API  (creators-graph.spotify.com/v2/graph-pq)
         │    └─ Analytics, comments, transcripts, chapters, etc.
         │       ← New features are migrating here.
         │
         ├─ Next.js Data API  (creators.spotify.com/interactivity/_next/data/)
         │    └─ SSR data fetch for the comments page
         │
         └─ spotifyconnector / generic.wg.spotify.com/podcasters/v0
              └─ Bearer token retrieval, episode list (unofficial Python library)
```

### When to use which

| Goal | Use |
|------|-----|
| Read/write episode title, description, or publish date | anchor.fm REST API |
| Fetch analytics, comments, or transcripts | GraphQL API |
| Obtain a Bearer token | `spotifyconnector` library |

---

## Quick Start

```python
from spotifyconnector import SpotifyConnector
import requests

# 1. Obtain a Bearer token
connector = SpotifyConnector(
    base_url="https://generic.wg.spotify.com/podcasters/v0",
    client_id="05a1371ee5194c27860b3ff3ff3979d2",
    podcast_id="YOUR_SHOW_ID",
    sp_dc="YOUR_SP_DC",
    sp_key="YOUR_SP_KEY",
)
connector._authenticate()
bearer = connector._bearer

# 2. Fetch episode overview
anchor_id = 123456789  # see ID Conversion (1-A)
r = requests.get(
    f"https://api-v5.anchor.fm/v3/episodes/{anchor_id}/overview"
    "?isMumsCompatible=true&returnWebIds=true",
    headers={"Authorization": f"Bearer {bearer}", "Accept": "application/json"},
)
print(r.json())
```

See [Authentication](auth.md) for how to obtain `sp_dc` / `sp_key` and a Bearer token.

---

## Pages in This Documentation

| Page | Contents |
|------|---------|
| [Authentication](auth.md) | Obtaining `sp_dc` / `sp_key` cookies and Bearer tokens |
| [REST API](rest-api.md) | All anchor.fm REST endpoints (1-A through 1-Z, including episode delete) |
| [GraphQL API](graphql.md) | All GraphQL queries and mutations — analytics (overview / engagement / retention / audience growth / benchmarks / discovery funnel / AI insights), comments, transcripts, chapters |
| [Recipes](recipes.md) | End-to-end Python implementation examples |
| [Browser Automation](browser-automation.md) | Playwright recipes for UI-only operations |
| [Changelog](changelog.md) | Research dates and update history |

---

## Research Methodology

Endpoints were discovered through three complementary methods:

- **Browser traffic capture** — injecting JavaScript `fetch` and `XHR` interceptors
  via DevTools, capturing live traffic, and performing real operations
  (episode create, update, delete, comment approve/delete/reply, pin, block)
  on a test show to capture mutation payloads
- **GraphQL schema introspection + live queries** against a production show
- **JavaScript bundle AST analysis** — extracting GraphQL operation ASTs from the
  S4C analytics microfrontend bundle

Initial research: **2026-05-25 to 2026-05-26** /
Last research: **2026-06-12** (see [Changelog](changelog.md))

---

## Contact

Found an error or noticed an API change? Feel free to reach out on X (Twitter): [@HiguchiKi](https://x.com/HiguchiKi)
