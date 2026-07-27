# 更新履歴

## 2026年7月27日 — コメント投稿者メタデータの発見・showComments スキーマ修正

**調査方法：** コメントダイジェストツールへの投稿者名追加作業中のライブイントロスペクション＋本番クエリ。

### graphql.md / graphql.ja.md

- **コメント投稿者名が取得可能** — `DecoratedComment.oneOfAuthorDisplayMetadata` は
  `UserAuthorDisplayMetadata { username, userFullName, userCoverImageUrl }`（リスナーのコメント）と
  `ShowAuthorDisplayMetadata`（制作者側の投稿）のUNION型。`userFullName` がSpotifyアプリの表示名。
- **`showComments` のシグネチャを修正** — 旧掲載の簡略形（`primaryFilters:` 直指定・`comments{…}`）は
  スキーマと一致しない。検証済みの正しい形は `listShowCommentsRequest` 入力オブジェクト
  （`commentFilters` / `sortOrder` / `pagingInfo`）＋ `decoratedCommentOrReply[]` 戻り値。
- `DecoratedComment` の全フィールド一覧（イントロスペクション）を追加 —
  `numberOfReplies`・`hasParentEntityReplied`・`hasParentEntityPinned`・
  `containsBlockedWords`・`initialModerationState` 等。返信スレッドを取得せずに
  「未返信コメント一覧」を作るのに有用。
- 認証メモ：Cookie失効後の再認証は **`sp_dc` の更新のみで復旧**した。ブラウザのCookie一覧に
  `sp_key` 自体が存在しないケースを確認（2026-07-27・spotifyconnectorフロー）。
- 同日追記：`ShowAuthorDisplayMetadata` のフィールドをイントロスペクションで確認 —
  `showUri`・`showFullName`・`showCoverImageUrl`（制作者投稿はユーザーではなく番組のアイデンティティを持つ）。

## 2026年6月24日 — 英語・日本語ドキュメントの双方向同期

**内容：** 3つのドキュメントペアを手動で差分確認・同期。

### graphql.ja.md

- アナリティクス — エンゲージメントタブ・ディスカバリータブ・ベンチマーク・AI インサイトの
  オペレーション一覧テーブルを追加（EN の構造に対応）
- アナリティクス — 概要タブ・オーディエンスタブのオペレーション一覧テーブルを追加
  （2026年6月追加の操作を含む）
- エピソード一覧ページ・エピソード作成・設定ページのセクションを追加
- `getEngagementStats`・`getEngagementStatsNRT`・`getShowRetention`・
  `getEpisodeTimeSeriesByMetric` の完全な GraphQL クエリボディを追加
  （`foregroundConsumptionPercent` の注記を含む）
- コメント管理 mutation（承認・削除・返信・既読マーク・ピン留め・ブロック）に
  variables 例と `!!! note` ブロックを追加
- `showsForUser` クエリに `pagination` フィールドを追加（EN と対応）

### rest-api.ja.md

- ID システムのテーブルを追加（4種類の ID と使用箇所を説明）
- 1-A・1-B・1-C・1-L に Python サンプルに加えて cURL サンプルを追加
- 1-B に下書きエピソードの 403 挙動に関する `!!! warning` ブロックを追加
- 1-L に `stationId` / `userId` / `webStationId` のフィールド説明テーブルを追加

---

## 2026年6月24日 — showByShowUri.showComments の文書化（番組横断コメント取得）

**調査方法：** 収録前コメントダイジェストツール開発中にライブ API テストで確認。

### graphql.md / graphql.ja.md

- **`getLatestCommentsForShow` は PQ エイリアスであり実在するクエリフィールドではない** — ブラウザのネットワークトラフィックにはこのオペレーション名が見えるが、`/v2/graph-pq` スキーマには存在しない。アドホッククエリとして送ると `ValidationError` が返る。
- **正しいパス：`showByShowUri` → `Show.showComments(…)`** — アナリティクス取得と同じ入り口で、`analytics` の代わりに `showComments` をネストフィールドとして指定する。番組全エピソードのコメントを1リクエストで取得できる。
- **`NEEDS_REVIEW` フィルターは `DataFetchingException` を引き起こす** — `showComments` 使用時は `primaryFilters` に `LIST_COMMENT_PRIMARY_FILTER_PUBLISHED` のみを指定すること。`getCommentsForEpisode` では両フィルター値が安全に使えるのと対照的。
- フルクエリスキーマと動作確認済みの variables 例（`pageSize: 20`、ルートコメントのみ）を追加。

---

## 2026年6月17日 — publishOn の整数要件の修正・音声アップロードの S3 パスを文書化

**調査方法：** ぼくらの戦略論番組への実際の EP アップロード（52.5 MB mp3、S3 パス）によるライブテスト。

### rest-api.ja.md / recipes.ja.md

- **`publishOn` を Unix 秒整数に修正** — ISO 8601 文字列を送ると値がサイレントに破損する（API が先頭の数字 `"2026"` を 2026 秒として解釈し `1970-01-01T00:33:46.000Z` になる）。`wizardDraftedToPublishOn` は逆で ISO 8601 文字列が必須（整数を送ると HTTP 400）。
- **音声アップロードは GCS ではなく AWS S3 にルーティングされる** — 音声の署名付き URL レスポンスは `{ uploadId, url, headers }` ではなく `{ requestUuid, signedUrl, fileKey }` を返す。process_upload と poll エンドポイントも音声では `requestUuid` を使用する。
- **音声のポーリングエンドポイントは一時的に 404 を返す** — `process_upload` 完了後、最大約 60 秒間レコードが利用できなくなる場合がある。バックオフでリトライすること。
- 動画 process_upload リクエストボディの `userId`（`stationId` ではない）を修正。

---

## 2026年6月12日 — エンゲージメントタブと新メトリクスの文書化（2026年6月調査）

**調査方法：** JavaScript バンドル解析 — S4C アナリティクス マイクロフロントエンドの
バンドル（`microfrontend-analytics-cdn.spotifycdn.com`）から GraphQL AST を
抽出・再構成。すべてのオペレーション名とクエリ構造は本番バンドルから直接取得した。

### 新規文書化されたオペレーション

**エンゲージメントタブ（確定）：**
- **`getEngagementStats`** — 視聴時間（`SHOW_CONSUMPTION` → `ConsumptionValue.totalConsumptionHours`）
  と平均視聴時間（`SHOW_AVERAGE_CONSUMPTION_TIME` → `AverageConsumptionTimeValue.averageConsumptionSeconds`）
  の日次時系列
- **`getEngagementStatsNRT`** — 期間合計に `periodOverPeriodPercentageDiff` 付き
  （ダッシュボードの `+36.6%` / `+7.4%` バッジの値）
- **`getShowRetention`** — `SHOW_RETENTION` / `AGGREGATION_TYPE_WEEKLY` による
  週次リテンション率時系列（`RatioValueFloat` —例：`0.537` = 53.7%）
- **`getEpisodeCompletionRates`** — 最新10エピソードの完全再生率（`EPISODE_AVERAGE_COMPLETION_RATE` /
  `WINDOW_FIRST_SEVEN_DAYS` → `PercentageValueFloat`）
- **`getEpisodeTimeSeriesByMetric`** — 単一エピソードの任意メトリクス時系列（
  `ConsumptionValue.foregroundConsumptionPercent` も取得可能）

**オーディエンス成長（コアファン3層モデル確定）：**
- **`getAudienceGrowthTimeSeries`** — コアファン / 発展途上 / 新規オーディエンスの
  日次時系列（`coreFanTimeSeries` フィールド使用。文字列日付、AnalyticsWindow enum ではない）
- **`getAudienceGrowthMetricSummary`** — `gainedCoreFans`（獲得コアファン数）と
  前期比付きサマリー
- **`getAudienceGrowthInsights`** — コアファンと全体オーディエンスの
  リテンション率・視聴時間比較

**ベンチマーク：**
- **`getBenchmarkTotal`** / **`getBenchmarkTimeSeries`** — 類似ショーとの
  パーセンタイル比較（20/40/50/60/80）
- `BenchmarkMetricType` enum（13値）と `BenchmarkEpisodePool` enum
  （`ALL_EPISODES`、`LAST_10_EPISODES`）を文書化

**概要タブ：**
- **`getShowStreamsAndDownloadsDaily`** — 全プラットフォーム時系列（`$aggregationType` 変数で日次/週次/月次切り替え可能）
- **`getShowTopEpisodesByMetric`** — 任意 `EpisodeAnalyticsMetricType` でエピソードランキング
- **`getShowTopEpisodesByImpressions`** — ソース別内訳付きインプレッションランキング（新 enum `TOP_EPISODES_BY_IMPRESSIONS_FACETED`）
- **`getShowStreams`** / **`getShowStreamsNRT`** — Spotify 専用ストリーム日次/合計
- **`getShowAllPlatformsStats`** / **`getShowAllPlatformsStatsNRT`** — 全期間全プラットフォーム合計
- **`getPerformanceStats`** / **`getPerformanceStatsNRT`** — 再生数・オーディエンス・フォロワー増減
- **`getShowMetadata`** — ショーメタデータと全期間再生数合計
- **`getShowEpisodesByMetric`** — メトリクス付きページネーションエピソード一覧

**ディスカバリーファネル（修正）：**
- 名前付きオペレーション **`getShowDiscoveryFunnelStats`** を確定
- `DiscoveryFunnelStep.stepValue` のフィールド構造を修正
  （union 型：`CountValueLong`（インプレッション/再生）/ `RatioValueFloat`（完了率））

**オーディエンスセグメント（名前付きオペレーション確定）：**
- 名前付きオペレーション **`getShowSegmentedAudienceTotal`** / **`getShowSegmentedAudienceTimeSeries`** を確定
  （以前は無名クエリとして文書化していた）

**AI インサイト：**
- **`generateAnalyticsInsight`**（mutation）、**`getAnalyticsInsight`**（query）、
  **`submitAnalyticsInsightFeedback`**（mutation）を文書化

### 新規文書化された enum 値
- `ShowAnalyticsMetricType`：`SHOW_CONSUMPTION`、`SHOW_AVERAGE_CONSUMPTION_TIME`、
  `SHOW_RETENTION`、`SHOW_BENCHMARK_EPISODE`、`SHOW_STREAMS_AND_DOWNLOADS_AVERAGE`、
  `SHOW_STREAMS_AND_DOWNLOADS_BY_APP`、`SHOW_STREAMS_AND_DOWNLOADS_BY_DEVICE`、
  `SHOW_STREAMS_AND_DOWNLOADS_BY_GEO_ALL_PLATFORMS`、`TOP_EPISODES_BY_IMPRESSIONS_FACETED`、
  `SHOW_IMPRESSIONS_FACETED`、`SHOW_STREAMS_FACETED`、`SHOW_FOLLOWER_GROWTH`
- `EpisodeAnalyticsMetricType`：`EPISODE_AVERAGE_COMPLETION_RATE`、
  `EPISODE_AVERAGE_CONSUMPTION_TIME`、`EPISODE_CONSUMPTION`、
  `EPISODE_AUDIENCE_SIZE`、`EPISODE_IMPRESSIONS`（＋8値）
- `AnalyticsWindow`：`WINDOW_FIRST_SEVEN_DAYS`、`WINDOW_FIRST_THIRTY_DAYS`、
  `WINDOW_FIRST_SIXTY_DAYS`、`WINDOW_LAST_SIXTY_DAYS`、`WINDOW_SINCE_PUBLISHED`、
  `WINDOW_UNSPECIFIED`

### エンドポイント数の更新
| カテゴリ | 件数 |
|---------|------|
| anchor.fm REST エンドポイント | 25 |
| GraphQL クエリ | 約50（+15件追加） |
| GraphQL ミューテーション | 約15（+3件：AI インサイト系） |

---

## 2026年6月12日 — エピソード削除エンドポイントの文書化

**調査方法：** 本番ショーでのウィザードによるアップロード＋削除サイクルの
ブラウザトラフィック記録、およびAPI直接検証。

- **`POST /v3/episodes/{ANCHOR_ID}/delete`** — エピソード削除エンドポイントを
  文書化（REST API → 1-Z）。空の `{}` ボディ、`200` を返す。使い捨ての
  テストエピソードで検証（下書き・予約公開・公開済みのいずれでも動作）。
  取り消し不可（ゴミ箱・復元なし）。

## 2026年6月12日 — 2026年5月追加メトリクスの文書化

**調査方法：** スキーマイントロスペクション + 本番ショーへの実クエリ。
2026年5月下旬のダッシュボード刷新（Investor Day 発表に連動とみられる）で追加されたメトリクス。

- **`SHOW_SEGMENTED_AUDIENCE`** — `SegmentedAudienceValue`（`totalAudience` / `newAudience` /
  `returningAudience`）を返す新メトリクス。`AGGREGATION_TYPE_TOTAL` および
  `AGGREGATION_TYPE_DAILY` の両方で動作確認済み。日次モードでは各 point の value が
  `SegmentedAudienceValue` に解決される（インラインフラグメント必須）。
- **重要な指標セマンティクス — `newAudience` と `returningAudience` の非対称性：**
  `newAudience` は期間ユニーク（日次合計 = 期間合計）。
  `returningAudience` はリスナーが聴いた日ごとにカウント（日次合計 >> 期間合計 — 90日窓で約9倍の例も）。
  90日分全日付で `new + returning == total` が完全一致することで整合性を検証済み。
- **`SHOW_DISCOVERY_FUNNEL`** — `DiscoveryFunnelValue` を返す新メトリクス。
  Impressions → Plays → 平均完了率の3ステップファネルで、各ステップに
  `conversionRateToNext`（転換率）と `periodOverPeriodPercentageDiff`（前期比）を含む。
- **`SHOW_WINDOWED_AVERAGE_COMPLETION_RATE`** — `PercentageValueFloat` を返すことを確認。
  期間内の全エピソードを横断した平均完了率。
- **`Show` 型の新フィールド（スキーマ確認済み・未テスト）：**
  `coreFanTimeSeries`、`coreFanMetricSummary`、`coreFanInsight`（コアファン関連）、
  `sponsorshipAnalytics`、`listSponsorships`（スポンサーシップ）、
  `analyticsInsight`、`getEpisodeInsights`（AI インサイト）。

詳細は [GraphQL API → 新メトリクス（2026年5月ダッシュボード刷新で追加）](graphql.ja.md) を参照。

## 2026年6月11日 — プラットフォーム変更：再生数（Plays）の定義変更（30秒 / AMP標準）

**API調査エントリではなく、本ドキュメント全体の指標セマンティクスに影響する
Spotify 側のプラットフォーム変更。**

- Spotify が **Plays（再生数）** の定義を「**30秒以上の視聴 = 1再生**」に変更
  （AMP（Audience Measurement Principles）業界標準準拠）。従来は再生開始時点で
  1再生とカウントされていた。
- S4C ダッシュボードの全プラットフォーム看板指標が **Streams + Downloads** から
  **Plays + Downloads** に交代。旧 Streams + Downloads は引き続き取得可能だが
  「参照用」に降格。
- 同時に新機能が追加：**エンゲージメントタブ**（視聴時間・リテンション・完全再生率）、
  **ベンチマーク**（類似番組とのパーセンタイル比較）、**オーディエンスセグメント**
  （コアファン / 新規 / リピーター）。対応する GraphQL オペレーションは上記
  2026年6月12日のエントリで文書化済み。
- **注意：** この日付より前に記録した再生数のセマンティクス（例：`SHOW_PLAYS` が
  podcasters API の *starts* 合計と一致するという記述）は旧定義に基づくもので、
  数値的に成立しなくなっている可能性がある。

## 2026年6月11日 — アナリティクス Query 実測確定

**調査方法**: スキーマイントロスペクション + 本番ショーへの実 GraphQL クエリ

- **`/v2/graph-pq` が完全な GraphQL クエリ文字列を受け付けることを確認** — Persisted Query は必須ではなく、イントロスペクションも有効
- アナリティクスデータの実際のアクセス経路を発見：ブラウザ観測のオペレーション名（`getShowAudienceAllPlatformsStats` 等）はトップレベル Query フィールドには**存在せず**、`showByShowUri → Show.analytics(getShowAnalyticsRequest)` / `episodeByUri → Episode.analytics(getEpisodePlayCountRequest)` のネスト構造でアクセスする
- `ShowAnalyticsMetricType` / `EpisodeAnalyticsMetricType` / `AggregationType` / `AnalyticsWindow` の enum を実測セマンティクス付きで文書化
- **重要発見 — metric 名は誤解を招く**：`SHOW_PLAYS` は Spotify 単体。ダッシュボード「すべてのプラットフォーム」に対応する全プラットフォーム値は `*_AND_DOWNLOADS` 系
- `SHOW_STREAMS_AND_DOWNLOADS` = 全エピソードの `EPISODE_STREAMS_AND_DOWNLOADS` 合算（誤差 <0.01%）を実証 — エピソードループは1コールで置き換え可能
- 日次/週次/月次の時系列・ウィンドウ指定・`ShowUri` / `EpisodeUri` スカラー・引数なし `analyticsStarts { startsCount }`（Spotify 単体 starts）を文書化

詳細は [GraphQL API → アナリティクス Query（実測確定）](graphql.ja.md) を参照。

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

## エンドポイント数サマリー（2026-06-12 時点）

| カテゴリ | 件数 |
|---------|------|
| anchor.fm REST エンドポイント（1-A 〜 1-Z） | 26（1-V・1-W・6-A〜6-D のサブエンドポイント含む） |
| GraphQL query | 約50 |
| GraphQL mutation | 約15 |
