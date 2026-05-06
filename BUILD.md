# How this app was built

A field guide to the Yap Family · California 2026 trip app — what's in it, why each piece exists, and how to extend it.

## What this is

A bilingual (English + 中文), aunty-friendly trip companion for 17 family members across 13 days in California. It started as a static viewer (v1) and grew into a personalized + manager-collaborative app (v2.5) with profiles, real-time budget, in-app location editing, and a built-in chat assistant.

Two index files live side-by-side:
- `index.html` — v1, the live deployed viewer at https://dyap123.github.io/yap-family-trip/
- `index-v2.html` — v2.5, the "neon Chongqing" rebuild with profiles, Firebase, and Ah Zai

`index-v2.html` is what we're iterating on now. v1 is kept around as a fallback.

## Stack — and why each piece

The constraints that drove every decision:
1. **Aunties without smartphone fluency need to use it.** No installs, no logins, no copy-paste of URLs.
2. **The trip lead (Danzel) needs to update plans live without redeploying.** No Vercel preview URLs, no `git push` for content fixes.
3. **The whole thing should fit in one HTML file** so any future editor can read the entire app top-to-bottom without bouncing between import maps, build configs, and dependency trees.
4. **Cost ≈ $0.** Hosted on GitHub Pages. Free Firebase RTDB tier. MiniMax with a $20 credit.

| Piece | Why this one |
|---|---|
| **Static GitHub Pages SPA** (single `.html` file, no build step) | One file. Push to `main`. Done. Aunty opens link, app works. |
| **Google Sheet (read-only via GViz)** for food + schedule | Trip lead types in a sheet, app refreshes within 60 seconds. No API key needed for public read. |
| **Firebase Realtime Database** for money + edit-overrides | Real-time sync across manager devices. Free tier. Trivial schema-less writes. Same pattern reused from CUP Dashboard. |
| **Leaflet + Carto Dark Matter tiles** for the map | OpenStreetMap-derived dark tiles. No API key. Works on iOS/Android browsers without permissions. |
| **MiniMax M2.7** (`MiniMax-Text-01`) for Ah Zai | Already paid for. Anthropic-compatible chat completions API. Bilingual EN/中文 out of the box. |
| **localStorage** for profile + chat history + Firebase URL | Persists between visits without an auth flow. Per-device, which is fine — aunties don't switch phones. |
| **Material Symbols Outlined** | Free, comprehensive icon font. Inline `<span class="material-symbols-outlined">name</span>`. |
| **Inter + Noto Sans SC + JetBrains Mono** | Inter for English, Noto Sans SC for Chinese characters that render correctly on every device, JetBrains Mono for the tech-y meta labels. |
| **No framework. No build.** | The app is HTML, vanilla CSS, and plain `<script>`. Future-you can edit it in any text editor. |

## File layout

```
~/yap-family-trip/
  index.html          # v1, original deployed viewer
  index-v2.html       # v2.5, current build (3700+ lines: HTML, CSS, JS in one file)
  config.json         # Firebase URL + MiniMax key fallback (mostly placeholders now —
                      # real values get pasted into Firebase or localStorage)
  README-v2.md        # User-facing setup instructions
  BUILD.md            # this file — architecture / build notes
```

That's it. No `package.json`, no `node_modules`, no `vite.config.js`. Open `index-v2.html` in a browser.

## Data flow

```
                   Google Sheet (public read)              Google Sheet (public read)
                   "Schedule Overview"                     "Food Recs (First Timer)"
                            │                                       │
                            └────────── GViz JSON ──────────┐       │
                                                             ▼       ▼
   ┌── localStorage ──────────────┐                    parseGviz()
   │  yap_profile                 │                          │
   │  yap_skip_welcome            │                          ▼
   │  yap_manager                 │      ┌──── state.schedule, state.food
   │  yap_firebase_url            │──►   │
   │  yap_ahzai_history_{name}    │      │     index-v2.html (the SPA)
   └──────────────────────────────┘      │              │
                                         │              ├──► render Today / Trip / Map / Food / Family
                                         │              │
                                         ▼              │
                            ┌─── Firebase RTDB ◄────────┤   write expenses, location overrides
                            │   yap-trip/expenses/      │
                            │   yap-trip/locations/     │
                            │   yap-trip/budget/        │
                            │   config/minimax_key  ────┴──► used by Ah Zai
                            │
                            └─── on('value') listeners feed live updates back into state
                                                        │
                                                        ▼
                                       MiniMax API (direct browser POST)
                                       https://api.minimaxi.com/v1/text/chatcompletion_v2
                                                        │
                                                        ▼
                                                 Ah Zai chat reply
```

Three data sources. Three different freshness models:
- **Sheet → app**: 60-second client-side cache. Fresh-on-load + background refresh.
- **Firebase → app**: real-time. `on('value')` callbacks fire as soon as anyone writes.
- **MiniMax**: per-request. Each Ah Zai message is a fresh chat completion.

## The shape of `state`

One big mutable JS object. Every render function reads from it:

```js
state = {
  schedule:  [Day1, Day2, …, Day13],     // from Sheet, with FALLBACK seed
  food:      [Restaurant, …],            // from Sheet
  updates:   [Update, …],                // from Sheet "Updates" tab
  loaded:    bool,
  isManager: bool,                       // PIN-gated
  profile:   {name, group: 1|2|null},    // from localStorage
  expenses:  {pushId: Expense},          // from Firebase
  budget:    {target, currency},         // from Firebase
  locationOverrides: {hotel:{}, activity:{}, food:{}},  // from Firebase
  ahzaiHistory: [{role, content}, …],    // per-user, localStorage
  ahzaiPending: bool,
  map:       LeafletMap,
  mapHotelMarkers: [LeafletMarker, …],
  mapPath:   [[lat,lng], …],
  mapProgressPoly: LeafletPolyline,
  tourTimer: number?,                    // setInterval handle
};
```

## Key features and how they work

### 1. User profiles + group filter

On first load (or after `Switch user`), the picker overlay shows all 17 names grouped by Group 1 / Group 2. Tap → `state.profile = {name, group}` → `localStorage.yap_profile`. The header subtitle replaces "MAY 14 – 26 · 加州" with "Hi {name} {Chinese} · 你好".

For split days (Days 3, 4, 6, 7), the `dayCard` renderer detects `s.split === true` and runs `splitDayParts(s)` which scans the accommodation/events strings for `(G1)`/`(G2)` or `Group 1`/`Group 2` markers and returns two halves. The card renders **two stacked sub-cards**:
- The user's group sub-card has `class="your-group"` → pink glow + "YOUR · 你的" badge
- The other group's sub-card has `class="dim"` → 0.55 opacity (skipped if user is in manager mode — they see both at full opacity)

The parser falls back gracefully: if it can't split the string, both sub-cards show the same combined text.

### 2. Manager mode

A toggle, not a separate user. Family-mode users tap the gear icon and see only "Theme / Welcome / Switch user / Trip lead access". Tapping `Trip lead access · 领队登录` opens a 4-digit PIN modal (default `8888`, override via `localStorage.yap_manager_pin`). On success: `body` gets a `.manager` class, the Money tab appears in bottom nav, edit pencils appear on hotel cards, and the "Edit in Sheet" button appears in food modals.

There's a deliberate decision here: profile and manager are **independent**. Picking "Danzel" doesn't auto-grant manager — manager is always PIN-gated. This means anyone can view-as-Danzel but only PIN-holders can write money or edit locations.

### 3. Firebase money + budget split

The most data-heavy feature. Schema:

```
yap-trip/expenses/{pushId}
  date "May 17", desc "Hell's Kitchen dinner", amount 680,
  paidBy "Dad", splitMode "all"|"group1"|"group2"|"custom",
  splitWith ["Mom", "Dad", "Da ma"]   // only when splitMode=custom
  createdBy "Danzel", createdAt 1715865432000

yap-trip/budget/
  target 30000, currency "USD"
```

Add/edit modal lets the manager pick:
- **Paid by**: chip-picker of all 17 names (single-select)
- **Split mode**: 4 big buttons. `Everyone · 所有人` / `Group 1 · 少年组` / `Group 2 · 家人组` / `Pick people · 选人`
- If `Pick people`, a multi-select chip-picker reveals.

Settle-up math (`computeSettle` in `index-v2.html`):
1. For each expense, compute the `eligible` array based on splitMode (all → ALL_NAMES, group1 → GROUP_1, etc.)
2. `share[name] += amount / eligible.length` for each eligible person.
3. `paid[name] += amount` for the payer.
4. `net[name] = paid[name] - share[name]`. Positive = net up. Negative = owes.

The Firebase listener fires `renderMoney()` on every change, so Danzel's phone and laptop see the same totals within milliseconds.

### 4. Edit locations (override layer)

Hotels and activities are **seeded** in JS arrays (`HOTELS`, `ACTIVITIES`). The manager-only edit modal writes overrides to `yap-trip/locations/{type}/{id}` in Firebase. The override **wins at render time** via `applyOverride(seed, override)` — fields present in the override replace seed fields; missing fields fall back to seed. "Reset to default · 还原" simply removes the Firebase override.

This means:
- The manager can move a pin or fix a typo without a redeploy.
- The seed data stays in source control as the canonical reference.
- Family members on stale caches still get sensible content (the seed) even if Firebase is unreachable.

### 5. Map immersion

The map tab is a Leaflet instance with Carto Dark Matter tiles. Custom DivIcon HTML for each pin so labels are always visible. Three immersion features:

- **TODAY pin**: When `tripDay()` returns 1–13, the corresponding hotel index gets a yellow gradient pin + a pulsing 「今天 · TODAY」 badge above it.
- **Day strip**: 13 horizontally-scrollable buttons under the map. Tap → `flyToDay(day)` smoothly zooms to that day's location and opens its popup.
- **Progress polyline**: A dim dashed pink line shows the whole route. A bright solid pink overlay polyline grows from start to the active day. Tour mode auto-advances 1 day every 2.4 seconds, and the progress line lights up segment-by-segment.
- **Info card**: A small card above the map shows the active day's plan ("DAY 5 · MAY 18 · Las Vegas · Bacchanal Buffet, Cirque O"). Updates on every `flyToDay`. Yellow theme when the active day is today.

### 6. Ah Zai 阿仔 chat

A floating 64px round avatar bottom-right. Tap → slide-up sheet (or right-rail panel ≥900px). The sheet has:
- Suggestion chips (`What's tomorrow?`, `Hotel tonight`, `My group`, `今天计划`, `Food rec`)
- Scrolling message list (per-user history in `localStorage.yap_ahzai_history_{name}`, capped at 30 messages)
- Textarea + send button (Enter to send, Shift+Enter for newline)
- "Clear chat · 清除" link

Each request to MiniMax includes:
1. A **system prompt** assembled live from `state` — user's profile, today's day/plan, tomorrow, tonight's hotel, group splits, top food picks. Tells the model: "be concise, witty, kid-brother vibe, 1–3 sentences, reply in the user's language."
2. The full chat history (filtered for non-error messages) so context carries across turns.

Direct browser POST to `https://api.minimaxi.com/v1/text/chatcompletion_v2`. The API key is read in priority order:
1. Firebase at `config/minimax_key` (set by listener at boot)
2. `config.json` `minimax.apiKey` (fallback)

If neither is set, Ah Zai shows a friendly error pointing to setup.

## Visual system — "neon Chongqing"

The aesthetic. Why this and not something else:
- The user wanted **OpenYap style** — already established across CUP Dashboard, Vault Brain, Roadmap site.
- Plus: aunties (and most family members) come from Singapore where neon-night-market night vibes feel familiar and warm, not cold or cyberpunk.
- The result: deep navy-black bg, hot pink + cyan + yellow + magenta neon aurora blobs, glassmorphic cards with glowing borders, vertical Chinese signage motifs (`霓虹 / 加州 / 雅`) sitting at low opacity in the corners.

CSS variables (top of `<style>` block) parameterize the palette. Swap them and the whole app re-skins.

Aunty-friendly touches in the same theme:
- 18px base font (most apps default to 14–16)
- 56px+ minimum touch targets on all buttons
- Bilingual labels everywhere: "Today · 今天", "Lock & Send · 锁定", etc.
- Cards have padding generous enough to avoid mis-taps
- The welcome screen + profile picker + Ah Zai avatar all double-encode in EN + 中文

## How to deploy

```sh
cd ~/yap-family-trip
git add index-v2.html config.json README-v2.md BUILD.md
git commit -m "v2.5: profiles, Firebase money, Ah Zai, edit locations"
git push
```

GitHub Pages auto-publishes within ~30 seconds. The live URL is `https://dyap123.github.io/yap-family-trip/index-v2.html`.

For local dev:
```sh
python3 -m http.server 8282
# then http://localhost:8282/index-v2.html
```

The `python3 -m http.server` server is needed because `fetch('config.json')` doesn't work from `file://` URLs in modern browsers.

## How to extend

A few common modifications and where to make them:

| Want to | Edit |
|---|---|
| Add a family member | `GROUP_1` or `GROUP_2` array near the top of the `<script>` block. Optionally add to `NAME_CN` map for Chinese label. |
| Change a day's plan | The `FALLBACK` array, OR the `Schedule Overview` tab in the Google Sheet (sheet wins). |
| Add a food spot | The Google Sheet `Food Recs (First Timer)` tab. App pulls automatically. |
| Add a new hotel/activity pin | `HOTELS` or `ACTIVITIES` arrays in the script. For activities, add a `day` field to drive the gold pin label. |
| Change the theme palette | CSS variables at the top of the `<style>` block. |
| Add a new bottom-nav tab | The `<nav class="nav">` HTML, the `<section class="tab">` content div, and a `data-tab` listener. |
| Change the manager PIN | `localStorage.setItem('yap_manager_pin', '1234')` in DevTools. |
| Add a new Firebase listener | Inside `initFirebase()`, after the existing `db.ref(...).on('value', ...)` blocks. |

## What's deliberately NOT here

To keep the app focused and under one file:
- No login. Profile is just identity, not auth.
- No multi-trip support. One trip per deploy.
- No push notifications. Ah Zai is request-response only.
- No PWA install prompt (yet — could add a manifest, but the bookmarked Safari/Chrome page works fine).
- No image uploads. Food photos come from the Sheet's URL field.
- No Yelp/Google Places auto-fetch (was originally planned, dropped — the user prefers to hard-code the food list in the Sheet).
- No Apps Script proxy for MiniMax (the user is OK with the key being browser-readable for a $20 trip-scale app).

Each of these would add power but also complexity. Trip ends in mid-2026; the app is built for that scope.

## Lessons from building it

- **Glassmorphism + heavy bg effects render fine on iPad Safari** as long as you cap aurora blob count to ~4 and lean on `backdrop-filter` instead of opacity stacks.
- **GViz is robust enough for production** for read-only public sheets; the trick is detecting the header row when banded headers cause GViz to misalign columns. The parser walks the first 5 rows looking for the row with the most short cells.
- **The `(G1) / (G2)` split-string parsing** is intentionally imperfect — when it can't parse, it shows both groups the same combined string. Better than a broken card.
- **Firebase RTDB compat SDK (v10.12) at ~80kb gzipped** is fine for a static page. The modular SDK saves bytes but adds build complexity we explicitly didn't want.
- **MiniMax's `MiniMax-Text-01` model** handles English, Mandarin, Cantonese, and Singlish in a single thread without explicit language switching. The system prompt says "reply in the user's language" and it just works.
- **Per-user localStorage chat history** keyed by `state.profile.name` means switching profiles (e.g. Danzel hands phone to Aunty Coke) instantly switches conversations. Useful for a shared device.

## Credits

- Inspired by the structural ideas in [andrewjiang/palantir-for-family-trips](https://github.com/andrewjiang/palantir-for-family-trips) (modules, convoy timeline, mission-launch overlay).
- Visual system extends the OpenYap aesthetic from CUP Dashboard and the openyap-roadmap site.
- Built iteratively with Claude Code in 2026-05.
