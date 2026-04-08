# Break Forecast — 6-Fix Improvement Spec

**Date:** 2026-04-08
**File:** `break-forecast.html`
**Status:** Approved

---

## Fix 1: Auto-save after click-to-send

**Problem:** Marking breaks as sent (left-click/right-click on cells) updates `row.sent` in memory but never calls `saveShifts()`. Browser crash = lost tracking.

**Fix:** Call `saveShifts()` after every sent increment/decrement and after `completeAllDue()`.

**Scope:** 3 lines added.

---

## Fix 2: Event listener stacking

**Problem:** `renderGrid()` attaches click and contextmenu handlers to `#forecastTable` on every call. After N interactions, N duplicate listeners fire per click.

**Fix:** Move event delegation to `#gridWrap` (stable container). Attach once on DOMContentLoaded. `renderGrid()` only rebuilds HTML — no listener attachment.

**Scope:** Move ~40 lines out of `renderGrid()`, attach once.

---

## Fix 3: Scroll position preserved on re-render

**Problem:** `innerHTML` replacement resets `scrollLeft` to 0. Grid jumps left on every interaction.

**Fix:** Save `gridWrap.scrollLeft` before innerHTML replacement, restore after. Same for the synced top scrollbar.

**Scope:** 4 lines in `renderGrid()`.

---

## Fix 4: Undo for destructive actions

**Problem:** Clear All and Complete All Due are instant and irreversible.

**Fix:**
- Before executing, snapshot `shiftRows` via `JSON.parse(JSON.stringify(shiftRows))`.
- Show toast with "Undo" link. 5-second window.
- Click Undo → restore snapshot, re-render, clear toast.
- Only one undo level (latest action). New action replaces previous snapshot.

**Scope:** ~25 lines. One `undoSnapshot` variable, modified `showToast()` to accept optional action, undo restore function.

---

## Fix 5: Multi-device state sharing via URL hash

**Problem:** localStorage is per-browser. Admin pastes shifts on one computer, viewer on another sees nothing.

**Fix:**
- Admin clicks "Share" button → `shiftRows` compressed to base64 string → appended to URL hash → URL copied to clipboard.
- On page load, if URL has `#data=...`, decode and load that state instead of localStorage.
- Toast confirms "Link copied" on share, "Loaded shared data" on receive.
- Hash data takes priority over localStorage on load. After loading from hash, save to localStorage so it persists.
- Viewer can open the shared URL and see the grid immediately.

**Encoding:** `JSON.stringify(shiftRows)` → `btoa()` (base64). Shift data is small (~2-5KB for 30 rows), well within URL limits.

**UI:** "Share" button next to "Save" in header (admin-only).

**Scope:** ~30 lines. Share button, encode/decode functions, load-from-hash in `applyMode()`.

---

## Fix 6: Grace period flagging

**Problem:** Break windows can be < 60 min apart for the same shift. Supervisor must mentally enforce 1hr spacing. No visual indicator.

**Fix:**
- During `renderGrid()`, for each shift row, check adjacent break windows.
- If two breaks have `wEnd(first)` to `wStart(second)` < 60 min AND neither is fully sent, flag the gap.
- Visual: amber left-border on the second break's cells in the overlap zone.
- CSS class: `.b-grace-warn` with `border-left: 2px solid #f59e0b`.

**Scope:** ~15 lines in renderGrid break-cell logic, 1 CSS class.

---

## Future (not in this spec)

- **OT prediction:** When remaining unsent breaks exceed what can physically be completed in remaining time, flag that OT extension may be needed. Builds on Next Up panel logic.

---

## Implementation Order

1. Fix 2 (event listeners) — must do first, renderGrid changes affect everything else
2. Fix 3 (scroll preservation) — small, pairs with Fix 2
3. Fix 1 (auto-save) — small, independent
4. Fix 6 (grace period) — renderGrid enhancement
5. Fix 4 (undo) — needs toast modification
6. Fix 5 (URL share) — independent, biggest addition
