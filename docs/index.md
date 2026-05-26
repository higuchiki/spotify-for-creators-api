# Spotify for Creators — Unofficial API Reference

!!! danger "Unofficial API — Use at your own risk"
    This documentation covers **unofficial, reverse-engineered** internal APIs
    used by Spotify for Creators (formerly Anchor.fm). These APIs are
    undocumented and may change or break without notice. Use of these APIs
    may violate Spotify's Terms of Service. The authors are not affiliated
    with Spotify.

This site documents the internal HTTP and GraphQL APIs powering
[Spotify for Creators](https://creators.spotify.com) (formerly Anchor.fm),
discovered through browser traffic analysis using JavaScript fetch/XHR
interceptors.

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
| [REST API](rest-api.md) | All anchor.fm REST endpoints (1-A through 1-Y) |
| [GraphQL API](graphql.md) | All GraphQL queries and mutations |
| [Recipes](recipes.md) | End-to-end Python implementation examples |
| [Changelog](changelog.md) | Research dates and update history |

---

## Research Methodology

All endpoints were discovered by:

- Injecting JavaScript `fetch` and `XHR` interceptors via browser DevTools
- Capturing live traffic while navigating the Spotify for Creators web app
- Performing real operations (episode create, update, comment approve/delete/reply, pin, block) on a test show to capture mutation payloads

Research dates: **2026-05-25 to 2026-05-26**
