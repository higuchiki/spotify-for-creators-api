# Spotify for Creators API（非公式）

!!! danger "非公式API — 自己責任でご利用ください"
    このドキュメントは、Spotify for Creators（旧 Anchor.fm）が内部で使用している
    **非公式・リバースエンジニアリングによって解析した** APIを対象としています。
    これらの API は非公開であり、予告なく仕様変更・廃止される可能性があります。
    本サイトの著者は Spotify とは無関係です。

このサイトでは、[Spotify for Creators](https://creators.spotify.com)（旧 Anchor.fm）の
内部 HTTP / GraphQL API を解説しています。ブラウザトラフィック解析・GraphQL スキーマ
イントロスペクション・JavaScript バンドル AST 解析により発見されたエンドポイントを
体系的にまとめています。

**最終調査日：2026-06-12**

!!! info "プラットフォーム変更（2026-06-11）：再生数（Plays）の定義変更"
    2026-06-11、Spotify は **Plays（再生数）** の定義を「**30秒以上の視聴 = 1再生**」に
    変更しました（AMP 業界標準準拠）。これに伴い S4C ダッシュボードの全プラットフォーム
    看板指標が **Streams + Downloads** から **Plays + Downloads** に交代し、
    旧指標は「参照用」に降格しました。同時に **エンゲージメントタブ**（視聴時間・
    リテンション・完全再生率）、**ベンチマーク**（パーセンタイル比較）、
    **オーディエンスセグメント**（コアファン / 新規 / リピーター）などの新機能が
    追加されています。この日付より前に記録した指標のセマンティクスは旧定義に
    基づいている可能性があります。詳細は[更新履歴](changelog.md)を参照してください。

---

## 全体アーキテクチャ

S4C のバックエンドは **2系統** に分かれており、用途によって使い分ける。

```
Spotify for Creators (creators.spotify.com)
         │
         ├─ anchor.fm 内部 REST API  (api-v5.anchor.fm)
         │    └─ エピソード読み書き・ショー設定・ユーザー状態
         │       ← Anchor 買収後も現役。Python + requests で叩ける
         │
         ├─ GraphQL API  (creators-graph.spotify.com/v2/graph-pq)
         │    └─ アナリティクス・コメント・トランスクリプト・章立て等
         │       ← 新機能はこちらに移行中
         │
         ├─ Next.js データ API  (creators.spotify.com/interactivity/_next/data/)
         │    └─ コメントページの SSR データ取得
         │
         └─ spotifyconnector / generic.wg.spotify.com/podcasters/v0
              └─ Bearer トークン取得・エピソード一覧取得（非公式 Python ライブラリ）
```

### 使い分け

| やりたいこと | 使う API |
|------------|---------|
| エピソードのタイトル・概要欄・公開日時を読み書きしたい | anchor.fm REST API |
| アナリティクス・コメント・トランスクリプトを取得したい | GraphQL API |
| Bearer トークンを取得したい | `spotifyconnector` ライブラリ |

---

## クイックスタート

```python
from spotifyconnector import SpotifyConnector
import requests

# 1. Bearer トークン取得
connector = SpotifyConnector(
    base_url="https://generic.wg.spotify.com/podcasters/v0",
    client_id="05a1371ee5194c27860b3ff3ff3979d2",
    podcast_id="YOUR_SHOW_ID",
    sp_dc="YOUR_SP_DC",
    sp_key="YOUR_SP_KEY",
)
connector._authenticate()
bearer = connector._bearer

# 2. エピソード詳細取得
anchor_id = 123456789  # ID変換は 1-A を参照
r = requests.get(
    f"https://api-v5.anchor.fm/v3/episodes/{anchor_id}/overview"
    "?isMumsCompatible=true&returnWebIds=true",
    headers={"Authorization": f"Bearer {bearer}", "Accept": "application/json"},
)
print(r.json())
```

認証の詳細は [認証](auth.md) ページを参照してください。

---

## ドキュメント構成

| ページ | 内容 |
|--------|------|
| [認証](auth.md) | `sp_dc` / `sp_key` クッキーと Bearer トークンの取得方法 |
| [REST API リファレンス](rest-api.md) | anchor.fm REST エンドポイント全一覧（1-A 〜 1-Z、エピソード削除含む） |
| [GraphQL API](graphql.md) | GraphQL クエリ・ミューテーション全一覧 — アナリティクス（概要 / エンゲージメント / リテンション / オーディエンス成長 / ベンチマーク / ディスカバリーファネル / AIインサイト）・コメント・トランスクリプト・チャプター |
| [レシピ・実装例](recipes.md) | Python によるエンドツーエンドの実装フロー |
| [ブラウザ自動化](browser-automation.md) | UI操作のみで可能な処理の Playwright レシピ |
| [更新履歴](changelog.md) | 調査日・更新内容の履歴 |

---

## 調査方法

エンドポイントは以下の3つの手法を組み合わせて発見した。

- **ブラウザトラフィックキャプチャ** — JavaScript の `fetch` / `XHR` インターセプターを
  DevTools から注入し、テスト用ショーでエピソード作成・更新・削除・コメント承認/削除/
  返信/ピン留め/ブロックなどの実操作を行い、ペイロードを確認
- **GraphQL スキーマイントロスペクション + 実クエリ** — 本番ショーに対する実測検証
- **JavaScript バンドル AST 解析** — S4C アナリティクス マイクロフロントエンドの
  バンドルから GraphQL オペレーション AST を抽出・再構成

初回調査：**2026年5月25日〜26日** ／
最終調査：**2026年6月12日**（詳細は[更新履歴](changelog.md)）

---

## お問い合わせ

誤りやAPI仕様の変更を発見した場合は、X（Twitter）までご連絡ください：[@HiguchiKi](https://x.com/HiguchiKi)
