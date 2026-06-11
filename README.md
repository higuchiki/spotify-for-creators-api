# Spotify for Creators API (Unofficial)

> **⚠️ Unofficial API — Use at your own risk.**
> These APIs are reverse-engineered, undocumented, and may change or break without notice. Not affiliated with Spotify.

Unofficial API reference for **Spotify for Creators** (formerly Anchor.fm), discovered through browser traffic analysis.

📖 **[Read the full documentation →](https://higuchiki.github.io/spotify-for-creators-api/)**
(Available in English and 日本語)

---

## What's covered

| API | Purpose |
|-----|---------|
| `api-v5.anchor.fm` REST | Episode CRUD, show metadata, audio upload, scheduling |
| `creators-graph.spotify.com` GraphQL | Analytics, comments, transcripts, chapters |
| `spotifyconnector` | Bearer token retrieval from `sp_dc` / `sp_key` cookies |

## Quick start

```python
from spotifyconnector import SpotifyConnector
import requests

# 1. Get a Bearer token from sp_dc / sp_key cookies
#    (obtained from browser DevTools → Application → Cookies)
connector = SpotifyConnector(
    base_url="https://generic.wg.spotify.com/podcasters/v0",
    client_id="05a1371ee5194c27860b3ff3ff3979d2",
    podcast_id="YOUR_SHOW_ID",
    sp_dc="YOUR_SP_DC_COOKIE",
    sp_key="YOUR_SP_KEY_COOKIE",
)
connector._authenticate()
bearer = connector._bearer

# 2. Fetch episode details
headers = {"Authorization": f"Bearer {bearer}", "Accept": "application/json"}
r = requests.get(
    "https://api-v5.anchor.fm/v3/episodes/ANCHOR_ID/overview?isMumsCompatible=true",
    headers=headers,
)
print(r.json())
```

See [Authentication](https://higuchiki.github.io/spotify-for-creators-api/auth/) for how to obtain cookies and tokens.

## Research methodology

All endpoints were discovered by injecting JavaScript `fetch` / `XHR` interceptors via browser DevTools and capturing live traffic while navigating the Spotify for Creators web app.

Research dates: **2026-05-25 to 2026-05-26**

## Local docs development

```bash
pip install mkdocs-material mkdocs-static-i18n
mkdocs serve
```

## Contact

Found an error or noticed an API change? → [@HiguchiKi](https://x.com/HiguchiKi)
