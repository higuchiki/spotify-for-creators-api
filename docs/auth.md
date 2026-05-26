# Authentication

All Spotify for Creators APIs share the same authentication chain:

```
sp_dc / sp_key cookies
       │
       └─▶ spotifyconnector._authenticate()
                  │
                  └─▶ Bearer token  (used in Authorization header for all requests)
```

---

## Step 1 — Obtain `sp_dc` and `sp_key` Cookies

These two cookies are the root credentials for the entire S4C session.

!!! warning "Security warning"
    `sp_dc` and `sp_key` grant full access to your Spotify account — music,
    payments, and all features. Never share them with anyone and store them
    in a `.env` file that is excluded from version control via `.gitignore`.

### How to obtain them

1. Open Chrome and navigate to `https://creators.spotify.com`.
2. Log in with your Spotify account.
3. Open DevTools (`F12` or `Cmd+Option+I`).
4. Go to **Application → Cookies → `https://creators.spotify.com`**.
5. Copy the **Value** column for `sp_dc` and `sp_key`.

---

## Step 2 — Find Your Show ID

You can read your Show ID directly from the Spotify for Creators URL:

```
https://creators.spotify.com/pod/show/【SHOW_ID_IS_HERE】/episodes
```

---

## Step 3 — Obtain a Bearer Token

Use the unofficial [`spotifyconnector`](https://pypi.org/project/spotifyconnector/)
Python library to exchange your cookies for a short-lived Bearer token.

### Install

```bash
pip install spotifyconnector==0.8.2
```

### Usage

```python
from spotifyconnector import SpotifyConnector

connector = SpotifyConnector(
    base_url="https://generic.wg.spotify.com/podcasters/v0",
    client_id="05a1371ee5194c27860b3ff3ff3979d2",  # S4C public OAuth client_id
    podcast_id="YOUR_SHOW_ID",
    sp_dc="YOUR_SP_DC_COOKIE_VALUE",
    sp_key="YOUR_SP_KEY_COOKIE_VALUE",
)
connector._authenticate()   # internal method
bearer = connector._bearer  # internal property — short-lived JWT
```

!!! note
    `_authenticate()` and `_bearer` are internal (private) attributes of the
    library. Their names may change in future versions of `spotifyconnector`.
    The `client_id` value (`05a1371ee5194c27860b3ff3ff3979d2`) is the public
    OAuth client ID embedded in the S4C web app and is the same for all users.

---

## Using the Bearer Token

Once you have the Bearer token, include it in every API request:

### GET requests

```python
headers_get = {
    "Authorization": f"Bearer {bearer}",
    "Accept": "application/json",
}
```

### POST / PUT requests

```python
headers_post = {
    "Authorization": f"Bearer {bearer}",
    "Accept": "application/json",
    "Content-Type": "application/json",
    "Origin": "https://creators.spotify.com",
    "Referer": "https://creators.spotify.com/",
}
```

!!! warning
    The `Origin` and `Referer` headers are required for POST requests.
    Omitting them can cause the server to reject the request.

---

## .env Template

```ini
SP_DC=your_sp_dc_value_here
SP_KEY=your_sp_key_value_here
SHOW_ID=your_show_id_here
```

```python
import os
from dotenv import load_dotenv

load_dotenv()
sp_dc = os.environ["SP_DC"]
sp_key = os.environ["SP_KEY"]
show_id = os.environ["SHOW_ID"]
```
