# Obsidian Sync — Design Plan

## Overview

Bidirectional sync between VoiceMap card nodes and an Obsidian vault using a GitHub repository as the shared sync medium.

**Why GitHub repo (not other approaches):**

| Option | Problem |
|---|---|
| Obsidian Local REST API | Requires Obsidian running as a localhost server — no iOS support |
| File System Access API | Not available on iOS/iPadOS Safari |
| Obsidian Sync API | No public API exists |
| iCloud/Dropbox direct | PWAs cannot access cloud storage folders on iOS |
| **GitHub repo** | ✅ Works on iPad; VoiceMap already has GitHub token; Obsidian Git plugin handles the vault side |

---

## How It Works

```
VoiceMap (iPad)
     │  pushToObsidian() — writes .md files via GitHub Contents API
     ▼
GitHub Repository  ◄──── Obsidian Git plugin (auto-pull, e.g. every 5 min)
                                │
                                ▼
                        Obsidian Vault (Mac)
                                │  iCloud / Obsidian Sync (already configured)
                                ▼
                        Obsidian (iPad)
```

On edits in Obsidian → Obsidian Git commits/pushes → VoiceMap polls and pulls changes back.

---

## One-Time User Setup

1. Create a GitHub repo (e.g. `username/voicemap-notes`), or use an existing notes repo with a dedicated subfolder
2. In Obsidian on Mac: install **Obsidian Git** community plugin
3. Configure Obsidian Git to point at the same repo using the same GitHub token
4. Set Obsidian Git auto-pull interval (e.g. 5 minutes) and auto-push on change
5. In VoiceMap → Settings → **Obsidian Sync**: enter repo path and subfolder

---

## Markdown File Format

Each VoiceMap node becomes one `.md` file.

**File name:** `{sanitized-label}-{id[0..5]}.md`
Example: `morning-standup-abc123.md`

**File content:**
```markdown
---
voicemap_id: abc123def456
voicemap_parent_id: xyz789abc0
voicemap_status: in_progress
created: 2026-04-01T10:00:00.000Z
modified: 2026-04-01T12:00:00.000Z
---

Summary body text here. Supports **markdown**.
```

- `voicemap_id` — matches `node.id`; used to find the node on pull
- `voicemap_parent_id` — preserves hierarchy (empty string = root node)
- Body text (after second `---`) ↔ `node.summary`
- `node.label` is derived from the filename on import (strip `-{shortId}.md`, replace `-` with spaces, title-case)

---

## Implementation: New Code in `index.html`

All changes go in the single `index.html` file. No new files.

### 1. Constants  
_(in `// ─── CONSTANTS ───` section, near line 2576)_

```javascript
const OBSIDIAN_REPO_STORE   = 'voicemap_obsidian_repo';    // e.g. "username/my-notes"
const OBSIDIAN_FOLDER_STORE = 'voicemap_obsidian_folder';  // subfolder, default "voicemap"
```

### 2. Helper Functions  
_(in `// ─── GITHUB SYNC ───` section, after existing gist functions)_

```javascript
function getObsidianRepo()   { return localStorage.getItem(OBSIDIAN_REPO_STORE)   || ''; }
function getObsidianFolder() { return localStorage.getItem(OBSIDIAN_FOLDER_STORE) || 'voicemap'; }

function _obsidianSanitizeLabel(label) {
  return label.toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-+|-+$/g, '');
}

function _obsidianFileName(node) {
  return _obsidianSanitizeLabel(node.label) + '-' + node.id.slice(0, 6) + '.md';
}

function _obsidianNodeToMd(node) {
  const parentId = node.parent_id || '';
  return [
    '---',
    'voicemap_id: ' + node.id,
    'voicemap_parent_id: ' + parentId,
    'voicemap_status: ' + (node.status || 'untouched'),
    'created: ' + (node.created || ''),
    'modified: ' + (node.last_modified || ''),
    '---',
    '',
    node.summary || ''
  ].join('\n');
}

function _obsidianMdToNode(filename, rawContent) {
  // Parse YAML frontmatter
  const parts = rawContent.split(/^---\s*$/m);
  // parts[0] = '' (before first ---), parts[1] = frontmatter, parts[2+] = body
  const fm = {};
  if (parts.length >= 3) {
    for (const line of parts[1].trim().split('\n')) {
      const m = line.match(/^(\w+):\s*(.*)$/);
      if (m) fm[m[1]] = m[2].trim();
    }
  }
  const body = parts.slice(2).join('---').trim();
  // Derive label from filename: strip trailing "-{6chars}.md"
  const labelRaw = filename.replace(/-[a-z0-9]{6}\.md$/i, '').replace(/-/g, ' ');
  const label = labelRaw.replace(/\b\w/g, c => c.toUpperCase());
  return {
    id:            fm['voicemap_id']     || null,
    parent_id:     fm['voicemap_parent_id'] || null,
    status:        fm['voicemap_status'] || 'untouched',
    created:       fm['created']         || new Date().toISOString(),
    last_modified: fm['modified']        || new Date().toISOString(),
    label,
    summary: body
  };
}
```

### 3. `pushToObsidian()` (async)

Uses [GitHub Contents API](https://docs.github.com/en/rest/repos/contents):

```javascript
async function pushToObsidian() {
  const token  = getGithubToken();
  const repo   = getObsidianRepo();
  const folder = getObsidianFolder();
  if (!token || !repo) return;

  const base = 'https://api.github.com/repos/' + repo + '/contents/' + folder;
  const hdrs = githubHeaders(token);  // reuse existing helper

  // 1. List existing files to get their SHAs (needed for updates)
  let existingMap = {}; // filename → sha
  try {
    const res = await fetch(base, { headers: hdrs });
    if (res.ok) {
      const files = await res.json();
      for (const f of files) {
        if (f.name.endsWith('.md')) existingMap[f.name] = f.sha;
      }
    }
  } catch {}

  // 2. Push each non-archived node
  const activeNodes = session.nodes.filter(n => !n.archived);
  const pushedNames = new Set();
  for (const node of activeNodes) {
    const name    = _obsidianFileName(node);
    const content = btoa(unescape(encodeURIComponent(_obsidianNodeToMd(node))));
    const body    = JSON.stringify({
      message: 'VoiceMap sync: ' + node.label,
      content,
      ...(existingMap[name] ? { sha: existingMap[name] } : {})
    });
    try {
      await fetch(base + '/' + name, { method: 'PUT', headers: hdrs, body });
      pushedNames.add(name);
    } catch {}
  }

  // 3. Delete files for archived/removed nodes
  for (const [name, sha] of Object.entries(existingMap)) {
    if (!pushedNames.has(name)) {
      const body = JSON.stringify({ message: 'VoiceMap: remove ' + name, sha });
      try { await fetch(base + '/' + name, { method: 'DELETE', headers: hdrs, body }); } catch {}
    }
  }

  updateSyncStatus('🗒️ pushed');
}
```

### 4. `pullFromObsidian()` (async)

```javascript
async function pullFromObsidian() {
  const token  = getGithubToken();
  const repo   = getObsidianRepo();
  const folder = getObsidianFolder();
  if (!token || !repo) return;

  const base = 'https://api.github.com/repos/' + repo + '/contents/' + folder;
  const hdrs = githubHeaders(token);

  let files;
  try {
    const res = await fetch(base, { headers: hdrs });
    if (!res.ok) return;
    files = await res.json();
  } catch { return; }

  let changed = false;
  for (const f of files) {
    if (!f.name.endsWith('.md')) continue;
    try {
      const res = await fetch(f.url, { headers: hdrs });
      if (!res.ok) continue;
      const data    = await res.json();
      const rawText = decodeURIComponent(escape(atob(data.content.replace(/\n/g,''))));
      const parsed  = _obsidianMdToNode(f.name, rawText);

      if (parsed.id) {
        // Update existing node if Obsidian version is newer
        const existing = session.nodes.find(n => n.id === parsed.id);
        if (existing) {
          if (new Date(parsed.last_modified) > new Date(existing.last_modified)) {
            existing.summary       = parsed.summary;
            existing.status        = parsed.status;
            existing.last_modified = parsed.last_modified;
            changed = true;
          }
        }
      } else {
        // New note created in Obsidian — create a VoiceMap node
        const newNode = {
          id:            makeId(),
          label:         parsed.label,
          summary:       parsed.summary,
          transcript:    '',
          parent_id:     session.nodes.find(n => n.id === parsed.parent_id) ? parsed.parent_id : null,
          status:        parsed.status,
          archived:      null,
          pinned:        null,
          created:       parsed.created,
          last_modified: parsed.last_modified
        };
        session.nodes.push(newNode);
        changed = true;
      }
    } catch {}
  }

  if (changed) {
    await saveSession();
    renderCards();
    showToast('Synced from Obsidian ✓');
    updateSyncStatus('🗒️ pulled');
  }
}
```

### 5. Scheduling

```javascript
let _obsidianSyncTimer = null;
function scheduleObsidianSync() {
  if (!getObsidianRepo()) return;
  clearTimeout(_obsidianSyncTimer);
  _obsidianSyncTimer = setTimeout(pushToObsidian, 3000);
}

let _obsidianPollTimer = null;
function startObsidianPolling() {
  if (_obsidianPollTimer) return;
  _obsidianPollTimer = setInterval(async () => {
    if (!getObsidianRepo()) return;
    await pullFromObsidian();
  }, 60000); // every 60s
}
```

Call `scheduleObsidianSync()` from `saveSession()` after the existing `scheduleSyncToGist()` call.

### 6. Settings Modal HTML  
_(near `#githubModal`)_

```html
<div class="modal-overlay hidden" id="obsidianModal">
  <div class="modal-sheet">
    <div class="modal-title">🗒️ Obsidian Sync</div>
    <div class="modal-desc">
      Sync cards to your Obsidian vault via a GitHub repository.<br><br>
      Uses your existing GitHub token. In Obsidian, install the
      <strong>Obsidian Git</strong> community plugin and point it at the same repo.
    </div>
    <div style="font-size:11px;color:#9ca3af;margin-bottom:4px">GitHub repo (owner/repo):</div>
    <input class="modal-input" id="obsidianRepoInput" type="text"
           placeholder="username/my-notes" autocomplete="off" autocorrect="off"
           autocapitalize="none" spellcheck="false" />
    <div style="font-size:11px;color:#9ca3af;margin-bottom:4px">Subfolder in repo (default: voicemap):</div>
    <input class="modal-input" id="obsidianFolderInput" type="text"
           placeholder="voicemap" autocomplete="off" autocorrect="off"
           autocapitalize="none" spellcheck="false" style="margin-bottom:8px" />
    <button class="modal-save" id="obsidianSave">Save &amp; sync now</button>
    <button class="modal-save" id="obsidianTest" style="background:#6b7280">Test connection</button>
    <div id="obsidianTestResult" style="font-size:12px;color:#6b7280;word-break:break-all;white-space:pre-wrap;max-height:80px;overflow-y:auto"></div>
    <button class="modal-save" id="obsidianClose" style="background:transparent;color:#6b7280;margin-top:4px">Close</button>
  </div>
</div>
```

### 7. Modal JS Functions

```javascript
function showObsidianModal() {
  document.getElementById('obsidianRepoInput').value   = getObsidianRepo();
  document.getElementById('obsidianFolderInput').value = getObsidianFolder();
  document.getElementById('obsidianTestResult').textContent = '';
  document.getElementById('obsidianModal').classList.remove('hidden');
}
function hideObsidianModal() {
  document.getElementById('obsidianModal').classList.add('hidden');
}
async function testObsidianConnection() {
  const token  = getGithubToken();
  const repo   = document.getElementById('obsidianRepoInput').value.trim();
  const folder = document.getElementById('obsidianFolderInput').value.trim() || 'voicemap';
  const result = document.getElementById('obsidianTestResult');
  if (!token) { result.textContent = 'No GitHub token saved — set one in GitHub Sync first'; return; }
  if (!repo)  { result.textContent = 'Enter a repo path first'; return; }
  result.textContent = 'Testing…';
  try {
    const res = await fetch(
      'https://api.github.com/repos/' + repo + '/contents/' + folder,
      { headers: githubHeaders(token) }
    );
    if (res.ok || res.status === 404) {
      result.textContent = res.status === 404
        ? 'Repo found — folder will be created on first sync'
        : 'Connected ✓ — ' + (await res.json()).length + ' files in folder';
    } else {
      result.textContent = 'Error ' + res.status + ': check repo name and token permissions';
    }
  } catch (e) {
    result.textContent = 'Network error: ' + e.message;
  }
}
function saveObsidianSettings() {
  const repo   = document.getElementById('obsidianRepoInput').value.trim();
  const folder = document.getElementById('obsidianFolderInput').value.trim() || 'voicemap';
  localStorage.setItem(OBSIDIAN_REPO_STORE,   repo);
  localStorage.setItem(OBSIDIAN_FOLDER_STORE, folder);
  hideObsidianModal();
  if (repo) {
    showToast('Obsidian sync configured — syncing…');
    pushToObsidian();
    startObsidianPolling();
  }
}
```

Wire in `DOMContentLoaded`:
```javascript
document.getElementById('obsidianSave').addEventListener('click', saveObsidianSettings);
document.getElementById('obsidianTest').addEventListener('click', testObsidianConnection);
document.getElementById('obsidianClose').addEventListener('click', hideObsidianModal);
```

### 8. Settings Menu Button

Add alongside the existing "☁️ GitHub Sync" button (wherever it appears in the toolbar/menu):
```html
<button onclick="showObsidianModal()">🗒️ Obsidian Sync</button>
```

### 9. `initApp()` Addition

After the existing `await pullFromGist(); if (getGithubToken()) startGistPolling();` block:
```javascript
await pullFromObsidian();
if (getObsidianRepo()) startObsidianPolling();
```

---

## Conflict Resolution

| Scenario | Action |
|---|---|
| VoiceMap node `last_modified` newer | `pushToObsidian` overwrites the file |
| Obsidian file `modified` frontmatter newer | `pullFromObsidian` updates node summary/status |
| Node archived in VoiceMap | File deleted from repo |
| New `.md` file in repo (no `voicemap_id`) | New node created in VoiceMap on next pull |
| File deleted from repo | Node stays in VoiceMap (no destructive deletes on pull) |

---

## Out of Scope (Future)

- Image/attachment sync to Obsidian
- Deep hierarchy reconstruction from nested Obsidian folders
- Obsidian tags ↔ VoiceMap status mapping
- Custom Obsidian plugin build
- iCloud/Dropbox direct file access
