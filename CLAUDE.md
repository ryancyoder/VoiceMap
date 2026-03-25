# CLAUDE.md - VoiceMap Codebase Guide

## Overview

VoiceMap is a single-file iOS/web PWA for capturing and organizing ideas through voice input. It uses the Claude API (`claude-haiku-4-5-20251001`) for intelligent card creation, merging, clustering, elaboration, and more. The entire application lives in one file: `index.html` (~6,200 lines of HTML, embedded CSS, and vanilla JavaScript).

## Repository Structure

```
VoiceMap/
├── index.html       # The entire application (HTML + CSS + JavaScript)
├── CLAUDE.md        # This file (technical reference for AI assistants)
└── USER_MANUAL.md   # End-user documentation
```

No build tools, package managers, frameworks, or external dependencies. Open `index.html` in a browser and it runs.

## Technology Stack

- **Language**: Vanilla JavaScript (ES6+, no strict mode declaration)
- **Styling**: Embedded CSS (no preprocessors)
- **AI**: Anthropic Claude API — `claude-haiku-4-5-20251001` at `https://api.anthropic.com/v1/messages`
- **Speech**: Web Speech API (browser native)
- **Storage**: IndexedDB (primary) + localStorage (backup)
- **Platform target**: iOS/iPadOS Safari standalone PWA; also works in Chrome/Chromium

## Development Workflow

### Running the App
Open `index.html` directly in a browser. No server, build step, or install required.

### Making Changes
Edit `index.html` directly. Reload the browser to see changes. All state persists automatically via IndexedDB/localStorage.

### Build Timestamp
`BUILD_TS` is a constant near the top of `<script>` that displays briefly on startup:
```javascript
const BUILD_TS = 'build:2026-03-26T11:00';
```
**Bump this on every deploy** so the user can confirm fresh PWA cache load. Format: `build:YYYY-MM-DDTHH:MM`.

### No Tests
There is no test framework. Manual browser testing is the only verification method.

### No CI/CD
No GitHub Actions, Dockerfiles, or build scripts exist.

## Code Organization

The JavaScript inside `index.html` is organized into sections separated by ASCII section headers:

```javascript
// ─── CONSTANTS ────────────────────────────────────────────────────
// ─── STATE ────────────────────────────────────────────────────────
// ─── STORAGE ──────────────────────────────────────────────────────
// ─── CLAUDE API ───────────────────────────────────────────────────
// ─── VOICE CAPTURE ────────────────────────────────────────────────
// ─── CARD MANAGEMENT ──────────────────────────────────────────────
// ─── CONSOLIDATE ──────────────────────────────────────────────────
// ─── MIND MAP ─────────────────────────────────────────────────────
// ─── UI / EVENT LISTENERS ─────────────────────────────────────────
```

The `DOMContentLoaded` handler at the end wires everything together.

## Key Constants

```javascript
const CLAUDE_MODEL  = 'claude-haiku-4-5-20251001';
const API_KEY_STORE = 'voicemap_api_key';   // localStorage key
const SESSION_STORE = 'voicemap_session';   // localStorage key
const DB_NAME       = 'VoiceMapDB';
const DB_VERSION    = 1;
const BUILD_TS      = 'build:YYYY-MM-DDTHH:MM'; // bump each deploy
```

When updating the Claude model version, change `CLAUDE_MODEL` only.

## Data Structures

### Session Object
```javascript
{
  session: string,        // Session name (e.g., "Untitled brainstorm")
  date: ISO string,       // Creation date
  nodes: Node[],          // Array of card nodes
  meta: {
    tool: "VoiceMap",
    version: "0.3",
    synced: ISO string    // Last GitHub Gist sync timestamp
  }
}
```

### Node (Card) Schema
```javascript
{
  id: string,                       // Unique identifier (makeId())
  label: string,                    // 2-5 word title, max 60 chars
  summary: string,                  // Description / notes (supports markdown)
  transcript: string,               // Original voice capture text
  parent_id: string | null,         // null = root node
  status: "untouched" | "in_progress" | "refined" | "locked" | "done",
  archived: boolean | null,
  pinned: boolean | null,
  deferred: boolean | null,         // Legacy defer system
  deferredUntil: ISO string | null, // Legacy defer date
  scheduled_date: string | null,    // "YYYY-MM-DD" for calendar view
  needs_scheduling: boolean | null, // true = in queue sidebar
  recurrence: "daily" | "weekly" | "biweekly" | "monthly" | null,
  recurrenceDayOfWeek: number | null, // 0–6 for weekly/biweekly
  type: "image" | null,             // Image cards only
  created: ISO string,
  last_modified: ISO string
}
```

## Storage Architecture

Dual-store strategy with automatic sync:

| Store | Key | Contents |
|-------|-----|----------|
| localStorage | `voicemap_session` | Full session JSON (text only) |
| localStorage | `voicemap_api_key` | Anthropic API key |
| IndexedDB (`sessions` store) | — | Full session JSON |
| IndexedDB (`images` store) | nodeId | Image data URLs |
| In-memory `imageCache` | nodeId | Decoded image data URLs |

**Load logic**: On startup, if IndexedDB has more nodes than localStorage → use IndexedDB; otherwise sync localStorage → IndexedDB.

Every write calls `saveSession()`, which saves to both localStorage and IndexedDB in parallel.

## Claude API Integration

### `callClaude(userMessage, systemPrompt, prefill, label)` → `string`
Makes a direct `fetch` to `https://api.anthropic.com/v1/messages`. Uses the prefill technique to force JSON-formatted responses:

```javascript
messages: [
  { role: "user",      content: userMessage },
  { role: "assistant", content: prefill }   // e.g., '{'
]
```

Returns the raw text content (callers parse JSON). All calls are logged to the prompt log.

### API Key Management
- User pastes Anthropic API key via modal on first launch
- Stored: `localStorage.getItem('voicemap_api_key')`
- Validated against `https://api.anthropic.com/v1/models`
- Never transmitted to any backend; all requests go directly to Anthropic

### AI Operations Reference

| Function | What Claude Does |
|----------|-----------------|
| `handleTranscript()` | Creates/completes a card from voice transcript |
| `mergeSelected()` | Merges multiple selected cards into one |
| `autoCluster()` | Groups cards at current level into topic clusters |
| `simplifyNode(id)` | Restructures a card's subtree (flatten, merge overlaps) |
| `refileNode(id)` | Suggests a new parent for a card |
| `startElaborate(id)` | Adds voice context to a card's summary |
| `openChatModal(id)` | Open-ended Q&A about the current card tree |
| Vision API (camera) | Detect sticky notes in photos, create cards |

## Views

VoiceMap has four main views switched via tab bar:

### 1. Cards (List View)
Default view. Shows cards at the current navigation level. Controlled by:
- `currentView` state variable: `'all' | 'pinned' | 'deferred' | 'orphaned'`
- `navStack` array for drill-down navigation
- Filter pills in `#cardFilterBar`
- `renderCards(animDir)` redraws the list

### 2. Mind Map
SVG-based visual tree. Key state:
- `mmVisible` — boolean, whether map is shown
- `mmSelectedId` — currently selected node id
- `mmFocusNodeId` — focus-mode root node id (null = show all)
- `mmCollapsed` — Set of collapsed node ids
- `mmTransform` — `{ x, y, scale }` for pan/zoom
- `mmMoveMode` — boolean, move mode active
- `mmMoveSourceId` / `mmMoveTargetId` — move mode state

Key functions:
- `renderMindMap()` — full re-render of SVG tree
- `selectMmNode(id, scroll)` — set selection + highlight
- `mmZoomToCard(id)` — pan/zoom to show a node
- `mmZoomToFit()` — fit all visible nodes
- `applyMmTransform()` — apply `mmTransform` to canvas CSS
- `mmArrowNav(dir)` — keyboard tree navigation
- `mmNodeOnScreen(id)` — perimeter check (pan only when node near edge)

### 3. Kanban Board
`#kbView`. Columns = children of current node. Cards = their children. Drag to reparent.

### 4. Week Calendar
`#calView`. 7-day grid + month strip + queue sidebar. Cards with `scheduled_date` appear in grid.

## Navigation

```javascript
navStack = [{ id: null, label: 'Home' }, ...]
```

- `navigateInto(nodeId, label)` — push to stack, renderCards('forward')
- `navigateBack()` — pop stack, renderCards('back')
- `currentParentId()` — returns current context's id (null = root)
- `childrenOf(id)` — `session.nodes.filter(n => n.parent_id === id)`

## Key UI Components

### Context Menu
`#nodeCtxMenu` — 13 items. Shown by `showNodeCtxMenu(nodeId, x, y)`.
Wired in `initNodeCtxMenu()`.

Items (id → function):
- `ctxPin` → `pinNode()`
- `ctxArchive` → `archiveNode()`
- `ctxDefer` → `showDeferModal()`
- `ctxSchedule` → `showScheduleModal()`
- `ctxQueue` → `queueNode()`
- `ctxEditTitle` → `startEditField(id, 'label')`
- `ctxEditDesc` → `startEditField(id, 'summary')`
- `ctxConsolidate` → `consolidateNode()`
- `ctxTaskList` → `taskListNode()`
- `ctxElaborate` → `startElaborate()`
- `ctxExplore` → `openChatModal()`
- `ctxRefile` → `refileNode()`
- `ctxDelete` → `deleteNode()`

### Edit Field Modal
`#editFieldModal` (`.defer-overlay`) — used for inline editing of title/summary, adding map child nodes, and elaborate mode. State managed by `pendingEdit`, `pendingMmAdd`.

### Consolidate Modal
`#consolidateModal` — shared by both Consolidate and Task List commands. State: `_consolidatePending = { nodeId }`. Preview textarea `#consolidatePreview` is editable before apply.

### Quick Find Modal
`#quickFindModal` — Cmd+K toggle. State:
```javascript
let _qfIdx = 0, _qfMatches = [], _qfOpenTime = 0, _qfCloseTime = 0;
```
- iOS ghost click guard: `_qfOpenTime` (ignore dismiss within 400ms of open)
- Map touchstart guard: `_qfCloseTime` (ignore touches within 400ms of close)
- Keypress capture fallback for iOS PWA focus issues

### Map Overlay Bars
All positioned in `#mmTopBar`:
- `#mmFocusBar` — shows when `mmFocusNodeId` set
- `#mmMoveBar` — shows when `mmMoveMode` active
- `#mmFilterBar` — deferred/archived toggle chips

## Gestures & Touch Architecture

### Card List
- **Swipe**: `initSwipeHandlers()` — horizontal swipe reveals action buttons. State: `swipeState`, `openSwipeNodeId`
- **Long press**: `swipeLongPressTimer` at 500ms → `showNodeCtxMenu()`. Bails if touch starts on `.drag-handle`
- **Drag reorder**: `initDragHandlers()` — grab `.drag-handle` (⠿ icon), move to activate immediately. State: `dragState`. Drop zones: before (top 30%), inside (middle 40%), after (bottom 30%)

### Mind Map
- **Touch**: Pan (1-finger drag), pinch zoom, single-tap select, double-tap focus, long-press context menu, long-press timer: `mmLongPressTimer` at 600ms
- **Mouse**: Drag to pan, wheel to zoom, click to select, dblclick to focus
- **Move mode touch**: Single tap sets move target (guards against source/descendants)

## Keyboard Shortcuts

All implemented via two listeners:

### 1. Document capture-phase listener (all views)
```javascript
document.addEventListener('keydown', handler, { capture: true });
```
- **Cmd+K** — Quick Find (global, any view)
- **Opt+Arrows** — Pan map canvas (map only)
- **Cmd+Up/Down** — Zoom map (map only)
- **Cmd+Left/Right** — Collapse/Expand node (map only)

### 2. `#mmCanvasWrap` keydown listener (map focused)
- **Arrow keys** — `mmArrowNav()` (normal) or `mmMoveArrowNav()` (move mode)
- **Escape** — cancel move mode
- **Enter** — execute move
- **Cmd+N** — add child node
- **Cmd+K** — Quick Find
- **Cmd+M** — toggle move mode

## Consolidate & Task List

Both functions reuse the same `#consolidateModal` and `_consolidatePending` state.

### Consolidate
```javascript
function _consolidateBuildMd(nodeId, depth) // recursive heading builder
function consolidateNode(nodeId)             // entry: builds md, shows modal
```
Output: `## Child\n\nbody\n\n### Grandchild\n\nbody`
Heading level = depth + 1 (capped at 6). Archived nodes excluded.

### Task List
```javascript
function _taskListBuildMd(nodeId, depth)    // recursive checkbox builder
function taskListNode(nodeId)               // entry: builds md, shows modal
```
Output: `- [ ] Title\n  - [ ] Subtitle`
Indentation = 2 spaces per depth level. Only labels used, descriptions ignored.

### Apply (shared)
```javascript
function _consolidateApply(mode) // mode: 'replace' | 'append'
```
1. Reads markdown from `#consolidatePreview` textarea (user may have edited it)
2. Replaces or appends to `node.summary`
3. Calls `getDescendants(nodeId)` and removes all descendants from `session.nodes`
4. `saveSession()` + `renderCards()`
5. Toast shows count of removed nodes

## Data Repair

Runs on every `initApp()`:

```javascript
function repairCircularRefs()  // detects parent_id chains that loop
function repairOrphans()       // detects parent_id pointing to deleted node
function repairData()          // calls both, saves + toasts if fixed
```

Both use a `seen` Set to detect cycles. Critical: `_qfPath()` and any other parent-chain traversal must also use a `seen` Set to guard against circular refs.

## GitHub Gist Sync

State: `getGithubToken()` / `getGistId()` / `setGistId()` (localStorage keys).

```javascript
function scheduleSyncToGist()    // debounced 3s, called on every save
function syncFromGist()          // polling every 30s
function pushToGist(data)        // POST/PATCH to api.github.com
function pullFromGist()          // GET from api.github.com
```

Merge logic: compare `meta.synced` timestamps; newest wins. Images stored as base64 in the Gist payload.

## iOS / PWA Specifics

Critical patterns for Safari PWA:

1. **Ghost clicks**: iOS fires synthetic click ~300ms after tap. Guard with timestamps:
   - `_qfOpenTime` — modal opened, ignore dismiss clicks for 400ms
   - `_qfCloseTime` — modal closed, ignore map touches for 400ms

2. **Focus blocking**: `input.focus()` is blocked after `metaKey` events. Workaround: keypress capture listener routes chars to input manually.

3. **System shortcut conflicts**: Some Cmd+key combos are reserved (Cmd+O freezes). Use Cmd+K for Quick Find.

4. **Capture-phase listeners**: `{ capture: true }` on document is required to intercept Cmd+Arrow and Opt+Arrow before Safari's built-in scroll handlers.

5. **Cmd+M in capture listener**: Does NOT use stopPropagation on map shortcuts — that causes WebKit pipeline hang.

6. **Safe areas**: All layout uses `env(safe-area-inset-*)` for notch/home indicator.

7. **viewport-fit=cover**: Required for edge-to-edge on modern iPhones.

## Naming Conventions

- **Variables**: `camelCase`
- **Constants**: `SCREAMING_SNAKE_CASE`
- **DOM IDs**: `camelCase` (e.g., `recordBtn`, `chatModal`)
- **Functions**: verb-first camelCase (`showMindMap`, `renderCards`, `deleteNode`)
- **Data attributes**: `kebab-case` (e.g., `data-node-id`, `data-swipe-delete-id`)
- **CSS classes**: `kebab-case`
- **Private/helper functions**: `_prefixedCamelCase` (e.g., `_qfRender`, `_consolidateBuildMd`)

## Key Functions Reference

### Initialization
- `initApp()` — main async init; loads data, repairs data, sets up views
- `initRecognition()` — Web Speech API setup
- `initSwipeHandlers()` — card list touch/swipe
- `initDragHandlers()` — card drag-to-reorder (grab `.drag-handle` only)
- `initNodeCtxMenu()` — wire context menu buttons
- `initConsolidateModal()` — wire consolidate/task-list modal buttons
- `initQuickFind()` — wire Quick Find modal + capture listeners

### Card Rendering
- `renderCards(animDir)` — redraws card list; respects `currentView` and `navStack`
- `renderMindMap()` — full SVG redraw; sets selection via `mmSelectedId` before calling

### Persistence
- `saveSession()` — dual-save localStorage + IndexedDB (always await)
- `loadSessionFromDB()` — async IndexedDB read
- `downloadJSON()` / `importJSON()` — export/import session file

### Node Lifecycle
- `pinNode(id)` — toggle pinned
- `archiveNode(id)` — toggle archived
- `deferNode(id, days)` — set deferred_until
- `deleteNode(id)` — confirm + remove node + descendants
- `getDescendants(id)` — returns Set of all descendant ids (recursive)
- `childrenOf(id)` — direct children array

### Mind Map
- `showMindMap()` / `hideMindMap()` — toggle map visibility
- `mmFocusOnNode(id)` — enter focus mode on a node
- `mmExitFocus()` — exit focus mode
- `startMmMove()` / `cancelMmMove()` / `executeMmMove()` — move mode
- `mmMoveArrowNav(dir)` — navigate move target with arrow keys
- `_mmIsDescendantOrSelf(nodeId, ancestorId)` — cycle guard for move

### Quick Find
- `showQuickFind()` / `hideQuickFind()` — toggle modal
- `_qfRender(query)` — filter + display results
- `_qfActivate(idx)` — navigate to selected result
- `_qfPath(node)` — builds ancestor breadcrumb (has cycle guard)

## Common Tasks

### Add a new context menu action
1. Add `<button class="node-ctx-item" id="ctxMyAction">…</button>` to `#nodeCtxMenu` in HTML
2. Wire in `initNodeCtxMenu()`: `document.getElementById('ctxMyAction').addEventListener('click', () => { closeNodeCtxMenu(); myFunction(getCtxNode()); });`
3. Write `myFunction(nodeId)`

### Add a new AI-powered card operation
1. Write `async function myOperation(nodeId)`
2. Call `callClaude(userMessage, systemPrompt, prefill, 'My Operation Label')` and parse JSON
3. Update `session.nodes` and call `saveSession()` then `renderCards()`

### Change the Claude model
Update `CLAUDE_MODEL` constant only. Do not hardcode model strings elsewhere.

### Add a new node field
1. Add to node creation logic (search for `makeId()` calls)
2. `saveSession()` / loading logic rarely need changes (JSON is schema-flexible)
3. Update `renderCards()` if field affects display

### Modify mind map layout
Edit `renderMindMap()`. It builds SVG/div elements directly. Node positions use simple tree-layout math. Node elements use `data-mmid` attribute.

### Add a new view filter pill
1. Add `<button class="filter-pill" data-view="myview">…</button>` to `#cardFilterBar`
2. Add `else if (currentView === 'myview')` case in `renderCards()`
3. Update comment for `currentView` declaration

## Important Notes for AI Assistants

1. **Single file**: All changes go in `index.html`. Do not create `.js` or `.css` files.
2. **No build step**: Never suggest webpack, vite, npm, or other build tooling.
3. **Read before editing**: The file is 6,000+ lines. Always read the relevant section first.
4. **Preserve section headers**: Keep the ASCII `// ─── SECTION ───` headers.
5. **Bump BUILD_TS**: After every set of changes, update `BUILD_TS` to confirm PWA cache refresh.
6. **API key safety**: Never log, transmit, or expose `voicemap_api_key`.
7. **IndexedDB is async**: All IndexedDB operations return Promises; always `await` them.
8. **iOS quirks**: Test PWA behavior separately from browser. Ghost clicks, focus blocking, and system shortcuts behave differently in standalone mode.
9. **Cycle guards**: Any function that walks `parent_id` chains MUST use a `seen` Set to guard against circular references.
10. **renderMindMap timing**: Set `mmSelectedId` BEFORE calling `renderMindMap()` so the selection is baked into the HTML. Use `selectMmNode()` in a `setTimeout(..., 50)` after for zoom/scroll.
11. **Consolidate modal is shared**: Both `consolidateNode()` and `taskListNode()` use `#consolidateModal` and `_consolidatePending`. Don't break this sharing when adding new operations.

## Git Branch Convention

Development branches follow: `claude/<description>-<id>` (e.g., `claude/add-claude-documentation-QCvRE`).

All commits should be pushed to both `main` and the active feature branch.
