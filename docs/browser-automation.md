# Browser Automation

S4C does not provide an official upload API. Browser automation with
[Playwright](https://playwright.dev/) fills that gap by scripting the
Episode Wizard UI directly.

---

## When to Use Browser Automation vs API

Use the REST / GraphQL API whenever possible — it is faster, more reliable,
and requires no headless browser. Fall back to Playwright only for operations
that have no API equivalent.

| Operation | Preferred approach |
|-----------|-------------------|
| Read episode metadata | REST API (`/overview`) |
| Update title, description, publish date | REST API (`/update`) |
| File upload | **Playwright** (no upload API exists) |
| Scheduled publish (initial wizard) | **Playwright** |
| Change publish date of a *published* episode | **Playwright** (REST `/update` is blocked by the server for already-published episodes) |
| Comment moderation | GraphQL API |

---

## Authentication

Playwright uses the same `sp_dc` / `sp_key` cookies described in the
[Authentication](auth.md) page — no login interaction is needed at runtime.

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

## Episode Wizard — Step by Step

The Episode Wizard is a multi-step React SPA at:

```
https://creators.spotify.com/pod/show/{SHOW_ID}/episode/wizard
```

!!! warning "Always embed your Show ID in the URL"
    The generic URL (`/episode/wizard` without a show ID) automatically
    selects **the last show you used**. If you manage multiple shows this
    silently uploads to the wrong one. Always use the show-specific URL.

### Step 1: File Upload

The file input is hidden from view but can still receive files via
`set_input_files()`.

```python
# Primary selector
file_input = page.query_selector("#uploadAreaInput")

# Fallback
if not file_input:
    file_input = page.wait_for_selector(
        "input[type=file]", state="attached", timeout=15000
    )

file_input.set_input_files("/path/to/episode.mp3")
```

!!! warning
    Do **not** use `input[type=file][accept*=audio]`. The `accept` attribute
    value is `'.mp3, .m4a, ...'` — it does not contain the string `audio`,
    so that selector never matches.

### Step 2: Detecting Upload Completion

After the file is set, the UI automatically advances to the Details step.
Wait for the completion signal before proceeding.

```python
# Wait for "Preview ready!" text — up to 5 minutes for large files
page.wait_for_selector("text=Preview ready!", timeout=300000)

# Fallback: wait for the title input to become visible
page.wait_for_selector("#title-input", state="visible", timeout=30000)
```

### Step 3: Title Input

```python
title_input = page.wait_for_selector("#title-input", state="visible", timeout=10000)
title_input.click()
title_input.fill("Your Episode Title")
```

### Step 3: Description Input (Slate.js)

S4C uses a Slate.js rich-text editor for the description field. Standard
Playwright text-entry methods (`fill()`, `keyboard.type()`) do not update
React's internal state — only the DOM changes, so the value is silently
discarded on submit.

!!! note "Why React Fiber?"
    React keeps its own virtual DOM tree separate from the browser DOM.
    `fill()` writes to the browser DOM but bypasses the virtual DOM, so
    React never registers the change. Traversing the Fiber tree lets you
    call the editor's own `insertText` method, which updates both trees
    simultaneously.

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

result = page.evaluate(desc_js, "Your episode description here.")
assert result == "ok", f"Description insert failed: {result}"
```

!!! note
    If you need to preserve HTML formatting (links, line breaks) in the
    description, use the REST API `/update` endpoint to write
    `htmlDescription` directly after the wizard completes. The Slate.js
    approach inserts plain text only.

### Step 3: Removing the OneTrust Banner

A cookie-consent overlay sometimes blocks click targets. Remove it from the
DOM before any click operations in this step.

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

### Step 4: Next Button → Review

```python
next_btn = page.wait_for_selector("button:has-text('Next')", timeout=10000)
next_btn.click()

# Do NOT use wait_for_load_state("networkidle") — it times out on SPAs.
# Wait for a DOM element in the next step instead.
page.wait_for_selector("#publish-date-now", state="attached", timeout=20000)
```

### Step 5: Scheduled Publish

The "Schedule for later" radio button is visually hidden — click its
`<label>` instead.

```python
schedule_label = page.query_selector("label[for='publish-date-schedule']")
if schedule_label:
    schedule_label.click()
else:
    # Fallback: trigger click via JavaScript
    page.evaluate("() => { document.getElementById('publish-date-schedule').click(); }")
```

### Step 5: Date Picker

```python
# Open the calendar
date_btn = page.wait_for_selector(
    "button[class*='calendarDatePicker'], "
    "div[class*='calendarDatePicker__wrapper'] button",
    timeout=8000,
)
date_btn.click()

# Read the currently displayed month
caption_sel = (
    "div[class*='CalendarMonth'][data-visible='true'] "
    "div[class*='CalendarMonth_caption'] > strong"
)
current_month_text = page.text_content(caption_sel)  # e.g. "June 2026"

# Navigate months if needed
page.click("div[class*='calendarDatePicker__navIcon--left']")   # previous month
page.click("div[class*='calendarDatePicker__navIcon--right']")  # next month

# Click the target day (no zero-padding — "1", "15", not "01")
day_sel = (
    f"xpath=//div[contains(@class, 'CalendarMonth') and @data-visible='true']"
    f"//td[text()='{day}']"
)
page.click(day_sel)
```

### Step 5: Time Picker

`fill()` and `keyboard.type()` do not update React-controlled inputs.
Use React Fiber to invoke `onChange` directly, then use the native
`select_option()` for AM/PM (which correctly fires React's synthetic event).

```python
# Hour (12-hour format — zero-padded string, e.g. "06" for 6 AM)
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

# Minute
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

# AM/PM — select_option() correctly fires React's synthetic onChange
page.select_option("select[aria-label='Meridiem picker']", "AM")
```

### Step 6: Schedule Button

After selecting "Schedule for later", the Publish button becomes a Schedule
button.

```python
schedule_btn = page.wait_for_selector(
    "button:has-text('Schedule')", timeout=10000
)
schedule_btn.click()
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `input[type=file][accept*=audio]` not found | The `accept` attribute value is `'.mp3, .m4a, ...'` — no `audio` substring | Use `#uploadAreaInput` instead |
| File uploads to the wrong show | The generic wizard URL selects the last-used show | Embed the Show ID in the URL |
| Slate.js description is empty after submit | `fill()` / `keyboard.type()` bypass React's virtual DOM | Use the React Fiber `insertText` approach above |
| Time picker value not saved | React-controlled inputs ignore direct DOM writes | Use React Fiber `onChange` for hour/minute |
| `wait_for_load_state("networkidle")` times out | SPA internal navigation never reaches network-idle | Wait for a specific DOM element in the next step |
| Description HTML formatting lost | Slate.js stores plain text; HTML tags are stripped | Use the REST API `/update` endpoint to set `htmlDescription` after the wizard |
| `publishOn` is `1970-01-01` | Unix timestamp was passed in seconds instead of milliseconds | Overwrite the schedule from the S4C UI |
