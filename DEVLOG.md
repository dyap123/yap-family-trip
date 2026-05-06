# Yap Family Trip · Dev Log

## 2026-05-05 — Big v2.5 build day

The whole `index-v2.html` v2.5 was built in this single session. Live at
https://dyap123.github.io/yap-family-trip/index-v2.html.

### Where we left off (pick up tomorrow here)

- Days **1–7 are fully detailed** in the timeline (per-row time, who, link, note, group tag).
- Days **8–13 are general-plan only** (Malibu Day 8 → Paso Robles Day 9 → SJ home Days 10-13). Marked "TBD" in the calendar / day strip; map info card stays helpful.
- Detailed timeline editing still left to do:
  - Day 8 (May 21) — OC → Malibu. PCH drive, El Matador sunset. Eugene departs LAX morning. **Day 8 morning: breakfast at Victory Diner Orange (artery warning)** — already noted as a "Tomorrow AM" hint in Day 7's timeline; needs to live properly on Day 8.
  - Day 9 (May 22) — Malibu → Paso. Malibu Farm brunch, Tin City evening.
  - Day 10 (May 23) — Paso → SJ. Brown Butter Cookies stop.
  - Day 11 (May 24) — SJ. Pickleball + brunch.
  - Day 12 (May 25) — SJ. Free / Santana Row shopping. Ee departs.
  - Day 13 (May 26) — Pack up, farewell brunch, SFO drop-offs.

### Built / shipped today

- **v2.5 visual rebuild** — neon Chongqing palette (deep navy + hot pink/cyan/yellow/magenta neon), bilingual EN/中文 throughout, glassmorphic cards, vertical Chinese signage.
- **5 family-mode tabs** (Today / Trip / Map / Food / Family) + manager-only Money tab.
- **User profiles** — pick your name on first launch from a 17-name picker. Group 1 (6) + Group 2 (10) + Alyssa (added Day 3). Bilingual labels (Da ma · 大妈, etc.). Settings → Switch user.
- **Group filter on split days** — split sub-cards highlight your group, dim the other.
- **Manager mode** — Danzel + Angel only. PIN `050103` (6 digits). Switching to a non-eligible profile auto-revokes.
- **Firebase money** — RTDB-backed expenses with split modes (all / G1 / G2 / pick people) + per-expense **excluded** list. Live `computeSettle`, settle-up table, gold + FAB on Money tab.
- **Family-only spend chip** in the header (USD + SGD, hidden in manager mode). Updates live via Firebase listener.
- **Edit-location override layer** — manager pencils on hotel cards write to `yap-trip/locations/`, render layer merges seed + override with a "Reset · 还原" button.
- **Ah Zai 阿仔** — floating pink avatar bottom-right, Anthropic-compat MiniMax endpoint (`api.minimax.io/anthropic/v1/messages`, model `MiniMax-M2.7`). Reads key from Firebase `config/minimax_key`. Chat history per profile. Trash icon clears.
- **Map immersion** — Today pin pulses yellow with 「今天 · TODAY」 badge, progressive polyline lights up segments through the active day, info card above the map shows the current focused day's plan, day strip auto-scrolls with the tour.
- **Trip tab calendar/list toggle** — 7-col May grid view; tap cell opens a single-day modal with full timeline + drag-handle dismiss + bottom "Close · 关闭" button + corner X.
- **Announcements** — Firebase-backed `yap-trip/announcements/`. Manager-only "+ Add announcement" dashed yellow card on Today/Family. Bilingual EN + 中文. Tap to edit/delete (manager only). First announcement seeded ("Charge phones · stay hydrated").
- **Detailed Day 1–7 timelines** with Schedule · 时间表 sections beneath each day card. Group tags (EDC / REST / G1 / G2 / ALL) dim rows for the other group.

### Wrinkles fixed this session

- MiniMax `**markdown**` was rendering as raw `**` — added `mdInline` rendering for bold/italic/code/newlines.
- iPhone keyboard was hiding the chat reply — switched chat sheet to `dvh` + `visualViewport` resize listener + autoscroll on focus.
- iPhone bottom bar had a gap — body had a duplicate `safe-area` padding that stacked with the nav's. Removed body's padding-bottom; nav handles its own safe-area inset.
- MiniMax was conflating Bicycle Club + Commerce Casino — system prompt now lists every hotel separately by name + city + dates + hosts.
- HOTELS dates were wrong (Bicycle Club May 15-16 → 15-17; Horseshoe 16-18 → 16-19; Pixar/Commerce 19-20 → 19-21). Pixar and Commerce are now two separate HOTELS entries.
- **Major attribution error caught at end of session**: graduation is **Danzel's** at Cal Poly Pomona, not Angel's. Fixed everywhere. LAX pickup for Eugene + Vanessa is **12 AM May 16**, not May 15. Fixed everywhere. System prompt has a hard rule for Ah Zai now.

### Roster (final)

- **Group 1 · 少年组 (7)**: Danzel, Alvin, Alyssa, Angel, Eugene, Vanessa, Amazi
- **Group 2 · 家人组 (10)**: Da ma · 大妈, Yi ma · 二妈, Aunty Coke · 可乐姨, Mom · 妈妈, Dad · 爸爸, Uncle Jimy, Steve Susu · 叔叔, Ah Gong · 阿公, Margaret · 婆婆, Jujuhub · 蜘蛛侠

### Vehicles

1 large van (12 seats) + 2 Teslas (5 each) — 22 seats for 17 family. **No truck** (was removed earlier).

### Known TODO / pickup items

1. **Days 8–13 detailed timelines** — biggest remaining work.
2. **Claude Design guides** — Angel manager + family guides via the prompts shipped in chat. The print-shadow/backdrop-filter `@media print` fix needs to be in the prompt for next regeneration.
3. **iPhone testing pass** after each batch of changes — the user has been doing this in real-time.
4. **Photo overrides** — currently 4 Wikimedia Commons photos for Boston Lobster, Victory Diner, Vegas Chinatown spots, Cheesecake Factory. User can curate better images via the Sheet (or extend `FOOD_PHOTO_OVERRIDES` in JS) when ready.
5. **Maybe later** — Apps Script writeback for food (deferred), more announcements, fine-tune Ah Zai prompts.

### Files

- `index-v2.html` — the SPA (~4000 lines now, single file)
- `index.html` — v1 fallback, kept up-to-date with name changes
- `config.json` — Firebase RTDB URL (shared with CUP), MiniMax overridable
- `README-v2.md` — user-facing setup
- `BUILD.md` — architecture + extension recipes
- `guides/manager-guide.html` — early HTML guide; superseded by Claude Design output
- `DEVLOG.md` — this file

### Tomorrow's plan (in priority order)

1. **Day 8 (May 21) timeline** — OC → Malibu. PCH drive, El Matador sunset. Eugene departs LAX morning. Move the Victory Diner Orange breakfast (currently a "Tomorrow AM" hint at end of Day 7) onto Day 8 properly.
2. **Build in-app timeline-row editor** for managers. Mirror the hotel-edit pattern:
   - Yellow pencil icon next to each Schedule · 时间表 row in manager mode
   - Tap → modal with fields: Time / Event / Who / Note / Link / Icon / Group
   - Saves to Firebase override path `yap-trip/timelines/{day}/{rowId}` and merges with the FALLBACK seed at render time
   - "+ Add row" button at the bottom of each day's timeline (manager only)
   - Reorder + delete
   - This unblocks Danzel/Angel from needing me for every timeline tweak
3. **Days 9–13 timelines** — Paso, return to SJ, free days, departures.
4. **Photos** — user can curate specific image URLs into `FOOD_PHOTO_OVERRIDES` or the sheet when ready.

### Pickup commands tomorrow

```sh
cd ~/yap-family-trip
git pull
python3 -m http.server 8282
open "http://localhost:8282/index-v2.html"
```

Latest commit at end-of-session: `a7a2b32`.
