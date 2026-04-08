# Break Forecast 6-Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix 6 bugs/improvements in break-forecast.html: event listener stacking, scroll loss, auto-save, grace period flagging, undo for destructive actions, and URL hash multi-device sharing.

**Architecture:** Single self-contained HTML file (`break-forecast.html`). All changes are in the `<script>` block. No build tools, no frameworks, no external dependencies. Manual browser testing only (no test framework).

**Tech Stack:** Vanilla HTML/CSS/JS, localStorage, URL hash encoding via `btoa`/`atob`.

---

## File Structure

- Modify: `break-forecast.html` (single file, all changes)
  - CSS additions: `.b-grace-warn` class, `.toast-undo` styles
  - JS changes: event delegation, scroll preservation, auto-save, grace period, undo, URL hash sharing

---

### Task 1: Move event listeners out of renderGrid (Fix 2)

**Files:**
- Modify: `break-forecast.html:1657-1701` (remove listeners from renderGrid)
- Modify: `break-forecast.html:1704-1714` (DOMContentLoaded block — add delegation here)

**Problem:** Three event listeners (row-remove click, sent-increment click, sent-decrement contextmenu) are attached to `#forecastTable` inside `renderGrid()`. Every render adds duplicates. After N interactions, N listeners fire per click.

- [ ] **Step 1: Remove the 3 event listeners from inside renderGrid()**

Delete lines 1657-1701 from `renderGrid()`. The function should end after `syncTopScroll(); renderNextUp();` (line 1655). The closing `}` of renderGrid moves up to right after `renderNextUp();`.

**Before (end of renderGrid):**
```javascript
  wrap.innerHTML = html;
  syncTopScroll();
  renderNextUp();

  var table = document.getElementById('forecastTable');

  // Remove one shift from row (delete row if count hits 0)
  table.addEventListener('click', function(e) {
    var rm = e.target.closest('.row-remove');
    if (rm) {
      var idx = parseInt(rm.getAttribute('data-remove'));
      var row = shiftRows[idx];
      if (row) {
        row.count--;
        if (row.count <= 0) shiftRows.splice(idx, 1);
      }
      renderGrid();
      return;
    }
  });

  // Left-click: increment sent count
  table.addEventListener('click', function(e) {
    var td = e.target;
    while (td && td.tagName !== 'TD') td = td.parentElement;
    if (!td || !td.getAttribute('data-row')) return;
    var rowIdx = parseInt(td.getAttribute('data-row'));
    var bType  = td.getAttribute('data-break');
    var row = shiftRows[rowIdx];
    if (!row || !row.sent) return;
    var cur = row.sent[bType] || 0;
    row.sent[bType] = cur >= row.count ? 0 : cur + 1;
    renderGrid();
  });

  // Right-click: decrement sent count
  table.addEventListener('contextmenu', function(e) {
    var td = e.target;
    while (td && td.tagName !== 'TD') td = td.parentElement;
    if (!td || !td.getAttribute('data-row')) return;
    e.preventDefault();
    var rowIdx = parseInt(td.getAttribute('data-row'));
    var bType  = td.getAttribute('data-break');
    var row = shiftRows[rowIdx];
    if (!row || !row.sent) return;
    var cur = row.sent[bType] || 0;
    row.sent[bType] = cur <= 0 ? row.count : cur - 1;
    renderGrid();
  });
}
```

**After (end of renderGrid):**
```javascript
  wrap.innerHTML = html;
  syncTopScroll();
  renderNextUp();
}
```

- [ ] **Step 2: Add delegated event listeners on `#gridWrap` inside DOMContentLoaded**

In the existing DOMContentLoaded block (~line 1705), after the scroll sync setup, add the three delegated listeners on `#gridWrap` (which is stable — never replaced by innerHTML):

```javascript
  // Delegated event listeners on gridWrap (stable container)
  // Row remove + sent increment (both on click)
  wrap.addEventListener('click', function(e) {
    // Row remove
    var rm = e.target.closest('.row-remove');
    if (rm) {
      var idx = parseInt(rm.getAttribute('data-remove'));
      var row = shiftRows[idx];
      if (row) {
        row.count--;
        if (row.count <= 0) shiftRows.splice(idx, 1);
      }
      saveShifts();
      renderGrid();
      return;
    }
    // Sent increment (left-click on break cell)
    var td = e.target;
    while (td && td.tagName !== 'TD') td = td.parentElement;
    if (!td || !td.getAttribute('data-row')) return;
    var rowIdx = parseInt(td.getAttribute('data-row'));
    var bType  = td.getAttribute('data-break');
    var row = shiftRows[rowIdx];
    if (!row || !row.sent) return;
    var cur = row.sent[bType] || 0;
    row.sent[bType] = cur >= row.count ? 0 : cur + 1;
    saveShifts();
    renderGrid();
  });

  // Sent decrement (right-click on break cell)
  wrap.addEventListener('contextmenu', function(e) {
    var td = e.target;
    while (td && td.tagName !== 'TD') td = td.parentElement;
    if (!td || !td.getAttribute('data-row')) return;
    e.preventDefault();
    var rowIdx = parseInt(td.getAttribute('data-row'));
    var bType  = td.getAttribute('data-break');
    var row = shiftRows[rowIdx];
    if (!row || !row.sent) return;
    var cur = row.sent[bType] || 0;
    row.sent[bType] = cur <= 0 ? row.count : cur - 1;
    saveShifts();
    renderGrid();
  });
```

Note: The `saveShifts()` calls are included here — this simultaneously implements Fix 1 (auto-save) for click-to-send. The `wrap` variable is already defined in this DOMContentLoaded scope.

- [ ] **Step 3: Verify in browser**

Open `break-forecast.html`, enter password, paste shifts, interact with the grid multiple times. Confirm:
- Clicking ✕ removes a shift (no duplicate removal)
- Left-clicking a past break cell increments sent count (no duplicate increments)
- Right-clicking decrements (no duplicate decrements)
- After 10+ interactions, behavior is still 1:1 (proves no listener stacking)

- [ ] **Step 4: Commit**

```bash
cd "/c/Users/diazc/OneDrive/Desktop/PASS Work"
git add break-forecast.html
git commit -m "fix: move event listeners out of renderGrid to prevent stacking"
```

---

### Task 2: Scroll position preserved on re-render (Fix 3)

**Files:**
- Modify: `break-forecast.html` — inside `renderGrid()`, around the `wrap.innerHTML = html` line

**Problem:** `innerHTML` replacement resets `scrollLeft` to 0. Grid jumps back to the left on every click interaction.

- [ ] **Step 1: Save and restore scrollLeft around innerHTML replacement**

In `renderGrid()`, find:
```javascript
  wrap.innerHTML = html;
  syncTopScroll();
  renderNextUp();
}
```

Replace with:
```javascript
  var savedScroll = wrap.scrollLeft;
  wrap.innerHTML = html;
  wrap.scrollLeft = savedScroll;
  var topBar = document.getElementById('gridScrollTop');
  if (topBar) topBar.scrollLeft = savedScroll;
  syncTopScroll();
  renderNextUp();
}
```

- [ ] **Step 2: Verify in browser**

Scroll grid to the right, click a break cell. Confirm grid does NOT jump back to the left.

- [ ] **Step 3: Commit**

```bash
cd "/c/Users/diazc/OneDrive/Desktop/PASS Work"
git add break-forecast.html
git commit -m "fix: preserve grid scroll position on re-render"
```

---

### Task 3: Auto-save after click-to-send (Fix 1)

**Files:**
- Modify: `break-forecast.html` — `completeAllDue()` function

**Note:** The click-to-send auto-save was already handled in Task 1 (saveShifts() calls added in the delegated listeners). This task only adds the missing `saveShifts()` call to `completeAllDue()`.

- [ ] **Step 1: Add saveShifts() to completeAllDue()**

Find:
```javascript
function completeAllDue() {
  if (currentTimeMins === null || shiftRows.length === 0) return;
  shiftRows.forEach(function(row) {
    row.breaks.forEach(function(b) {
      if (b.wEnd <= currentTimeMins) {
        row.sent[b.type] = row.count;
      }
    });
  });
  renderGrid();
}
```

Replace with:
```javascript
function completeAllDue() {
  if (currentTimeMins === null || shiftRows.length === 0) return;
  shiftRows.forEach(function(row) {
    row.breaks.forEach(function(b) {
      if (b.wEnd <= currentTimeMins) {
        row.sent[b.type] = row.count;
      }
    });
  });
  saveShifts();
  renderGrid();
}
```

- [ ] **Step 2: Verify in browser**

Click "Complete All Due", then check `localStorage.getItem('bf_shiftRows')` in console — sent values should be persisted.

- [ ] **Step 3: Commit**

```bash
cd "/c/Users/diazc/OneDrive/Desktop/PASS Work"
git add break-forecast.html
git commit -m "fix: auto-save shifts after completeAllDue"
```

---

### Task 4: Grace period flagging (Fix 6)

**Files:**
- Modify: `break-forecast.html` — CSS section (add `.b-grace-warn` class)
- Modify: `break-forecast.html` — inside `renderGrid()`, break cell rendering logic

**Problem:** When two break windows for the same shift are < 60 min apart, there's no visual indicator. Supervisors must mentally enforce 1hr spacing.

- [ ] **Step 1: Add CSS class for grace period warning**

After the `.b-done-cell` rule (around line 821), add:

```css
    .b-grace-warn { border-left: 3px solid #f59e0b !important; }
```

- [ ] **Step 2: Pre-compute grace flags per row before building HTML**

Inside `renderGrid()`, after the `shiftRows.forEach` that builds the HTML starts (line 1517), but before the slot loop, add grace period detection for each row. Insert this block right after `html += '<td class="td-count">' + row.count + '</td>';` (line 1520):

```javascript
    // Grace period detection: flag break windows < 60min apart
    var graceSlots = {};
    for (var bi = 1; bi < row.breaks.length; bi++) {
      var prev = row.breaks[bi - 1];
      var curr = row.breaks[bi];
      var gap = curr.wStart - prev.wEnd;
      if (gap < 60 && !((row.sent[prev.type] || 0) >= row.count && (row.sent[curr.type] || 0) >= row.count)) {
        // Flag all slots covered by the second break's window
        for (var gs = curr.wStart; gs < curr.wEnd; gs += 30) {
          graceSlots[gs] = true;
        }
      }
    }
```

Then in the cell rendering, when building `<td>` elements for break cells (both single-break and multi-break), append `b-grace-warn` class if `graceSlots[t]` is true.

For single-break cells (the `else` branch at line ~1553), change the td output to include the grace class:

**Before:**
```javascript
        var isDone   = sentAmt >= row.count;
        var trackCls = isDone ? ' b-done-cell' : ' b-track';
        if (isPast) {
          var countTxt = sentAmt > 0 ? sentAmt + '/' + row.count : '';
          html += '<td class="td-cell ' + bCls + trackCls + timeCls + '" data-row="' + rowIdx + '" data-break="' + bType + '">' + countTxt + '</td>';
        } else {
          var countTxt = sentAmt > 0 ? sentAmt + '/' + row.count : '';
          html += '<td class="td-cell ' + bCls + trackCls + timeCls + '" data-row="' + rowIdx + '" data-break="' + bType + '">' + countTxt + '</td>';
        }
```

**After:**
```javascript
        var isDone   = sentAmt >= row.count;
        var trackCls = isDone ? ' b-done-cell' : ' b-track';
        var graceCls = graceSlots[t] ? ' b-grace-warn' : '';
        if (isPast) {
          var countTxt = sentAmt > 0 ? sentAmt + '/' + row.count : '';
          html += '<td class="td-cell ' + bCls + trackCls + graceCls + timeCls + '" data-row="' + rowIdx + '" data-break="' + bType + '">' + countTxt + '</td>';
        } else {
          var countTxt = sentAmt > 0 ? sentAmt + '/' + row.count : '';
          html += '<td class="td-cell ' + bCls + trackCls + graceCls + timeCls + '" data-row="' + rowIdx + '" data-break="' + bType + '">' + countTxt + '</td>';
        }
```

For multi-break (overlap) cells (~line 1546), add grace class too:

**Before:**
```javascript
        var grad = 'repeating-linear-gradient(45deg,' + stops.join(',') + ')';
        html += '<td class="td-cell' + timeCls + '" style="background:' + grad + '"></td>';
```

**After:**
```javascript
        var grad = 'repeating-linear-gradient(45deg,' + stops.join(',') + ')';
        var graceCls = graceSlots[t] ? ' b-grace-warn' : '';
        html += '<td class="td-cell' + graceCls + timeCls + '" style="background:' + grad + '"></td>';
```

- [ ] **Step 3: Verify in browser**

Add a shift like `0300-1300` (10hr). This produces 4 breaks. Check if any adjacent break windows are < 60min apart and show the amber left border on the second break's cells.

- [ ] **Step 4: Commit**

```bash
cd "/c/Users/diazc/OneDrive/Desktop/PASS Work"
git add break-forecast.html
git commit -m "feat: amber border flagging when breaks are < 60min apart"
```

---

### Task 5: Undo for destructive actions (Fix 4)

**Files:**
- Modify: `break-forecast.html` — CSS (toast undo styling)
- Modify: `break-forecast.html` — `showToast()` function
- Modify: `break-forecast.html` — `clearAll()` and `completeAllDue()` functions
- Add: `undoSnapshot` variable and `undoRestore()` function

- [ ] **Step 1: Add CSS for undo link in toast**

After the `#toast.show` rule (~line 780), add:

```css
    #toast a.toast-undo {
      color: var(--accent);
      font-weight: 700;
      margin-left: 12px;
      cursor: pointer;
      text-decoration: underline;
    }
```

- [ ] **Step 2: Add undoSnapshot variable and undoRestore function**

Right after `var shiftRows = [];` (line 1151), add:

```javascript
var undoSnapshot = null;
var undoTimer = null;

function undoRestore() {
  if (!undoSnapshot) return;
  shiftRows = undoSnapshot;
  undoSnapshot = null;
  clearTimeout(undoTimer);
  saveShifts();
  renderGrid();
  showToast('Restored');
}
```

- [ ] **Step 3: Modify showToast to support an undo action**

Replace the existing `showToast` function:

**Before:**
```javascript
var toastTimer;
function showToast(msg) {
  var t = document.getElementById('toast');
  t.textContent = msg;
  t.classList.add('show');
  clearTimeout(toastTimer);
  toastTimer = setTimeout(function() { t.classList.remove('show'); }, 2000);
}
```

**After:**
```javascript
var toastTimer;
function showToast(msg, undoable) {
  var t = document.getElementById('toast');
  if (undoable) {
    t.innerHTML = msg + ' <a class="toast-undo" onclick="undoRestore()">Undo</a>';
  } else {
    t.textContent = msg;
  }
  t.classList.add('show');
  clearTimeout(toastTimer);
  var duration = undoable ? 5000 : 2000;
  toastTimer = setTimeout(function() {
    t.classList.remove('show');
    if (undoable) undoSnapshot = null;
  }, duration);
}
```

- [ ] **Step 4: Add snapshot to clearAll()**

**Before:**
```javascript
function clearAll() {
  shiftRows = [];
  document.getElementById('pasteArea').value = '';
```

**After:**
```javascript
function clearAll() {
  if (shiftRows.length === 0) return;
  undoSnapshot = JSON.parse(JSON.stringify(shiftRows));
  shiftRows = [];
  document.getElementById('pasteArea').value = '';
```

Also change the `showToast('Cleared')` call at the end of clearAll to `showToast('Cleared', true)`.

- [ ] **Step 5: Add snapshot to completeAllDue()**

**Before:**
```javascript
function completeAllDue() {
  if (currentTimeMins === null || shiftRows.length === 0) return;
  shiftRows.forEach(function(row) {
```

**After:**
```javascript
function completeAllDue() {
  if (currentTimeMins === null || shiftRows.length === 0) return;
  undoSnapshot = JSON.parse(JSON.stringify(shiftRows));
  shiftRows.forEach(function(row) {
```

Also change the end of completeAllDue to add a toast. After `renderGrid();`, add:
```javascript
  showToast('All due completed', true);
```

- [ ] **Step 6: Verify in browser**

1. Add shifts, click "Clear" — toast shows "Cleared Undo" for 5 seconds
2. Click Undo within 5s — shifts reappear
3. Add shifts, set time, click "Complete All Due" — toast shows with Undo
4. Click Undo — sent counts restored to pre-complete state

- [ ] **Step 7: Commit**

```bash
cd "/c/Users/diazc/OneDrive/Desktop/PASS Work"
git add break-forecast.html
git commit -m "feat: undo for Clear All and Complete All Due with 5s toast"
```

---

### Task 6: Multi-device state sharing via URL hash (Fix 5)

**Files:**
- Modify: `break-forecast.html` — header HTML (add Share button)
- Modify: `break-forecast.html` — JS (encode/decode functions, load-from-hash in applyMode)

- [ ] **Step 1: Add Share button in header**

In the header-actions div, after the Save button, add:

**Before:**
```html
  <div class="header-actions">
    <button class="save-btn admin-only" id="saveBtn" onclick="manualSave()">Save</button>
    <button class="theme-btn" id="themeBtn" onclick="toggleTheme()">&#9728;&#65039;</button>
  </div>
```

**After:**
```html
  <div class="header-actions">
    <button class="save-btn admin-only" id="saveBtn" onclick="manualSave()">Save</button>
    <button class="save-btn admin-only" id="shareBtn" onclick="shareState()">Share</button>
    <button class="theme-btn" id="themeBtn" onclick="toggleTheme()">&#9728;&#65039;</button>
  </div>
```

- [ ] **Step 2: Add shareState() and loadFromHash() functions**

After the `manualSave()` function, add:

```javascript
function shareState() {
  if (shiftRows.length === 0) {
    showToast('Nothing to share');
    return;
  }
  var encoded = btoa(JSON.stringify(shiftRows));
  var url = window.location.origin + window.location.pathname + '#data=' + encoded;
  navigator.clipboard.writeText(url).then(function() {
    showToast('Link copied');
  }).catch(function() {
    // Fallback: select a temporary input
    var tmp = document.createElement('input');
    tmp.value = url;
    document.body.appendChild(tmp);
    tmp.select();
    document.execCommand('copy');
    document.body.removeChild(tmp);
    showToast('Link copied');
  });
}

function loadFromHash() {
  var hash = window.location.hash;
  if (!hash || hash.indexOf('#data=') !== 0) return false;
  try {
    var encoded = hash.slice(6); // remove '#data='
    var decoded = JSON.parse(atob(encoded));
    if (Array.isArray(decoded) && decoded.length > 0) {
      shiftRows = decoded;
      saveShifts(); // persist to localStorage so it survives refresh
      window.location.hash = ''; // clean URL
      showToast('Loaded shared data');
      return true;
    }
  } catch(e) {}
  return false;
}
```

- [ ] **Step 3: Call loadFromHash() in applyMode() before loadShifts()**

**Before:**
```javascript
function applyMode() {
  // Load shifts first — prevents race where renderGrid fires before DOMContentLoaded
  loadShifts();
```

**After:**
```javascript
function applyMode() {
  // Load from shared URL hash first, fall back to localStorage
  if (!loadFromHash()) {
    loadShifts();
  }
```

- [ ] **Step 4: Verify in browser**

1. Add shifts, click Share — URL copied to clipboard
2. Open the URL in a different browser/incognito — shifts appear immediately
3. Confirm URL hash is cleaned after loading
4. Confirm data persists in localStorage after loading from hash

- [ ] **Step 5: Commit**

```bash
cd "/c/Users/diazc/OneDrive/Desktop/PASS Work"
git add break-forecast.html
git commit -m "feat: Share button encodes shifts to URL hash for multi-device sync"
```

---

## Implementation Order Summary

1. **Task 1** — Event listener delegation (Fix 2) — must go first, changes renderGrid structure
2. **Task 2** — Scroll preservation (Fix 3) — pairs with Task 1
3. **Task 3** — Auto-save completeAllDue (Fix 1) — click-to-send covered by Task 1
4. **Task 4** — Grace period flagging (Fix 6) — renderGrid enhancement
5. **Task 5** — Undo (Fix 4) — toast modification
6. **Task 6** — URL hash sharing (Fix 5) — independent, biggest addition

## Final Commit

After all 6 tasks, one final push:
```bash
cd "/c/Users/diazc/OneDrive/Desktop/PASS Work"
git push
```
