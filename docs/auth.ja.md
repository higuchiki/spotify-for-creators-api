# 認証

Spotify for Creators のすべての API は、共通の認証チェーンを使用する。

```
sp_dc / sp_key クッキー
       │
       └─▶ spotifyconnector._authenticate()
                  │
                  └─▶ Bearer トークン（全リクエストの Authorization ヘッダーで使用）
```

---

## Step 1 — `sp_dc` と `sp_key` クッキーの取得

この2つのクッキーが S4C セッション全体の認証の起点。

!!! warning "セキュリティ警告"
    `sp_dc` と `sp_key` は **Spotify アカウント全体**（音楽・支払い情報を含む全機能）への
    アクセス権限を持つ。絶対に他人と共有しないこと。`.env` ファイルに保存し、
    `.gitignore` でバージョン管理対象から除外すること。

### 取得手順

1. Chrome で `https://creators.spotify.com` にアクセスする
2. Spotify アカウントでログインする
3. DevTools を開く（`F12` または `Cmd+Option+I`）
4. **Application → Cookies → `https://creators.spotify.com`** に移動する
5. `sp_dc` と `sp_key` の **Value** 列をコピーする

---

## Step 2 — Show ID の確認

Spotify for Creators の URL から直接確認できる。

```
https://creators.spotify.com/pod/show/【ここが Show ID】/episodes
```

---

## Step 3 — Bearer トークンの取得

非公式ライブラリ [`spotifyconnector`](https://pypi.org/project/spotifyconnector/) を使って
クッキーを短命の Bearer トークンに変換する。

### インストール

```bash
pip install spotifyconnector==0.8.2
```

### 使い方

```python
from spotifyconnector import SpotifyConnector

connector = SpotifyConnector(
    base_url="https://generic.wg.spotify.com/podcasters/v0",
    client_id="05a1371ee5194c27860b3ff3ff3979d2",  # S4C のパブリックな OAuth client_id
    podcast_id="YOUR_SHOW_ID",
    sp_dc="YOUR_SP_DC_COOKIE_VALUE",
    sp_key="YOUR_SP_KEY_COOKIE_VALUE",
)
connector._authenticate()   # 内部メソッド
bearer = connector._bearer  # 内部プロパティ — 短命な JWT
```

!!! note
    `_authenticate()` と `_bearer` は `spotifyconnector` ライブラリの内部（プライベート）属性。
    ライブラリの将来バージョンで名前が変わる可能性がある。
    `client_id`（`05a1371ee5194c27860b3ff3ff3979d2`）は S4C の Web アプリに埋め込まれた
    パブリックな OAuth クライアント ID で、すべてのユーザーで共通。

---

## Bearer トークンの使い方

取得した Bearer トークンをすべての API リクエストに含める。

### GET リクエスト

```python
headers_get = {
    "Authorization": f"Bearer {bearer}",
    "Accept": "application/json",
}
```

### POST / PUT リクエスト

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
    POST リクエストには `Origin` と `Referer` ヘッダーが必要。
    省略するとサーバーにリクエストが拒否される場合がある。

---

## .env テンプレート

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
