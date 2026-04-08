# Dexter's Lab — Break Forecast Sandbox + OT Prediction

**Date:** 2026-04-08
**File:** `break-forecast-lab.html`
**Status:** Approved

---

## Overview

A full sandbox clone of `break-forecast.html` called "Dexter's Lab." Separate data, admin-only access, accessible from hub only. First feature built on top: OT Prediction — a forward simulation that flags when remaining breaks can't be completed within their windows given current staffing surplus, suggesting OT extension may be needed.

---

## Page Structure

**File:** `break-forecast-lab.html`
**Title:** Dexter's Lab
**Subtitle:** Break forecast sandbox

**Base:** Full clone of `break-forecast.html` including all 6 fixes (event delegation, scroll preservation, auto-save, grace period flagging, undo, URL hash sharing).

**Access:**
- Admin-only — password `1781` only, no viewer mode (password `1234` rejected)
- Accessible from hub card on `index.html` only
- No bottom nav entry on other tool pages

**Data isolation:**
- `bflab_shiftRows` — shift data (separate from `bf_shiftRows`)
- `bflab_staffing` — staffing bar state (separate from `bf_staffing`)
- `sessionStorage` auth key shared — logging into any THGOT tool carries over

**Hub card on `index.html`:**
- Icon: beaker emoji
- Title: "Dexter's Lab"
- Subtitle: "Break forecast sandbox"
- Admin-only visibility (hidden until admin password entered, same pattern as Classified)

---

## OT Prediction Engine

### Inputs (all already exist on the page)

1. **`shiftRows`** — array of shift objects with `start`, `duration`, `count`, `breaks[]`, `sent{}`, `isOT`
2. **`currentTimeMins`** — current time in minutes from midnight (live clock or manual)
3. **Required staffing** — derived from mini staffing calculator state:
   - Zone A: baseline (3) + per-lane counts based on active lanes, or 9 if Skelly
   - Zone B: 14 if on (7 if Skelly), 0 if off
   - H: time-based schedule (CR + L3 + O/S), or manual toggles if Skelly
   - NPS: time-based (L0 + L1 + L2 + G + V)
4. **Headcount per slot** — already computed in `renderGrid()` as `totalsHC[]` (sum of `row.count` for shifts covering each 30-min slot)

### Simulation Logic

Runs every time `renderGrid()` runs (clock tick, break sent, shift added/removed, staffing bar change).

**Algorithm:**

1. **Gather unsent breaks.** For each `shiftRow`, for each break where `sent[type] < count` and `wEnd > currentTimeMins`, create an entry:
   - `{ rowIdx, type, remaining: count - sent, wStart, wEnd, duration: (type === 'Lunch' ? 35 : 20) }`

2. **Sort by urgency.** Order by `wEnd` ascending (earliest deadline first).

3. **Walk forward in 30-min slots** from current time to the latest `wEnd`.

4. **Per slot, calculate surplus:**
   - `headcount` = number of shift workers on the floor at that slot (from `totalsHC[]`)
   - `required` = staffing requirement at that slot's midpoint time (call existing `calcMiniStaffing` logic for that time — Zone A + Zone B + H + NPS)
   - `surplus = headcount - required`
   - `availableToSend = max(0, surplus)`

5. **Per slot, process breaks:**
   - From the urgency-sorted list, take up to `availableToSend` breaks whose window covers this slot (`wStart <= slotStart` and `wEnd > slotStart`)
   - Mark each as "scheduled" — decrement its `remaining` count
   - If `remaining` hits 0, remove from the list

6. **After walking all slots:** any breaks still in the list with `remaining > 0` are **missed**.

7. **Calculate OT extension:** sum of `missed.remaining * missed.duration` across all missed breaks.

### Status Classification

- **On Track** (green): no projected misses
- **At Risk** (amber): all breaks can be completed, but the tightest slot has surplus <= 1 AND there are breaks due within 60 minutes
- **OT Likely** (red): one or more breaks projected to be missed

### Required Staffing Per Slot

The mini staffing calculator currently computes required staffing for `currentTimeMins` only. For the OT simulation, we need required staffing at *each future slot*. This means extracting the staffing calculation into a pure function:

```
function getRequiredStaffing(timeMins, staffingState) → number
```

Where `staffingState` captures: Zone B on/off, Zone B Skelly, Zone A Skelly, active lanes, H Skelly toggles. Zone A and Zone B counts are time-independent (based on toggles/lanes), but H and NPS are time-dependent (schedule-based). The function evaluates H and NPS at the given `timeMins` and adds the toggle-based Zone A/B counts.

---

## OT Prediction Panel — UI

**Location:** Right column, below the existing Next Up panel.

**Layout:**

```
+---------------------------+
|  OT PREDICTION            |
|  [STATUS BADGE]           |
|                           |
|  Bottleneck               |
|  14:30 — surplus 1        |
|                           |
|  Projected Misses         |
|  0300-1100  Lunch  08:00  |
|  0400-1200  3rd    09:30  |
|                           |
|  Est. Extension           |
|  55 min                   |
+---------------------------+
```

**Status badge:**
- Green pill: "On Track"
- Amber pill: "At Risk"
- Red pill: "OT Likely"
- Uses same color system as Next Up countdown badges

**Bottleneck slot:**
- Shows the 30-min slot with the lowest surplus relative to break demand
- Format: `HH:MM — surplus N`

**Projected misses list:**
- Each entry: shift label, break type, window close time
- Sorted by window close time
- Only appears when status is "At Risk" or "OT Likely"

**Est. Extension:**
- Total minutes of missed break time
- Only appears when status is "OT Likely"

**Empty state:** When no shifts are entered or no time is set: "Add shifts and set time to see predictions"

---

## Reactive Update Flow

The OT simulation re-runs whenever any of these change:

| Trigger | How |
|---|---|
| Live clock tick (every 5s) | `renderGrid()` is called, which calls `runOTPrediction()` |
| Break marked sent (click/right-click) | `renderGrid()` is called after sent update |
| Shift added/removed | `renderGrid()` is called after shiftRows change |
| Staffing bar toggle (Zone B, lanes, Skelly) | `calcMiniStaffing()` triggers, then calls `runOTPrediction()` |
| Manual time change | `renderGrid()` is called via `onManualTime()` |

Implementation: `runOTPrediction()` is called at the end of `renderGrid()` (after `renderNextUp()`), and also hooked into `calcMiniStaffing()`.

---

## Changes to Hub (`index.html`)

Add a new card for Dexter's Lab. Admin-only — hidden by default, shown only after admin password is entered. Same pattern as the Classified tool card.

- Icon: beaker emoji
- Title: "Dexter's Lab"
- Subtitle: "Break forecast sandbox"
- Links to: `break-forecast-lab.html`

---

## What Is NOT In Scope

- No viewer mode — admin only
- No bottom nav link on other pages
- No position-level modeling (which officer is where)
- No roster/name integration
- No OT callout integration (this flags the need, callout.html tracks the execution)
- No changes to the production `break-forecast.html`
