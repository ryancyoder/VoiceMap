# CLAUDE.md - VoiceMap Codebase Guide

## Overview

VoiceMap is a single-file iOS/web application for capturing and organizing ideas through voice input. It uses the Claude API (claude-haiku-4-5-20251001) for intelligent card creation, merging, clustering, and elaboration. The entire application lives in one file: `index.html` (3,637+ lines of HTML, embedded CSS, and vanilla JavaScript).

## Repository Structure

```
VoiceMap/
├── index.html       # The entire application (HTML + CSS + JavaScript)
└── CLAUDE.md        # This file
```

No build tools, package managers, frameworks, or external dependencies. Open `index.html` in a browser and it runs.

## Technology Stack

- **Language**: Vanilla JavaScript (ES6+, strict mode)
- **Styling**: Embedded CSS (no preprocessors)
- **AI**: Anthropic Claude API — `claude-haiku-4-5-20251001` at `https://api.anthropic.com/v1/messages`
- **Speech**: Web Speech API (browser native)
- **Storage**: IndexedDB (primary) + localStorage (backup)
- **Platform target**: iOS/iPadOS Safari, also works in Chrome/Chromium

## Development Workflow

### Running the App
Open `index.html` directly in a browser. No server, build step, or install required.

### Making Changes
Edit `index.html` directly. Reload the browser to see changes. All state persists automatically via IndexedDB/localStorage.

### No Tests
There is no test framework. Manual browser testing is the only verification method.

### No CI/CD
No GitHub Actions, Dockerfiles, or build scripts exist.

## Code Organization

The JavaScript inside `index.html` is organized into sections separated by ASCII headers:

```javascript
// ─── CONSTANTS ────────────────────────────────────────────────────
// ─── STATE ────────────────────────────────────────────────────────
// ─── STORAGE ──────────────────────────────────────────────────────
// ─── CLAUDE API ───────────────────────────────────────────────────
// ─── VOICE CAPTURE ────────────────────────────────────────────────
// ─── CARD MANAGEMENT ──────────────────────────────────────────────
// ─── MIND MAP ─────────────────────────────────────────────────────
// ─── UI / EVENT LISTENERS ─────────────────────────────────────────
```

The `DOMContentLoaded` handler at the end wires everything together.

## Key Constants

```javascript
const CLAUDE_MODEL = 'claude-haiku-4-5-20251001';
const API_KEY_STORE = 'voicemap_api_key';   // localStorage key for API key
const SESSION_STORE = 'voicemap_session';   // localStorage key for session data
const DB_NAME = 'VoiceMapDB';
const DB_VERSION = 1;
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
    version: "0.3"
  }
}
```

### Node (Card) Schema
```javascript
{
  id: string,                       // Unique identifier
  label: string,                    // 2-5 word title, max 60 chars
  summary: string,                  // Optional 1-2 sentence summary
  transcript: string,               // Full voice capture text
  parent_id: string | null,         // null for root nodes
  status: "untouched" | "in_progress" | "refined" | "locked" | "done",
  archived: boolean | null,
  deferred_until: ISO string | null,
  recurrence: "daily" | "weekly" | "biweekly" | "monthly" | null,
  recurrence_day: number | null,    // 0–6 for weekly recurrence
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

**Upgrade logic**: On load, if IndexedDB has more nodes than localStorage, use IndexedDB; otherwise sync localStorage → IndexedDB.

Every write calls `saveSession()`, which saves to both localStorage and IndexedDB in parallel.

## Claude API Integration

### `callClaude(systemPrompt, userMessage, prefill)` → `string`
Makes a direct `fetch` to `https://api.anthropic.com/v1/messages`. Uses the prefill technique to force JSON-formatted responses:

```javascript
messages: [
  { role: "user", content: userMessage },
  { role: "assistant", content: prefill }   // e.g., '{'
]
```

The function returns the raw text content (caller parses JSON).

### API Key Management
- User pastes their Anthropic API key via a modal dialog on first launch
- Key stored in `localStorage.getItem('voicemap_api_key')`
- Validated against `https://api.anthropic.com/v1/models`
- Never transmitted to any backend; all requests go directly to Anthropic

### AI Operations
| Function | What Claude Does |
|----------|-----------------|
| `handleTranscript()` | Creates a new card from voice transcript |
| `mergeSelected()` | Merges multiple selected cards into one |
| `autoCluster()` | Groups cards into logical topic clusters |
| `simplifyNode()` | Restructures a card's subtree |
| `refileNode()` | Suggests a new parent for a card |
| `startElaborate()` | Generates sub-cards to expand a card |
| Chat modal | Open-ended Q&A about the current card tree |

## Naming Conventions

- **Variables**: `camelCase`
- **Constants**: `SCREAMING_SNAKE_CASE`
- **DOM IDs**: `camelCase` (e.g., `recordBtn`, `chatModal`)
- **Functions**: verb-first camelCase (`showMindMap`, `renderCards`, `deleteNode`)
- **Data attributes**: `kebab-case` (e.g., `data-node-id`, `data-swipe-delete-id`)
- **CSS classes**: `kebab-case`

## Key Functions Reference

### Initialization
- `initApp()` — Main async init; loads data, shows API key modal if needed
- `initRecognition()` — Sets up Web Speech API
- `initSwipeHandlers()` — Touch swipe for card actions
- `initDragHandlers()` — Drag-and-drop for mind map nodes

### Navigation
- `navigateInto(nodeId)` — Drill into a node's children
- `navigateBack()` — Pop navigation stack

### Card Rendering
- `renderCards()` — Redraws the card list for current nav context
- `renderMindMap()` — Draws SVG mind map

### Persistence
- `saveSession()` — Dual-save to localStorage + IndexedDB
- `loadSessionFromDB()` — Async IndexedDB read
- `downloadJSON()` / `importJSON()` — Export/import session file

### Status / Lifecycle
- `pinNode(id)` — Toggle pinned status
- `deferNode(id, date)` — Set deferred_until
- `archiveNode(id)` — Toggle archived
- `deleteNode(id)` — Remove node (with confirmation)

## Mobile / iOS Specifics

- Viewport: `viewport-fit=cover` for notch/home-indicator safety areas
- CSS `safe-area-inset-*` used throughout for layout
- `apple-mobile-web-app-capable` meta tag for standalone mode
- Back Tap shortcut: `?record=1` URL param starts recording automatically
- Touch events preferred over click events for responsiveness
- `-webkit-tap-highlight-color: transparent` and controlled text selection

## UI Patterns

- **Swipe left** on a card → delete/complete/pin (swipe action buttons reveal)
- **Long press** on a card → context menu
- **Defer modal** — date + recurrence picker
- **Mind map** — SVG with pan/zoom; auto-shows on iPad landscape (≥900px)
- **Toast notifications** — `showToast(message)` for brief status feedback
- **Status text** — `setStatus(message)` for persistent status line

## Common Tasks

### Add a new AI-powered card operation
1. Write a new function (e.g., `async function myOperation(nodeId)`)
2. Call `callClaude(systemPrompt, userContent, prefill)` and parse the JSON response
3. Update `currentData.nodes` and call `saveSession()` then `renderCards()`
4. Add a button/gesture in the relevant UI section of `index.html`

### Change the Claude model
Update `CLAUDE_MODEL` constant at the top of the `<script>` section.

### Add a new node field
1. Add the field to the node creation logic (search for where nodes are constructed)
2. Update `saveSession()` / loading logic if schema migration is needed
3. Update `renderCards()` if the field affects display

### Modify mind map layout
Edit `renderMindMap()` — it builds SVG elements directly; node positions are calculated with simple tree-layout math.

## Git Branch Convention

Development branches follow: `claude/<description>-<id>` (e.g., `claude/add-claude-documentation-QCvRE`).

## Important Notes for AI Assistants

1. **Single file**: All changes go in `index.html`. Do not create additional `.js` or `.css` files unless explicitly requested.
2. **No build step**: Never suggest adding webpack, vite, npm, or other build tooling unless the user specifically asks.
3. **Read before editing**: The file is 3,600+ lines. Always read the relevant section before modifying it.
4. **Preserve section headers**: Keep the ASCII `// ─── SECTION ───` headers when adding new code sections.
5. **Test manually**: There are no automated tests. Remind the user to test in a real browser.
6. **API key safety**: Never log, transmit, or expose the `voicemap_api_key` value.
7. **IndexedDB is async**: All IndexedDB operations return Promises; always `await` them.
8. **Model ID**: Use `CLAUDE_MODEL` constant; do not hardcode model strings elsewhere.
