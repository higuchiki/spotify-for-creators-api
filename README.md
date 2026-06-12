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
| `api-v5.anchor.fm` REST | Episode CRUD (incl. delete), show metadata, audio/video upload, scheduling, distribution, monetisation |
| `creators-graph.spotify.com` GraphQL | Analytics (~50 queries: overview, engagement, retention, audience growth / core fans, benchmarks, discovery funnel, segmented audience, AI insights), comments, transcripts, chapters |
| `spotifyconnector` | Bearer token retrieval from `sp_dc` / `sp_key` cookies |
| Browser automation | Playwright recipes for UI-only operations |

> **📢 Platform change (2026-06-11):** Spotify redefined **Plays** as *30+ seconds listened = 1 play* (aligning with the AMP industry standard). The S4C dashboard headline metric switched from **Streams + Downloads** to **Plays + Downloads** across all platforms; the old metric remains available for reference only. New Engagement tab, benchmarks, and audience segments shipped alongside. Metric semantics documented before this date may reflect the old definitions — see the [Changelog](https://higuchiki.github.io/spotify-for-creators-api/changelog/).

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

Endpoints were discovered through three complementary methods:

1. **Browser traffic capture** — JavaScript `fetch` / `XHR` interceptors injected via DevTools while navigating and performing live operations on the S4C web app (initial research, 2026-05-25 to 2026-05-26)
2. **GraphQL schema introspection + live queries** against a production show (2026-06-11 to 2026-06-12)
3. **JavaScript bundle AST analysis** — GraphQL operation ASTs extracted from the S4C analytics microfrontend bundle (2026-06-12)

Initial research: **2026-05-25 to 2026-05-26**
Last research: **2026-06-12**

## Local docs development

```bash
pip install mkdocs-material mkdocs-static-i18n
mkdocs serve
```

## Contact

Found an error or noticed an API change? → [@HiguchiKi](https://x.com/HiguchiKi)
