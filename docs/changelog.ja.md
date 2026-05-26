# 更新履歴

## 2026年5月25日〜26日 — 初回調査

**調査方法**: Claude in Chrome MCP による JavaScript fetch/XHR インターセプター計測

### 2026-05-25 — コア API 発見

- 2バックエンド構成の確認：anchor.fm REST API と GraphQL API
- 認証フローの確定：`sp_dc` / `sp_key` → `spotifyconnector` → Bearer トークン
- 主要エピソードエンドポイントのキャプチャ：
    - **1-A**：Spotify Episode ID → Anchor numeric ID 変換
    - **1-B**：エピソード詳細取得（`GET /v3/episodes/{id}/overview`）
    - **1-C**：エピソード更新（`POST /v3/episodes/{id}/update`）
- `publishOn` のタイムゾーン（UTC）・状態マトリクス・予約公開挙動の確定
- 既知の注意事項の確認：`description` の HTML 自動変換・draft エピソードの 403 挙動

### 2026-05-26 — API カバレッジ拡大

テストショー（STATION_ID: YOUR_STATION_ID）を使って各ページを操作し、追加調査を実施。

**新規発見の REST エンドポイント：**

- **1-D**：リージョンエンドポイント
- **1-E / 1-Q**：スポンサードコンテンツ状態（GET / PUT）
- **1-F**：カバーアート配色取得
- **1-G**：エピソードフォームデータ一括取得
- **1-H**：ショーメタデータ取得 — 全フィールドをキャプチャ
- **1-I**：パートナーID取得
- **1-J**：ユーザー認証確認
- **1-K**：オンボーディング状態確認
- **1-L**：Spotify Show ID → anchor numeric IDs 変換（`legacyIds`）— **重要な新発見**
- **1-M**：エピソード一覧インデックス更新
- **1-N**：ショー形式データ取得
- **1-O**：配信設定取得 — 全フィールドをキャプチャ
- **1-P**：ユーザー利用規約・連絡可否設定
- **1-Q**：スポンサードコンテンツ状態更新（PUT）
- **1-R**：マネタイズライフサイクル同期
- **1-S**：エピソード新規作成（空レコード作成）
- **1-T**：ショーメタデータ更新（POST）
- **1-U**：vanity slug 更新
- **1-V**：画像アップロードパイプライン（カバーアート・エピソードサムネイル）
- **1-W**：収益化エンドポイント（ペイウォール・ウォレット）
- **1-X**：番組設定ページ固有エンドポイント
- **1-Y**：配信ページ固有エンドポイント

**新規発見の GraphQL オペレーション：**

- アナリティクス：`getShowOnSpotifyStats`、`getShowOnSpotifyStatsNRT`、`getShowOverviewStatsNRT`、`getEpisodePlaysTotal`、`WebGetIndexedEpisodeList`
- オーディエンスタブ：`getShowAudienceDemographicsStats`、`getShowAudienceAllPlatformsGeoStats`
- 配信：`getPublishedEpisodeCount`
- エピソード作成：`getChaptersEligibility`、`createEpisodeSmartlink`、`getEverboardingModal`、`getLatestCommentsForShow`

**コメント管理 — 実操作によるキャプチャ：**

リスナーアカウント（樋口さん）がテストショーにコメントを投稿し、制作者アカウントで承認・返信・削除・ピン留め・ブロック等の実操作を行いインターセプター計測。

確定した mutation：

- `publishCommentByCommentUri`（承認）
- `deleteCommentByCommentUri`（削除、`filter` フィールドの値確定）
- `createCommentReplyByCommentUri`（制作者返信）
- `markCommentAsRead`（承認・返信と同時に自動発火）
- `pinComment` / `unPinComment`（ピン留め・解除）
- `blockUserForShow`（ユーザーブロック）
- `updateThreadCreatorLastSeen`（返信スレッド既読更新）

確定した query：

- `getPinnedCommentForEpisode`
- `GetCommentReplies`
- `getEpisodeData`

非承認（要審査に戻す）操作：**S4C UI に存在しない**ことを確認。

**メディアアップロードパイプライン — 確定：**

- 音声ファイルは `uploadType: "default"` を使う（`"audio"` は誤り）
- 動画ファイルは `uploadType: "video"` + `isExtractedFromVideo: true`
- GCS の ETag はダブルクォートを除去してから `parts` に渡す

**未取得・取得不可と判明したもの：**

- トランスクリプト編集 mutation：テストアカウントでは機能が無効化されており取得不可
- チャプター作成・更新・削除 mutation：テストショーの資格なしで UI にアクセス不可

---

## エンドポイント数サマリー

| カテゴリ | 件数 |
|---------|------|
| anchor.fm REST エンドポイント（1-A 〜 1-Y） | 25（1-V・1-W・6-A〜6-D のサブエンドポイント含む） |
| GraphQL query | 約35 |
| GraphQL mutation | 約12 |
