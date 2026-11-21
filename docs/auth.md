# Authentication

All Spotify for Creators APIs share the same authentication chain:

```
sp_dc cookie
       │
       └─▶ spotifyconnector._authenticate()   (Python)
           or a PKCE `prompt=none` exchange   (Node.js, no library)
                  │
                  └─▶ Bearer token  (used in Authorization header for all requests)
```

---

## Step 1 — Obtain the `sp_dc` Cookie

This cookie is the root credential for the entire S4C session.

!!! info "`sp_key` is no longer needed (Oct 2026)"
    `sp_key` no longer appears in the browser's cookie list for
    `creators.spotify.com`, and authentication ignores it entirely — an empty
    string or an arbitrary value both authenticate successfully (verified
    2026-10-17). `spotifyconnector` still requires the `sp_key` argument, so
    pass an empty string.

!!! warning "Security warning"
    `sp_dc` grants full access to your Spotify account — music,
    payments, and all features. Never share it with anyone and store it
    in a `.env` file that is excluded from version control via `.gitignore`.

### How to obtain them

1. Open Chrome and navigate to `https://creators.spotify.com`.
2. Log in with your Spotify account.
3. Open DevTools (`F12` or `Cmd+Option+I`).
4. Go to **Application → Cookies → `https://creators.spotify.com`**.
5. Copy the **Value** column for `sp_dc`.

---

## Step 2 — Find Your Show ID

You can read your Show ID directly from the Spotify for Creators URL:

```
https://creators.spotify.com/pod/show/【SHOW_ID_IS_HERE】/episodes
```

---

## Step 3 — Obtain a Bearer Token

Exchange your cookie for a short-lived Bearer token (about 1 hour).

=== "Python"

    Use the unofficial [`spotifyconnector`](https://pypi.org/project/spotifyconnector/)
    Python library.

    ```bash
    pip install spotifyconnector==0.8.2
    ```

    ```python
    from spotifyconnector import SpotifyConnector

    connector = SpotifyConnector(
        base_url="https://generic.wg.spotify.com/podcasters/v0",
        client_id="05a1371ee5194c27860b3ff3ff3979d2",  # S4C public OAuth client_id
        podcast_id="YOUR_SHOW_ID",
        sp_dc="YOUR_SP_DC_COOKIE_VALUE",
        sp_key="",  # required arg, but ignored — sp_dc alone authenticates
    )
    connector._authenticate()   # internal method
    bearer = connector._bearer  # internal property — short-lived JWT
    ```

=== "Node.js"

    No library is needed (Node 20+, built-in `fetch` and `crypto`). This is the
    same exchange `spotifyconnector._authenticate()` performs: an OAuth
    authorization-code request with PKCE and `prompt=none`, answered silently
    because the `sp_dc` cookie identifies the logged-in user, followed by a
    token request.

    ```javascript
    // bearer.mjs — sp_dc cookie → Bearer token (Node 20+, no dependencies)
    import crypto from "node:crypto";

    const CLIENT_ID = "05a1371ee5194c27860b3ff3ff3979d2"; // S4C public OAuth client_id
    const REDIRECT_URI = "https://podcasters.spotify.com";

    export async function bearer(spDc) {
      const rnd = (n) => crypto.randomBytes(n).toString("base64url").slice(0, n);
      const state = rnd(32);
      const verifier = rnd(64);
      const challenge = crypto.createHash("sha256").update(verifier).digest("base64url");

      // 1. Authorization request — returns an HTML page, not a redirect
      const auth = new URL("https://accounts.spotify.com/oauth2/v2/auth");
      for (const [k, v] of Object.entries({
        response_type: "code",
        client_id: CLIENT_ID,
        scope: "streaming ugc-image-upload user-read-email user-read-private",
        redirect_uri: REDIRECT_URI,
        code_challenge: challenge,
        code_challenge_method: "S256",
        state,
        response_mode: "web_message",
        prompt: "none",
      })) auth.searchParams.set(k, v);

      // An empty sp_key is sent as in the tested flow (it is ignored, see Step 1)
      const html = await (await fetch(auth, { headers: { Cookie: `sp_dc=${spDc}; sp_key=` } })).text();
      if (html.includes("login_required")) throw new Error("sp_dc expired or invalid");

      // 2. The code is embedded in a script: `authorizationResponse = { ... };`
      const block = html.match(/authorizationResponse\s*=\s*([\s\S]*?);/)?.[1] ?? "";
      const code = block.match(/code["']?\s*:\s*["']([^"']+)/)?.[1];
      const gotState = block.match(/state["']?\s*:\s*["']([^"']+)/)?.[1];
      if (!code) throw new Error("authorization code not found in response");
      if (gotState && gotState !== state) throw new Error("state mismatch");

      // 3. Token request (PKCE — no client secret)
      const res = await fetch("https://accounts.spotify.com/api/token", {
        method: "POST",
        headers: { "Content-Type": "application/x-www-form-urlencoded" },
        body: new URLSearchParams({
          grant_type: "authorization_code",
          client_id: CLIENT_ID,
          code,
          redirect_uri: REDIRECT_URI,
          code_verifier: verifier,
        }),
      });
      const tok = await res.json();
      if (!tok.access_token) throw new Error("token request failed: " + JSON.stringify(tok));
      return tok.access_token; // valid for tok.expires_in seconds (about 3600)
    }

    // Usage:
    // const token = await bearer(process.env.SP_DC);
    ```

    !!! note "Verified 2026-11-21"
        Used to upload and publish 28 video episodes. The HTML shape of the
        `web_message` response is undocumented and may change; if the `code`
        regex stops matching, inspect the page for the `authorizationResponse`
        object. Cache the token and reuse it until shortly before
        `expires_in` runs out rather than re-authenticating per request.

!!! note
    `_authenticate()` and `_bearer` are internal (private) attributes of the
    library. Their names may change in future versions of `spotifyconnector`.
    The `client_id` value (`05a1371ee5194c27860b3ff3ff3979d2`) is the public
    OAuth client ID embedded in the S4C web app and is the same for all users.

---

## Using the Bearer Token

Once you have the Bearer token, include it in every API request:

### GET requests

=== "Python"
    ```python
    headers_get = {
        "Authorization": f"Bearer {bearer}",
        "Accept": "application/json",
    }
    ```

=== "Node.js"
    ```javascript
    const headersGet = {
      Authorization: `Bearer ${token}`,
      Accept: "application/json",
    };
    ```

### POST / PUT requests

=== "Python"
    ```python
    headers_post = {
        "Authorization": f"Bearer {bearer}",
        "Accept": "application/json",
        "Content-Type": "application/json",
        "Origin": "https://creators.spotify.com",
        "Referer": "https://creators.spotify.com/",
    }
    ```

=== "Node.js"
    ```javascript
    const headersPost = {
      ...headersGet,
      "Content-Type": "application/json",
      Origin: "https://creators.spotify.com",
      Referer: "https://creators.spotify.com/",
    };
    ```

!!! warning
    The `Origin` and `Referer` headers are required for POST requests.
    Omitting them can cause the server to reject the request.

---

## .env Template

```ini
SP_DC=your_sp_dc_value_here
SHOW_ID=your_show_id_here
```

```python
import os
from dotenv import load_dotenv

load_dotenv()
sp_dc = os.environ["SP_DC"]
show_id = os.environ["SHOW_ID"]
```
