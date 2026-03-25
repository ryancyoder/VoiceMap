# VoiceMap User Manual

VoiceMap is a personal idea-capture and organization tool built for iOS/iPadOS. Speak your thoughts, and the app turns them into structured cards you can organize, link, and explore — all powered by your own Anthropic API key.

---

## Getting Started

### First Launch
On first open, VoiceMap asks for your **Anthropic API key**. This is the key from your account at console.anthropic.com. The key is stored only on your device and sent directly to Anthropic — never to any other server.

Paste your key and tap **Test** to verify it, then **Save**.

### Installing as a PWA (iPhone/iPad)
1. Open `index.html` in Safari
2. Tap the Share button → **Add to Home Screen**
3. Launch from your home screen for full-screen standalone mode

---

## The Four Views

Tap the view tabs at the top right to switch between views.

| Tab | What it shows |
|-----|--------------|
| **Cards** | Scrollable list of cards at the current navigation level |
| **Map** | Visual mind map of the full card tree |
| **Board** | Kanban columns — each child of the current node is a column |
| **Week** | Weekly calendar grid for scheduled cards |

On iPad in landscape orientation (≥900px wide), the mind map opens automatically alongside the card list.

---

## Capturing Ideas

### Voice
Tap the red **⏺** record button and speak. VoiceMap transcribes your speech live. When you stop speaking (or tap the stop button), the AI processes your words and creates a new card — or completes an existing one if your phrasing matches something already there (e.g. "I finished the proposal draft").

### Keyboard
Tap **Aa Type instead** to switch to text input.
- **Enter** — add card directly (no AI)
- **Cmd+Enter** — add card with AI processing

### Back Tap Shortcut (iPhone)
Add a Back Tap shortcut in iOS Settings → Accessibility → Touch → Back Tap that opens the URL `voicemap://` or your saved URL with `?record=1` appended. VoiceMap will start recording automatically on launch.

### Photo / Camera
Tap the **📷** camera button to photograph sticky notes or a whiteboard. The AI detects individual sticky notes in the image and creates a separate card for each one.

---

## Cards

### Card Status
Each card has a colored status indicator (pip) on its left edge:

| Color | Status | Meaning |
|-------|--------|---------|
| Gray | Untouched | New, not yet acted on |
| Amber | In Progress | Actively being worked |
| Green | Refined | Polished and ready |
| Indigo | Locked | Frozen, don't change |
| Solid green | Done | Completed |

### Navigating Into a Card
Tap a card to drill into its children. The title bar shows where you are, and a **‹ Back** button appears. You can drill as deep as you like.

### Expanding a Card
Tap the **ⓘ** button on the right of any card to expand it and see the full description, any attached image, and action buttons.

---

## Gestures

### Swipe Right (on a card)
Reveals two action buttons:
- **✓ Done** — mark complete (or archive/reschedule)
- **📌 Pin** — pin or unpin the card

### Swipe Left (on a card)
Reveals two action buttons:
- **⏳ Defer** — push the card to a future date
- **🗑 Delete** — delete the card and all its children

### Long Press (on a card)
Opens the **context menu** with all available actions (see below).

### Drag Handle ⠿ (on a card)
Grab the ⠿ grip icon on the right edge of any card and drag to reorder or reparent:
- Drop **above** or **below** a card to reorder it
- Drop **on top of** a card to make it a child of that card

---

## Filter Pills

The filter bar under the view tabs lets you filter the card list:

| Filter | Shows |
|--------|-------|
| **All** | All cards at the current level |
| **📌 Pinned** | Cards you've pinned |
| **⏳ Deferred** | Cards deferred to a future date |
| **◎ Orphaned** | Root-level cards with no children (standalone notes) |

---

## Context Menu

Long-press any card to open the context menu:

| Action | What it does |
|--------|-------------|
| **📌 Pin / Unpin** | Marks the card for quick access in the Pinned filter |
| **📁 Archive / Unarchive** | Hides the card from normal view |
| **⏳ Defer** | Schedule the card for 1 day, 1 week, or 1 month from now |
| **📅 Schedule** | Pick a specific date on the calendar |
| **📋 Queue** | Add to the scheduling queue (appears in Week view sidebar) |
| **✎ Edit Title** | Edit the card's title directly |
| **📝 Edit Description** | Edit the card's description/notes |
| **⊞ Consolidate to Markdown** | Convert the child tree to a nested markdown document |
| **☑ Convert to Task List** | Convert child cards to a markdown checkbox list |
| **✨ Elaborate with AI** | Add more context to the card via voice or keyboard |
| **💬 Explore with AI** | Start an AI conversation about this card |
| **↗ Re-file** | Ask AI to suggest a better parent for this card |
| **✕ Delete** | Delete the card and all children (asks for confirmation) |

---

## Card Action Row

When a card is expanded (tap ⓘ), action buttons appear at the bottom:

| Button | What it does |
|--------|-------------|
| **↗ Re-file** | AI suggests a new location for this card in the tree |
| **⚡ Simplify** | AI restructures the children — merges overlaps, removes redundancy |
| **✏ Elaborate** | Speak or type to add more detail to the card's description |
| **💬 Explore** | AI chat conversation seeded with this card's content |
| **📤 Export Prompt** | Copies the card's full subtree as a formatted system prompt |

---

## AI Features

All AI features use the Claude API with your own key. They appear as a brief "Thinking…" status.

### Elaborate
Add more context to a card by speaking. Say "rename [new title]" to change the card's title instead.

### Re-file
AI reviews all other cards at the root level and suggests which one should be the parent of the selected card. You can confirm or cancel.

### Simplify
Restructures a card's children — merges overlapping ideas, removes redundancies, flattens excessive nesting. The AI keeps all distinct concepts but improves the structure.

### Auto-Cluster
Available from the toolbar (Wand icon ✦). Groups all cards at the current level into logical topic clusters, creating parent cards for each cluster.

### Merge
Select multiple cards (tap ⊞ Select, then choose cards) and tap **Merge** to combine them into one card using AI.

### Explore / Chat
Opens a full chat interface seeded with the selected card's content. Claude starts by asking an opening question. You can voice-input your responses. Save any assistant reply as a child card.

### Consolidate to Markdown
Converts the entire subtree of a card into a nested markdown document:
```
## Child Card Title
Child card description text

### Grandchild Card Title
Grandchild description text
```
A preview appears so you can choose to **Append** (add to existing description) or **Replace** it. After confirming, all child cards are deleted — the information now lives in the description.

### Convert to Task List
Converts the child cards into a markdown checkbox list:
```
- [ ] First child title
  - [ ] Grandchild title
  - [ ] Another grandchild
- [ ] Second child title
```
Only titles are used — descriptions are discarded. Same Append/Replace preview as Consolidate. Child cards are deleted after confirming.

---

## Mind Map

Switch to the **Map** tab to see a visual tree of all your cards.

### Navigating the Map

| Action | How |
|--------|-----|
| Pan | One or two-finger drag (touch), or drag with mouse |
| Zoom | Pinch (touch), scroll wheel, or Cmd+Up/Down (keyboard) |
| Select a node | Tap or click |
| Open right panel | Tap/click a node |
| Collapse/Expand | Tap the ▼/▶ button on a node |

### Keyboard Shortcuts (Map View)

| Shortcut | Action |
|----------|--------|
| **Arrow keys** | Navigate between nodes |
| **Cmd+Up / Cmd+Down** | Zoom in / Zoom out |
| **Cmd+Left** | Collapse selected node |
| **Cmd+Right** | Expand selected node |
| **Opt+Arrows** | Pan the canvas (80px per press) |
| **Cmd+N** | Add child to selected node |
| **Cmd+M** | Enter / exit Move mode |
| **Cmd+K** | Open Quick Find |

### Focus Mode
Double-tap a node (or tap **🔍 Focus on this card** in the right panel) to zoom into just that card and its descendants. A blue "Focus" bar appears at the top. Tap **✕ Exit** or press Escape to return to the full map.

### Move Mode (Cmd+M)
1. Select a node (red outline)
2. Press **Cmd+M** — a green "Move" bar appears
3. Navigate with arrow keys — a blue outline follows your selection as the destination
4. After 800ms pause, or press **Enter**, the selected node moves under the blue-outlined destination
5. Press **Escape** or tap **✕ Cancel** to abort

You can also tap directly on any node to set it as the destination.

### Map Filter Bar
The filter bar at the top of the map lets you hide **Deferred** and/or **Archived** nodes to reduce clutter.

---

## Quick Find

**Cmd+K** (or tap the 🔍 button) opens Quick Find from any view.

- Start typing to search all cards by title, description, or transcript
- Arrow keys to move through results
- **Enter** to navigate to the selected card
- In map view, activates Focus Mode on the card
- **Escape** to close

---

## Week View (Calendar)

The Week view shows your scheduled cards in a 7-day grid.

### Scheduling a Card
1. Long-press a card → **📅 Schedule** to pick a date
2. Or long-press → **📋 Queue** to add it to the sidebar queue
3. From the sidebar, drag the card to a day column to schedule it

### The Queue Sidebar
Cards you've queued (but not yet scheduled) appear in the **📋 Queue** sidebar on the left. Drag them to a day to schedule, or drag them back to remove the date.

### Recurrence
When completing a scheduled card, you can set it to recur:
- **Daily** — next day
- **Weekly** — same day next week (pick the day)
- **Bi-weekly** — same day, two weeks later
- **Monthly** — same date next month

### Month Strip
A scrollable strip of month boxes appears below the calendar grid. Dots indicate days with cards. Tap any month to jump to it.

---

## Kanban Board

The **Board** view shows cards as columns (each child of the current node is a column, and their children are the cards within that column).

- **Tap a column header** to drill into it
- **Tap a card** to drill into that card
- **Drag a card between columns** to move it (changes its parent)

---

## Deferring & Archiving

### Defer
Deferred cards are hidden from the main **All** view and appear in the **Deferred** filter. They show a date badge (Today, Tomorrow, day name, or M/D). When their date arrives, they reappear.

Options: 1 day, 1 week, 1 month from today.

### Archive
Archived cards are hidden from normal views. Access them via the **📁 View Archive** option in the session menu. Swipe right → Done on an archived card to unarchive it.

---

## Data & Sync

### Exporting & Importing
Tap the **⋯ menu** (or session name) to access:
- **Export JSON** — download a `.json` backup file
- **Copy JSON** — copy session JSON to clipboard
- **Import JSON** — load a `.json` file
- **Paste from Clipboard** — load JSON from clipboard

### GitHub Gist Sync
Connect a GitHub account to sync your session across devices automatically.

1. Create a **Personal Access Token** at github.com/settings/tokens with `gist` scope
2. Open the sync settings and paste your token
3. VoiceMap will find or create a Gist and sync every 30 seconds

The sync status (☁️ pushed / pulled / up to date) appears below the record button.

### Data Repair
On startup, VoiceMap automatically detects and repairs:
- **Circular parent references** (card A → parent B → parent A)
- **Orphaned parent IDs** (parent_id points to a deleted card)

A toast notification appears if any repairs are made.

---

## Session Management

### Renaming a Session
Tap the session title at the top of the screen to rename it.

### Session Name
The session name appears in exports and as the top-level title in the map view.

---

## Prompt Log

VoiceMap logs every Claude API call for transparency and debugging.

Tap the **log icon** (or access via settings) to see:
- Each operation with timestamp and duration
- The system prompt, user message, and response for each call
- Error messages if a call failed
- Copy buttons for each section

Up to 100 entries are stored. Tap **Clear** to reset.

---

## Troubleshooting

### PWA Not Updating
The build timestamp shown briefly at startup (e.g. `build:2026-03-26T11:00`) confirms which version is loaded. If you don't see the latest timestamp after an update, try: Settings → Safari → Clear History and Website Data, then re-add to home screen.

### Recording Not Working
- Ensure microphone permission is granted for the app in iOS Settings → Privacy → Microphone
- Try switching to keyboard input (Aa Type instead)

### Cards Appear in Search but Not in the Tree
This can happen if a card's parent was deleted. Run the app — it auto-repairs orphaned nodes on startup. The repaired cards will move to the root level.

### API Errors
- Check that your Anthropic API key is valid (Settings → Test)
- Ensure you have sufficient API credits at console.anthropic.com
- AI features gracefully degrade — you can still use VoiceMap without AI if needed

---

## Tips & Workflows

**Brain dump → organize**: Capture everything by voice at the root level, then use **Auto-Cluster** to group them into topics automatically.

**Consolidate for export**: Build out a topic with child cards, then use **Consolidate to Markdown** to turn the whole tree into a readable document in one step.

**Task planning**: Use voice to capture tasks as child cards, then use **Convert to Task List** to turn them into a checkbox list in the parent card's description.

**Daily review**: Use the **Deferred** filter to see everything due today. Swipe right → Done on completed items.

**Focus sessions**: In the map, double-tap any project card to enter Focus Mode and see only that project's cards, free of distraction.

**Cross-device sync**: Set up GitHub Gist sync so your session is available on all your devices with no manual export/import.
