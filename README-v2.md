# Yap Family Trip · v2.5 setup

Static GitHub Pages SPA. No build step. Open `index-v2.html` to use.

## What's new in v2.5

- **User profiles** — every family member picks their name on first load. Header greets them by name. Split days highlight their group.
- **Firebase budget with split** — manager-only Money tab. Add/edit expenses with split modes (everyone / Group 1 / Group 2 / pick people). Live sync across manager devices. Settle-up math.
- **Edit locations** — manager can edit hotel/activity entries (name, address, lat/lng, dates, notes). Persisted as overrides in Firebase. "Reset to default" reverts to seed.
- **Ah Zai 阿仔** — floating MiniMax assistant on the right. Concise, witty, bilingual EN / 中文 / Singlish.

## Files

```
~/yap-family-trip/
  index.html          # v1, original (still served)
  index-v2.html       # v2.5, the new build
  config.json         # Firebase RTDB URL + MiniMax API key
  README-v2.md        # this file
```

## Setup (5 minutes)

### 1. Create a Firebase Realtime Database

1. Go to https://console.firebase.google.com → **Add project** → name it `yap-family-trip` → continue, disable Analytics → create.
2. In the project console: **Build → Realtime Database → Create Database** → pick a region → start in **test mode** (rules: `read/write: true` for now).
3. Copy the URL — it ends in `.firebaseio.com` (e.g. `https://yap-family-trip-default-rtdb.firebaseio.com`).
4. Paste it into `config.json` under `firebase.databaseURL`.

### 2. MiniMax API key (Firebase-backed)

**Recommended path:** drop your existing MiniMax key into the Firebase RTDB so the trip app picks it up at startup, and you don't have to commit it to a public repo.

1. Open the Firebase console → your trip RTDB → **Data** tab.
2. Add a node at root: `config/minimax_key` with your MiniMax API key as the value.
3. The app's Firebase listener pulls it on page load. No `config.json` edit needed.
4. If you also have a key in `config.json` `minimax.apiKey`, the Firebase value wins.

**Why Firebase instead of config.json:** `config.json` ships with your GitHub Pages build and is publicly readable. Firebase is also publicly readable in test mode, but it's at least one more hop to scrape, and you can lock it down with rules later. Either way, MiniMax keys with low credit limits are fine to expose for a small family app.

To get a key the first time: sign up at https://www.minimaxi.com/platform_overview, create an API key, and paste it into `config/minimax_key` per step 2 above.

### 3. Push and serve

```sh
cd ~/yap-family-trip
git add index-v2.html config.json README-v2.md
git commit -m "v2.5: profiles, Firebase money, Ah Zai, edit locations"
git push
```

Then open https://dyap123.github.io/yap-family-trip/index-v2.html.

For local dev: `python3 -m http.server 8282` from `~/yap-family-trip` then http://localhost:8282/index-v2.html.

## How it's used

### Family members (default flow)

1. Open the link. **Welcome** screen → **Enter · 进入**.
2. **"Who are you known to Danzel as?"** picker. Tap your name (or "I'm just looking" to skip).
3. App greets you in the header: "Hi {name} · 你好".
4. On split days (3, 4, 6, 7), your group's plan is highlighted with a pink glow + "YOUR · 你的" badge. The other group's plan is dimmed but visible.
5. **Ah Zai 阿仔** is the pink avatar bottom-right. Tap any time to ask "what's tomorrow?" or "今晚酒店在哪?" Ah Zai knows your group, today's plan, hotels, and food picks.

### Trip lead (Danzel)

1. Tap the **gear icon** in the header → settings panel.
2. Bottom of the panel: **Trip lead access · 领队登录** → enter PIN `8888` → unlocks manager mode.
3. **Money** tab now appears in the bottom nav. Tap it.
4. Tap the gold **+ FAB** to add an expense. Pick paid-by, amount, split mode.
5. Settle-up updates live. Open the same URL on a second manager phone — expenses sync automatically.
6. **Edit locations**: in manager mode, hotel cards on the Trip tab show a yellow pencil. Tap to edit. Save persists to Firebase; family members see updates on their next load.

## Data model (Firebase RTDB)

```
config/
  minimax_key                       # string — your MiniMax API key (read at boot)

yap-trip/
  expenses/{pushId}
    date, desc, amount, paidBy, splitMode, splitWith?, createdBy, createdAt
  budget/
    target, currency
  locations/
    hotel/{0..6}        # override fields for HOTELS[i]
    activity/{0..3}     # override fields for ACTIVITIES[i]
```

**Family member roster:**
- Group 1 · 少年组 (7): Danzel, Alvin, Angel, Eugene, Vanessa, Jujuhub, Amazi
- Group 2 · 家人组 (10): Da ma · 大妈, Yi ma · 二妈, Coke Yi Yi, Mom · 妈, Dad · 爸, Uncle Yap, Uncle Jimy, Uncle Steven, Ah Gong · 阿公, Margaret

Override-layer rule: hardcoded JS arrays are seed; Firebase override fields win at render time. Removing the override at `locations/hotel/3` reverts to seed.

## Schema for expense splits

| Mode | Who pays the share |
|---|---|
| `all` | All 19 named family members |
| `group1` | The 7 names in Group 1 |
| `group2` | The 12 names in Group 2 |
| `custom` | The names selected via chip-picker (`splitWith` array) |

Settle-up: `paid[name] - share[name]`. Positive = others owe them. Negative = they owe.

## Manager PIN

Default `8888`. Override via:
```js
localStorage.setItem('yap_manager_pin', '1234');
```

## Resetting

- **Profile**: settings → Switch user, or `localStorage.removeItem('yap_profile')` in DevTools.
- **Manager mode**: tap the lock-open icon in the header.
- **Welcome screen**: settings → toggle "Skip welcome screen".
- **Ah Zai chat history**: open Ah Zai sheet → "Clear chat · 清除".
- **Firebase data**: open Firebase console → Realtime Database → delete the `yap-trip` node.

## Troubleshooting

- **"Firebase not configured" on Money tab** — `config.json` `firebase.databaseURL` is still `REPLACE_…`. Update and reload.
- **Ah Zai says "needs a MiniMax key"** — same: `config.json` `minimax.apiKey` is still placeholder.
- **Ah Zai is "napping"** — MiniMax request failed. Check console for CORS or auth errors. If CORS blocks browser calls, set up an Apps Script proxy (out of scope for v2.5).
- **Edits don't sync** — confirm Firebase rules allow read/write. In the Firebase console: Realtime Database → Rules tab → set both to `true` (test mode) for the trip duration.

## Out of scope (deliberately)

- Yelp / Google Places auto-fetch (dropped).
- Popular menu items (drop, will hard-code in food sheet later).
- Apps Script proxy for MiniMax (user OK with browser-exposed key).
- Auto-geocoding addresses (manager enters lat/lng manually when editing).
