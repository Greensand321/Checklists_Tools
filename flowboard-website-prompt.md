# FlowBoard — Full Website Implementation Prompt

## Project Overview

Build **FlowBoard**, a fully client-side Kanban board web application delivered as a single HTML file (no build step, no server required). The app stores all data in `localStorage` with optional Firebase Realtime Database sync for cross-device access. The existing prototype at `flowboard.html` contains the complete reference implementation; use this document to understand every page, panel, modal, and interaction so the final product is pixel-perfect and fully functional.

---

## Technology Stack

- **Single HTML file** — all CSS and JavaScript inline, no external dependencies except Firebase SDK loaded from CDN
- **Vanilla JavaScript** (no frameworks)
- **CSS custom properties** for theming (all colors via `var(--*)` tokens)
- **Firebase Realtime Database** (optional; app works fully offline without it)
- **LocalStorage** for persistence

---

## Global Layout

```
┌──────────────────────────────────────────────────────────────────┐
│  TOPBAR  (fixed height, frosted-glass blur, z-index 10)          │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  BOARD (flex:1, overflow-y:auto, padding 18px 22px 40px)          │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

`html, body` are `height:100%; overflow:hidden`. The board is the only scrollable region vertically. Each group row scrolls horizontally independently.

---

## Page 1 — Topbar

A single persistent top navigation bar. All controls live here.

### Left side (in order)
1. **Logo** — `⚡ FlowBoard`, 19px, font-weight 800, gradient text (`135deg, #60a5fa → #a78bfa → #fb7185`), no background clip fallback needed
2. **Stats chip** — live text showing `{n} groups · {n} cols · {n} cards` (counts active, non-archived items only), 11px, muted color

### Center (flex:1 spacer, then):
3. **Search input** — 190px wide, rounded 10px, placeholder `🔍 Search…`, filters cards by title and tags in real time as the user types (calls `render()` on `oninput`)

### Right side (in order)
4. **Sync status pill** — small rounded badge showing current sync state. Four states with matching colors:
   - `💾 Local` — muted gray (no Firebase key active)
   - `🔄 Syncing` — amber
   - `☁ Synced` — emerald green
   - `⚠ No Sync` — red
   - `📴 Offline` — slate

5. **☁ Sync** button (ghost style) — opens Cloud Sync modal
6. **🎨 Theme** button (ghost style) — opens Theme Panel dropdown
7. **↩ Undo** button (ghost style) — disabled when stack is empty; tooltip shows step count and `Ctrl+Z`
8. **↪ Redo** button (ghost style) — disabled when stack is empty; tooltip shows step count and `Ctrl+Y`
9. **💾 Backups** button (green style) — opens Backup & Restore modal
10. **📦 Archive** button (ghost style) — opens Archive modal
11. **＋ Group** button (ghost style) — opens Add Group modal
12. **＋ Column** button (primary blue-purple gradient) — opens Add Column modal

### Button styles
- `.btn-primary` — `linear-gradient(135deg, #3b82f6, #8b5cf6)`, white text, indigo glow shadow
- `.btn-ghost` — semi-transparent background, muted border
- `.btn-green` — emerald tint background + border
- `.btn-danger` — rose tint background + border
- `.btn-amber` — amber tint background + border
- All buttons: `border-radius: 9px`, `font-weight: 700`, hover reduces `opacity` to 0.85, active scales to 0.95 with `btnPop` keyframe animation

---

## Page 2 — Board (Main Canvas)

The board renders groups top-to-bottom. Each group contains a horizontal row of columns. Columns contain cards.

### Groups

Each group is a `<div class="group">` with:

**Group Header** (`group-hdr`):
- **Color accent dot** (11px circle) — clicking opens an inline color picker popup (`cpicker`) with 9 palette colors. The selected color (`pi` index into PAL array) sets the group's accent color and is used for header/body backgrounds of all its columns
- **Collapse toggle button** (`▼` / `▶`) — collapses/expands the entire group row
- **Group name** — bold, accent color; double-click enters inline rename mode (replaces text with a borderless input, saves on blur/Enter, cancels on Escape)
- **Card count chip** — `{n} cards`, gray rounded badge
- **Card Limit button** (`⊞ Limit` / `⊞ {n}`) — clicking opens a WIP popup (positioned below the button) with a number input and Set/Clear buttons. When a limit is set the button turns active (accent styled). The limit controls the visible card height of every column in the group — the column body animates to `limit × CARD_SLOT_H` px height; overflow scrolls
- **Inline color picker popup** — 9 colored dots, clicking applies and closes
- **+ icon button** — adds a column to this group (opens Add Column modal pre-set to this group)
- **× icon button** — deletes the group (columns become ungrouped), with confirm dialog; creates an auto-snapshot before deleting

**Group Body** — horizontal flex row (`group-cols`), `overflow-x: auto`, `align-items: stretch`. Contains column elements and a dashed `+` button at the end to add a column.

When a group is collapsed (`g.col === true`), the body row is not rendered.

**Ungrouped section** — columns not assigned to any group are collected into a fake group with title `📦 Ungrouped`. It has no controls (no delete, no limit, no color picker, no collapse).

**"Add Group" button** — rendered after all groups as a ghost button.

**Group entrance animation** — `anim-group-new` CSS class applied for 0.22s on creation.

---

### Columns

Each column is 268px min/max width, `flex-shrink: 0`, `border-radius: 14px`.

**Column Header** (`col-hdr`):
- Background uses `PAL[pi].h` (dark mode) or `hexToRgba(PAL[pi].a, 0.22)` (light mode)
- **Collapse button** (`◀`) — collapses column to a vertical pill; opacity 0.55
- **Column name** — bold, double-click enters inline rename
- **Card count badge** — shows active card count, accent color; clicking opens WIP limit popup (same as group limit but per-column — actually this is the column count chip; WIP is configured at group level)
- **Sort button** (`⇅` / `↑↓`) — toggles priority sort (High → Low). Active state uses accent color
- **Color dot** (13px circle) — clicking opens inline color picker for this column's palette index
- **Archive button** (`⊡`) — archives the entire column without deleting
- **Delete button** (`×`) — deletes column and all cards, confirm dialog, creates auto-snapshot

**Column is draggable** — hold mousedown on header for 300ms to initiate column drag. Ghost copy floats with `rotate(2deg) scale(1.04)`. A vertical blue line indicator shows drop position between columns within any group row. Dropping assigns the column to the target group.

**Column Body** (`col-body`):
- Background: `PAL[pi].b` (dark) or `hexToRgba(PAL[pi].a, 0.07)` (light)
- `padding: 8px 8px 10px`, `min-height: 56px`, `gap: 6px`
- During drag-over: blue border + inset glow (`drag-over` class)
- Empty state: centered 📭 icon + "No cards yet" text
- Cards rendered in order; if `sort` is true, sorted by priority desc before render
- **Add Card button** — dashed border ghost button at bottom; clicking shows inline add form
- **Inline add form** — text input (Enter to confirm, Escape to cancel) + color dot picker (12 colors + None) + Add/Cancel buttons. Animated entrance

**Collapsed column → Pill** (`col-pill`):
- 38px wide, vertical writing mode for title
- Shows card count badge
- Small `▶` expand indicator
- Click to expand
- `pillIn` entrance animation

**Column entrance animation** — `anim-col-new` for 0.24s on creation.

---

### Cards

Each card is `border-left: 4px solid {color}`, `border-radius: 10px`, `cursor: grab`.

**Card content**:
- **Title** — 12px, font-weight 600
- **Progress bar** — only shown if card has checklist items. Shows `{done}/{total}` label row + 3px colored progress bar
- **Meta row** (flex wrap, gap 4px):
  - **Priority badge** — Low (green), Med (amber), High (rose); colored background + text
  - **Tag chips** — gray rounded small badges, one per tag
  - **Due date badge** — colored by status:
    - `due-ok` (future > 2 days): blue
    - `due-soon` (≤ 2 days): amber with ⏰ icon
    - `due-over` (past): red with ⚠ icon; label text is "Today", "Tomorrow", "Yesterday", "{n}d overdue", or locale short date

**Card interactions**:
- **Hover** — lifts `translateY(-2px) scale(1.01)` + shadow
- **Click** — opens Card Detail Modal
- **Drag** — pointer-based drag; hold and move 5px+ to initiate. Creates floating ghost copy (`rotate(2deg) scale(1.04)`, opacity 0.88). Original card fades to 15% opacity. Blue horizontal line indicator (`drop-ind`) shows insertion point between cards or at column bottom. On drop, `moveCard()` is called and `anim-card-settle` plays on the landing card

**Card entrance animation** — `anim-card-new` on creation; `anim-card-fill` when added to a column at its WIP limit.

**Card background tint** — if card has a color, body background gets `hexToRgba(color, 0.13)` (dark) or `hexToRgba(color, 0.09)` (light).

**Momentum scroll** — dragging on empty board/group areas triggers physics-based momentum scroll (mouse-based swipe detection with velocity tracking).

---

## Page 3 — Card Detail Modal

Opens as a centered overlay (`position:fixed, inset:0, rgba(0,0,0,0.72) background`). The modal is resizable via a resize handle in the bottom-right corner. Default width 510px (stored in prefs). Max height 92vh, overflow-y auto.

In dark mode, if the card has a color, the modal background is tinted: `color-mix(in srgb, {color} 10%, #1a2538)`.

**Header row**:
- **5px accent bar** (left edge, full height of header row, card color or priority color)
- **Color circle picker** — 20px circle in card's color (or gray if none). Click toggles a dropdown (`cm-color-drop`) with 12 color dots + 1 "None" dot (strikethrough). Click a dot applies instantly and re-tints modal
- **Title input** — large (19px, bold), transparent background, blue underline border. Saves on blur
- **Due date badge** (if set) — color-coded, pulsing animation when overdue

**Description textarea** — full width, resizable, `min-height: 72px`. Saves on blur. Resize size is stored in `card.descH`.

**Controls row** (flex wrap):
- **🏷 Tags button** — toggles a small dropdown with a text input. Press Enter or click + to add a tag. Existing tags shown below as blue chip badges with × to remove
- **Priority** label + 3 buttons: Low / Med / High (active state gets tinted background matching priority color)
- **Column** label + dropdown (lists all non-archived columns; selecting moves the card immediately via `moveCMCard`)
- **Deadline** label + calendar button (shows selected date label or `🗓 Set date`). Clicking opens the custom calendar dropdown. When a date is set, a `✕` clear button appears beside it

**Custom Calendar Dropdown** (`cm-cal-drop`):
- 280px wide, slide-in animation
- Month/year header with `◂` and `▸` nav buttons
- 7-column grid: Su Mo Tu We Th Fr Sa headers, then day cells
- Day cell states: `today` (blue ring), `selected` (solid blue), `past` (dimmed), `other` (previous/next month, very dimmed)
- Footer: Clear button (removes date), Today button (sets today)

**Tags display area** — shows current tag chips with × remove; animated entrance for new chips

**Checklist section**:
- Section header: "Checklist" label + `{n}% done` percentage (green if 100%)
- Progress bar (4px height, transitions smoothly on change)
- Add item input + `+` button (prepends new item with slide-in animation)
- **Controls row** (when items exist):
  - `⇅ Sort` / `↺ Original` — sorts items incomplete-first, done-last; toggle restores original order
  - `☑ Check all` / `☐ Uncheck all` — toggles all
  - `🗑 Clear done` — removes all completed items (shown only when items are checked)
- **Checklist item row** (per item):
  - Drag handle (`⠿`) — visible on row hover; HTML5 drag to reorder
  - **Checkbox** — animated (scale + green fill on check, `✓` tick mark)
  - **Item text** — `overflow: hidden, text-overflow: ellipsis`; hover shows tooltip if truncated; double-click enters inline edit (borderless input, save on blur/Enter, cancel on Escape); strikethrough + muted when done
  - **× remove button** — faded, becomes visible on row hover

**Bottom action row**:
- Left: `🗑 Delete` (danger), `📋 Duplicate` (amber), `📦 Archive` (ghost)
- Right: `Close` (ghost), `Save` (primary)

**Unsaved changes banner** — appears at top of modal when close is attempted with dirty checklist (toggled/removed but not saved). Three options: "💾 Save & Close", "🗑 Discard & Close", "← Go Back"

**Unsaved draft item banner** — appears when close is attempted with text in the "Add checklist item" input. Three options: "➕ Add & Close", "🗑 Discard & Close", "← Go Back"

---

## Page 4 — Theme Panel

A floating panel (`position: fixed`) anchored below the `🎨 Theme` button. Rendered as a 330px wide card with a 4-column grid of theme swatches.

**Sections**:
1. `Dark` — 8 themes
2. `Light · Apple-Inspired` — 6 themes

**Each swatch** (`tswatch`):
- Shows a top ribbon (topbar color) + bottom fill (background color) + centered emoji
- Name below in small text
- Hover: scale 1.08 + shadow; active swatch: blue border ring
- **Hover preview** — on `mouseenter`, immediately applies the theme to the page (no save); on `mouseleave` with 80ms delay, reverts to current theme
- **Click** — permanently applies theme, saves to `S.prefs.theme` and localStorage, closes panel

**14 themes**:
| Name | Emoji | Type |
|---|---|---|
| Midnight | 🌑 | Dark |
| Obsidian | ⬛ | Dark |
| Graphite | 🪨 | Dark |
| Navy | ⚓ | Dark |
| Twilight | 🌆 | Dark |
| Aurora | 🌌 | Dark |
| Forest | 🌲 | Dark |
| Ember | 🔥 | Dark |
| Pearl | 🤍 | Light |
| Azure | 💙 | Light |
| Blossom | 🌸 | Light |
| Meadow | 🌿 | Light |
| Lavender | 💜 | Light |
| Sunset | 🌅 | Light |

Click outside the panel closes it and cancels any preview.

---

## Page 5 — Cloud Sync Modal (☁ Sync)

Opened by the `☁ Sync` topbar button. Standard modal overlay.

**When Firebase is not configured**:
- Static info panel explaining how to add Firebase config to the script block
- Close button only

**When Firebase is configured**:

**Status block** (at top):
- If syncing: green tinted panel showing active board key
- If local: gray panel saying "Running in local-only mode"

**Board key list** (max height 240px, scrollable):
Each saved key entry shows:
- **Label** + status indicator (`● ACTIVE` in green or `● inactive` in muted)
- **Key string** (monospace, small, truncated)
- **Action buttons**: Switch (hidden if already active), Copy, Email, Rename, Remove (danger)
  - Switch: calls `connectToKey()`
  - Copy: copies key to clipboard with notification
  - Email: opens `mailto:` with key in body + copies to clipboard
  - Rename: `prompt()` dialog, updates label in local key list and in `S.prefs.boardName`
  - Remove: removes from local list; if active, disconnects

**Bottom controls**:
- `Join existing board` label + text input (`placeholder="Paste board key…"`) + `Join` button
- `+ New Board` button (green) — generates a 10-byte hex key formatted as `xxxx-xxxx-xxxx-xxxx-xx`, creates entry in local key list, connects

**Footer**:
- `Disconnect` button (danger, only when connected) — with confirmation dialog
- `Close` button

---

## Page 6 — Backup & Restore Modal (💾 Backups)

**Sections from top to bottom**:

### Undo/Redo Status Bar
- Shows `↩ {n} undo step(s) in memory` and optionally `↪ {n} redo step(s)`
- Quick `↩ Undo` button (shown if stack non-empty)

### Auto-save Banner
- Green tinted: "✅ Auto-save is ON — every change saves instantly to local storage [and cloud]"

### Auto Snapshots (up to 15)
Automatically created by the system at:
- **Session start** — when board was modified since last auto-snap
- **Idle checkpoint** — 20 minutes after last user action (if changes exist)
- **Before delete** — before any destructive action (delete group, delete column, restore, import)

Each auto-snapshot row shows:
- Name (non-editable)
- Timestamp + card count + **trigger badge** (color-coded pill):
  - `Session start` — blue
  - `Idle checkpoint` — purple
  - `Before delete` — red
  - `Auto` — gray
- Restore button (blue tint) + Delete button (red tint)

### Manual Snapshot
- Name input + `📌 Save` button
- Keyboard shortcut `Ctrl/Cmd+S` creates a "Quick Save {time}" snapshot instantly

### Manual Snapshot List (up to 25)
Each row:
- **Name** — double-click to rename inline (borderless input, save on blur/Enter)
- Timestamp + card count
- Restore (blue tint) + Export (gray) + Delete (red) buttons

### Transfer Board
- `⬇ Export Board` — downloads `S` as `.json` file
- `⬆ Import Board` — file picker (`.json`), replaces board after confirm dialog, creates auto-snapshot before import

---

## Page 7 — Archive Modal (📦 Archive)

Full-width modal (up to 960px). Title: "📦 Archive".

**Legend bar** (when archive is non-empty):
- Explains `🗄 Archived` badge (entire column archived) vs columns without badge (individually archived cards from active column)

**Column panels** (horizontal scrollable row):
Panels sorted by most-recently-archived first.

**Fully archived column panel** (column was archived as a whole):
- Header: color dot + column name + `🗄 Archived` amber badge + ↩ Restore button + × Permanently Delete button
- Body: lists all cards with restore/delete buttons

**Partially archived column panel** (active column with some archived cards):
- Header: color dot + column name + card count badge (no archive badge)
- Body: lists only archived cards from this column

**Each card in archive**:
- Standard card visual (title, priority badge, tags, due badge, progress bar)
- `↩ Restore` button (ghost) — removes `archived` flag, card returns to its column
- `🗑` delete button (danger) — permanently deletes (with confirm)

**Empty state** — 📭 icon + "Archive is empty" + description text

---

## Page 8 — Add Group Modal

Small 360px modal.

- Title: "Add Group"
- Label: "Group Name"
- Input — `placeholder="e.g. 🚀 Work, 🏠 Personal…"`, auto-focused, Enter submits, Escape cancels
- Buttons: Cancel (ghost) + Create Group (primary)

---

## Page 9 — Add Column Modal

380px modal.

- Title: "Add Column"
- Label: "Column Name" + input (`placeholder="e.g. In Review, Blocked…"`)
- Label: "Column Color" + row of 9 color dots (PAL palette, dot picks selected, active dot has white border + scale 1.18)
- Label: "Assign to Group" + `<select>` with all groups + "— Ungrouped —" option
- Buttons: Cancel (ghost) + Create Column (primary)

---

## Data Model

```js
S = {
  prefs: {
    theme: 'midnight',      // theme id string
    modalW: 510,            // card modal width
    modalH: null,           // card modal height (null = auto)
    boardName: 'My Board'   // synced board display name
  },
  lastModified: 0,          // unix timestamp, updated on every real save
  groups: [
    {
      id: 'g1',             // uid
      title: '🚀 Work',
      pi: 0,                // palette index (0–8)
      col: false,           // true = group is collapsed
      limit: 0              // visible cards per column (0 = unlimited)
    }
  ],
  cols: [
    {
      id: 'c1',
      gid: 'g1',            // null if ungrouped
      title: '📋 To Do',
      pi: 0,                // palette index
      wip: 0,               // not used at column level in current impl
      collapsed: false,
      sort: false,          // true = sort cards by priority desc
      archived: false,      // true = whole column archived
      archivedAt: null,
      cards: [
        {
          id: 'k1',
          title: 'Card title',
          pri: 0,           // 0=Low, 1=Med, 2=High
          color: null,      // hex string or null
          done: [],         // array of completed checklist item strings
          items: [],        // ordered array of checklist item strings
          tags: [],         // array of tag strings
          desc: '',         // markdown-ish description text
          descH: null,      // user-resized textarea height in px
          due: '',          // ISO date string 'YYYY-MM-DD' or ''
          archived: false,
          archivedAt: null,
          archivedFromCol: null
        }
      ]
    }
  ]
}
```

---

## Color Palette (PAL — 9 colors)

| Index | Name | Accent (a) | Header bg dark (h) | Body bg dark (b) |
|---|---|---|---|---|
| 0 | Blue | #2563eb | #1a3a6e | #0c1f3d |
| 1 | Violet | #7c3aed | #3b1580 | #1e0a45 |
| 2 | Rose | #e11d48 | #7a1030 | #3d0818 |
| 3 | Emerald | #059669 | #065f3e | #022c1c |
| 4 | Amber | #d97706 | #78420a | #3d2005 |
| 5 | Cyan | #0891b2 | #0a5570 | #052838 |
| 6 | Fuchsia | #c026d3 | #6b0f6b | #360838 |
| 7 | Orange | #ea580c | #7c2d00 | #3d1500 |
| 8 | Indigo | #4f46e5 | #283895 | #131c50 |

---

## Card Color Options (12 + None)

None (transparent), Slate #64748b, Red #ef4444, Orange #f97316, Amber #f59e0b, Lime #84cc16, Emerald #10b981, Cyan #06b6d4, Blue #3b82f6, Violet #8b5cf6, Fuchsia #d946ef, Rose #f43f5e

---

## Priority Colors

| Level | Text color | Background |
|---|---|---|
| Low | #10b981 | rgba(16,185,129,.15) |
| Med | #f59e0b | rgba(245,158,11,.15) |
| High | #e11d48 | rgba(225,29,72,.15) |

---

## CSS Animations

| Name | Usage |
|---|---|
| `cardNew` | Card entrance (0.22s, spring cubic-bezier) |
| `cardSettle` | Card landing after drag drop (0.28s) |
| `colNew` | Column entrance (0.24s) |
| `pillIn` | Collapsed pill entrance (0.22s) |
| `groupIn` | Group entrance (0.22s) |
| `bannerIn` | Warning banner entrance (0.18–0.2s) |
| `cardFill` | Card added to a WIP-limited column (0.34s) |
| `btnPop` | Button press feedback (0.15s, scale 1→0.92→1) |
| `deadlinePulse` | Overdue card badge (2s ease-in-out infinite, red glow) |
| `modalIn` | Theme panel entrance (0.2s) |

---

## Keyboard Shortcuts

| Shortcut | Action |
|---|---|
| `Escape` | Close modal / cancel active drag |
| `Ctrl/Cmd + S` | Quick save manual snapshot |
| `Ctrl/Cmd + Z` | Undo (not in text inputs) |
| `Ctrl/Cmd + Y` or `Ctrl/Cmd + Shift + Z` | Redo (not in text inputs) |
| `Enter` in modal inputs | Submit form |
| `Escape` in modal inputs | Cancel / close |

---

## Undo / Redo System

- Full board state (`S`) is serialized to JSON and pushed onto `_undoStack` before every mutation
- Max 50 entries per stack
- Redo stack cleared on any new action
- Last 10 undo entries mirrored to `sessionStorage` for crash/F5 recovery
- `_undoInProgress` flag prevents re-entrancy during undo/redo applies
- Auto-snapshots (Firebase + localStorage) are separate from in-memory undo and survive page reloads

---

## Firebase Sync System

### Connection flow
1. On load, check `localStorage` for `fb4_active_key`; if present, call `connectToKey()`
2. `connectToKey()` tears down existing listeners, sets up `boards/{key}/state` value listener
3. On first connection: if remote is null, push local up; if remote exists, compare `lastModified` timestamps
4. Conflict resolution: **last-write-wins** via `lastModified`. Local wins only if structurally safe
5. **Safety gate** (`_isSafeWrite()`): blocks writes that would:
   - Wipe all groups when baseline had groups
   - Wipe all columns when baseline had columns
   - Drop card count to 0 from a meaningful board
   - Lose >75% of cards (>5 cards)
   - Introduce completely foreign column IDs (<20% match)
6. Auto-retry on failure: exponential backoff 5s → 15s → 30s → 1m → 5m
7. Reconnect on `window.online` event and `visibilitychange` (tab regains focus)
8. Debounced writes: 600ms after last save, batches rapid changes into one Firebase write

### Auto-snapshots in cloud
- Stored at `boards/{key}/autoSnaps` (separate from board state)
- Bidirectional merge: local-only snaps pushed up; remote-only snaps pulled down
- Capped at 15 entries after merge
- Debounced 2s before writing to Firebase

### Sync status transitions
`local` → `syncing` → `synced` | `error` | `offline` → auto-retry → `syncing` → `synced`

---

## Persistence

All data is saved to `localStorage`:
- `fb4` — serialized board state `S`
- `fb4_baks` — manual snapshots array (max 25)
- `fb4_auto` — auto-snapshots array (max 15)
- `fb4_keys` — saved board keys array `[{key, label, lastUsed}]`
- `fb4_active_key` — currently connected board key string

Storage usage is monitored; warn the user when approaching 4MB via a notification toast.

---

## Notification Toast

A fixed `bottom: 22px, right: 22px` toast (`#notif`). Slides in from below, auto-hides after ~2.6s. Used for: undo/redo confirmations, sync state changes, save confirmations, storage warnings, clipboard operations.

---

## Drag and Drop Details

**Card drag** (pointer events, not HTML5):
1. `pointerdown` on card body initiates a pending drag (stores start position)
2. `pointermove` past 5px threshold activates drag: clones card as ghost (fixed position), original fades to 15%
3. Ghost tracks pointer offset from card origin
4. On each move: `document.elementFromPoint` (ghost hidden temporarily) determines drop target
5. Drop on card: horizontal line indicator inserted above/below based on pointer Y vs card center
6. Drop on empty column body: highlights body with blue border
7. `pointerup`: removes ghost, calls `moveCard()` or `moveColumn()`, plays `anim-card-settle`

**Column drag**:
- 300ms hold delay on column header before ghost is created (prevents accidental drags on click)
- Vertical blue line indicator appears between columns in the target group row
- Dropping reassigns `col.gid` and reorders `S.cols`

**Momentum scroll**:
- While dragging on board/group row backgrounds, tracks velocity
- On pointer up, applies decelerated momentum animation via `requestAnimationFrame`
- Separate X (group row) and Y (board) axes

---

## Light Theme Adjustments

When a light theme is active (`isLight()` returns true):
- Column header backgrounds use `hexToRgba(accent, 0.22)` instead of `PAL[pi].h`
- Column body backgrounds use `hexToRgba(accent, 0.07)` instead of `PAL[pi].b`
- Card tint: `hexToRgba(color, 0.09)` instead of 0.13
- Color pickers and accent texts use the accent `.a` color directly
- Date inputs use `color-scheme: light`
- Modal background stays white (`var(--modal)`)

---

## Default Demo State

On first load (no localStorage), seed the board with:

**Groups**: `🚀 Work` (pi=0), `🏠 Personal` (pi=3)

**Work columns**:
- `📋 To Do` (3 cards: "Read design brief", "Set up dev environment", "Write project proposal")
- `🔄 In Progress` (wip=3, 2 cards: "Build FlowBoard", "Weekly team sync")
- `👀 Review` (wip=2, 1 card: "Homepage redesign")
- `✅ Done` (collapsed, 1 card: "Project kickoff")

**Personal columns**:
- `📝 Todo` (2 cards: "Grocery run", "Book dentist appointment")
- `✅ Done` (collapsed, 1 card: "Gym session")

Cards have sample tags, priorities, colors, checklist items, and some pre-completed checklist items.

---

## Implementation Notes for the Engineer

1. **No build tooling** — everything in one HTML file. CDN scripts for Firebase at the top of `<head>`
2. **CSS variables first** — define all color tokens as `:root` properties; override them per `[data-theme]` selector. Light themes need a separate group selector that flips to dark-on-light token values
3. **Render function is imperative** — `render()` rebuilds `#board` innerHTML from scratch on every state change. Preserve `board.scrollTop` before and restore after to prevent jarring jumps
4. **Modal system** — a single `#modal-root` div. `setModal(html, width, cb, closable)` injects a `.overlay` + `.modal` pair; `closeModal()` checks for dirty state before removing; `forceCloseModal()` bypasses dirty checks
5. **Inline rename pattern** — replace a `<span>` with a temporary `<input>`, save on blur/Enter, cancel on Escape. Used for group names, column names, and backup snapshot names
6. **Firebase safety gate** — implement `_isSafeWrite()` exactly as described. This prevents a common bug where an uninitialized S gets pushed to Firebase and wipes a real board
7. **Pointer drag** — use `pointerdown/pointermove/pointerup` on `document`. Do NOT use HTML5 `draggable` for cards (conflicts with touch and produces poor ghost visuals). The HTML5 drag API is used only for checklist item reordering within the modal
8. **Date handling** — dates stored as `'YYYY-MM-DD'` strings. Always construct dates with `new Date(due + 'T00:00:00')` to avoid UTC offset bugs. `dueClass()` uses today at midnight local time
9. **Session auto-snapshot** — call `_sessionStartSnap()` once after `load()` at startup. Only fires if `lastModified > lastAutoTs`
10. **Undo stack** — push `JSON.stringify(S)` BEFORE mutating S in every action function. Skip during `_undoInProgress` and `_skipFbWrite` (remote render) to prevent pollution
