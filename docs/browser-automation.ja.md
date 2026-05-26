# ブラウザ自動化

S4C にはファイルアップロード用の公式 API が存在しない。
[Playwright](https://playwright.dev/) によるブラウザ自動化で、
エピソードウィザードの UI を直接スクリプト化することで対応する。

---

## API とブラウザ自動化の使い分け

可能な限り REST / GraphQL API を使う。速く、安定しており、ヘッドレスブラウザが不要。
Playwright が必要なのは、API で対応できない操作に限定する。

| 操作 | 推奨アプローチ |
|------|--------------|
| エピソードメタデータの読み取り | REST API（`/overview`） |
| タイトル・概要・公開日の更新 | REST API（`/update`） |
| ファイルアップロード | **Playwright**（アップロード API なし） |
| 予約公開の初回設定（ウィザード） | **Playwright** |
| **公開済み**エピソードの公開日変更 | **Playwright**（REST `/update` はサーバー側でロックされる） |
| コメントモデレーション | GraphQL API |

---

## 認証

Playwright は [認証](auth.md) ページで説明している `sp_dc` / `sp_key` クッキーをそのまま使う。
実行時にログイン操作は不要。

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    context = browser.new_context()

    context.add_cookies([
        {
            "name": "sp_dc",
            "value": sp_dc,
            "domain": ".spotify.com",
            "path": "/",
            "secure": True,
            "httpOnly": True,
            "sameSite": "None",
        },
        {
            "name": "sp_key",
            "value": sp_key,
            "domain": ".spotify.com",
            "path": "/",
            "secure": True,
            "httpOnly": True,
            "sameSite": "None",
        },
    ])

    page = context.new_page()
```

---

## エピソードウィザード — ステップバイステップ

エピソードウィザードは以下の URL にある複数ステップ構成の React SPA。

```
https://creators.spotify.com/pod/show/{SHOW_ID}/episode/wizard
```

!!! warning "URL に必ず Show ID を含める"
    Show ID を含まない汎用 URL（`/episode/wizard`）は**最後に使用した番組**を
    自動選択する。複数番組を管理している場合、意図せず別の番組にアップロードされる。
    必ず番組固有の URL を使うこと。

### Step 1: ファイルアップロード

ファイル入力欄は非表示だが、`set_input_files()` でファイルを渡せる。

```python
# 優先セレクタ
file_input = page.query_selector("#uploadAreaInput")

# フォールバック
if not file_input:
    file_input = page.wait_for_selector(
        "input[type=file]", state="attached", timeout=15000
    )

file_input.set_input_files("/path/to/episode.mp3")
```

!!! warning
    `input[type=file][accept*=audio]` は使わない。`accept` 属性の値は
    `'.mp3, .m4a, ...'` 形式で `audio` という文字列を含まないため、
    このセレクタは一致しない。

### Step 2: アップロード完了の検出

ファイルをセットすると、UI が自動的に Details ステップへ遷移する。
次の操作に進む前に、完了シグナルを待つ。

```python
# "Preview ready!" テキストの出現を待つ（大きなファイルは最大5分）
page.wait_for_selector("text=Preview ready!", timeout=300000)

# フォールバック: タイトル入力欄が表示されるのを待つ
page.wait_for_selector("#title-input", state="visible", timeout=30000)
```

### Step 3: タイトル入力

```python
title_input = page.wait_for_selector("#title-input", state="visible", timeout=10000)
title_input.click()
title_input.fill("エピソードタイトル")
```

### Step 3: 概要欄入力（Slate.js）

S4C の概要欄は Slate.js ベースのリッチテキストエディタを使用している。
`fill()` や `keyboard.type()` は React の内部状態を更新しないため、
送信時に入力内容が無視される。

!!! note "なぜ React Fiber を使うのか"
    React はブラウザ DOM とは別に仮想 DOM ツリーを管理している。
    `fill()` はブラウザ DOM に書き込むが仮想 DOM をバイパスするため、
    React は変更を検知できない。Fiber ツリーを辿ることで
    エディタ自身の `insertText` メソッドを直接呼び出し、
    両方のツリーを同時に更新できる。

```python
desc_js = """(text) => {
    const ed = document.querySelector(
        "div[contenteditable='true'][data-slate-editor='true']"
    );
    if (!ed) return 'editor not found';
    const fk = Object.keys(ed).find(
        k => k.startsWith('__reactFiber') || k.startsWith('__reactInternalInstance')
    );
    if (!fk) return 'fiber not found';
    let cur = ed[fk];
    let sl = null;
    for (let i = 0; i < 100 && cur; i++) {
        if (cur.memoizedProps?.editor?.insertText) {
            sl = cur.memoizedProps.editor;
            break;
        }
        cur = cur.return;
    }
    if (!sl) return 'slate not found';
    const start = sl.start([]);
    const end   = sl.end([]);
    sl.select({ anchor: start, focus: end });
    sl.insertText(text);
    return 'ok';
}"""

result = page.evaluate(desc_js, "エピソードの概要テキスト")
assert result == "ok", f"概要欄の挿入に失敗: {result}"
```

!!! note
    HTML フォーマット（リンク・改行）を保持したい場合は、
    ウィザード完了後に REST API `/update` エンドポイントで
    `htmlDescription` を直接書き込む方が確実。
    Slate.js アプローチはプレーンテキストのみ挿入できる。

### Step 3: OneTrust バナーの除去

クッキー同意オーバーレイがクリック操作を妨害することがある。
このステップのクリック操作の前に DOM から除去する。

```python
page.evaluate("""() => {
    const ids = [
        'onetrust-consent-sdk',
        'onetrust-banner-sdk',
        'onetrust-pc-dark-filter',
    ];
    ids.forEach(id => {
        const el = document.getElementById(id);
        if (el) el.remove();
    });
    document.body.style.overflow = '';
}""")
```

### Step 4: Next ボタン → Review へ遷移

```python
next_btn = page.wait_for_selector("button:has-text('Next')", timeout=10000)
next_btn.click()

# wait_for_load_state("networkidle") は SPA ではタイムアウトする。
# 次ステップの DOM 要素の出現を待つ。
page.wait_for_selector("#publish-date-now", state="attached", timeout=20000)
```

### Step 5: 予約公開設定

「後で公開する」ラジオボタンは視覚的に非表示のため、`<label>` 経由でクリックする。

```python
schedule_label = page.query_selector("label[for='publish-date-schedule']")
if schedule_label:
    schedule_label.click()
else:
    # フォールバック: JavaScript 経由でクリック
    page.evaluate("() => { document.getElementById('publish-date-schedule').click(); }")
```

### Step 5: 日付ピッカー

```python
# カレンダーを開く
date_btn = page.wait_for_selector(
    "button[class*='calendarDatePicker'], "
    "div[class*='calendarDatePicker__wrapper'] button",
    timeout=8000,
)
date_btn.click()

# 現在表示中の月を読み取る
caption_sel = (
    "div[class*='CalendarMonth'][data-visible='true'] "
    "div[class*='CalendarMonth_caption'] > strong"
)
current_month_text = page.text_content(caption_sel)  # 例: "June 2026"

# 月の移動
page.click("div[class*='calendarDatePicker__navIcon--left']")   # 前月
page.click("div[class*='calendarDatePicker__navIcon--right']")  # 次月

# 日付セルをクリック（ゼロパディングなし — "1", "15" など）
day_sel = (
    f"xpath=//div[contains(@class, 'CalendarMonth') and @data-visible='true']"
    f"//td[text()='{day}']"
)
page.click(day_sel)
```

### Step 5: 時刻ピッカー

`fill()` と `keyboard.type()` は React 制御のインプットでは機能しない。
React Fiber 経由で `onChange` を直接呼び出す。AM/PM は `select_option()` が
React の合成イベントを正しく発火させる。

```python
# 時（12時間制 — ゼロパディングあり文字列、例: "06" = 午前6時）
page.wait_for_selector("input[data-testid='hour-picker']", timeout=5000)
page.evaluate("""(hourVal) => {
    const input = document.querySelector('input[data-testid="hour-picker"]');
    const fk = Object.keys(input).find(
        k => k.startsWith('__reactFiber') || k.startsWith('__reactInternalInstance')
    );
    let cur = input[fk];
    for (let i = 0; i < 20 && cur; i++) {
        if (cur.memoizedProps?.onChange) {
            cur.memoizedProps.onChange({ target: { value: hourVal } });
            return 'ok';
        }
        cur = cur.return;
    }
    return 'onChange not found';
}""", "06")

# 分
page.evaluate("""(minVal) => {
    const input = document.querySelector('input[data-testid="minute-picker"]');
    const fk = Object.keys(input).find(
        k => k.startsWith('__reactFiber') || k.startsWith('__reactInternalInstance')
    );
    let cur = input[fk];
    for (let i = 0; i < 20 && cur; i++) {
        if (cur.memoizedProps?.onChange) {
            cur.memoizedProps.onChange({ target: { value: minVal } });
            return 'ok';
        }
        cur = cur.return;
    }
}""", "00")

# AM/PM — select_option() は React の合成 onChange を正しく発火させる
page.select_option("select[aria-label='Meridiem picker']", "AM")
```

### Step 6: Schedule ボタン

「後で公開する」を選択すると、Publish ボタンが Schedule ボタンに変わる。

```python
schedule_btn = page.wait_for_selector(
    "button:has-text('Schedule')", timeout=10000
)
schedule_btn.click()
```

---

## トラブルシューティング

| 症状 | 原因 | 解決策 |
|------|------|--------|
| `input[type=file][accept*=audio]` が見つからない | `accept` 属性の値に `audio` 文字列が含まれない | `#uploadAreaInput` を使う |
| 別の番組にアップロードされる | 汎用 URL が最後に使用した番組を自動選択する | URL に Show ID を埋め込む |
| 概要欄が送信時に空になる | `fill()` / `keyboard.type()` が React の仮想 DOM をバイパスする | React Fiber の `insertText` アプローチを使う |
| 時刻ピッカーの値が保存されない | React 制御のインプットは DOM 直接書き換えを無視する | React Fiber の `onChange` アプローチを使う |
| `wait_for_load_state("networkidle")` がタイムアウト | SPA の内部遷移はネットワークが止まらない | 次ステップの特定 DOM 要素の出現を待つ |
| 概要欄の HTML フォーマットが失われる | Slate.js はプレーンテキストとして扱い HTML タグを除去する | ウィザード後に REST API `/update` で `htmlDescription` を設定する |
| `publishOn` が `1970-01-01` になる | ミリ秒ではなく秒単位の Unix タイムスタンプが入った | S4C UI のスケジュール設定で上書きする |
