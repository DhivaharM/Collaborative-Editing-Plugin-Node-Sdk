# Suggestion & Comment Features — Implementation Plan

**Jira:** FROALA-721 — Implement Suggestion & Comment Features for Collaborative Editing Plugin  
**Branch:** `FROALA-721-Collaborative`  
**Author:** DhivaharM  
**Date:** 2026-04-30

---

## 1. Architecture Overview

```
collaborative.js  (orchestrator — wire two new sub-modules)
├── CollabTrackChange  (full rebuild in collab_track_change.js)
├── CollabComments     (new flat peer: collab_comments.js)
└── collab_base.js     (add shared exports + CollabBase panel methods)
    ├── buildSuggestionCard()   ← named export (like _buildAvatarHTML)
    ├── buildCommentCard()      ← named export
    ├── _getOrCreatePanel()     ← CollabBase instance method
    ├── _renderPanel()          ← CollabBase instance method
    └── _scrollPanelTo(id)      ← CollabBase instance method
```

`buildSuggestionCard` and `buildCommentCard` are **pure functions exported from `collab_base.js`** — following the exact same pattern as `_buildAvatarHTML` (line 333, imported by `collab_core.js`).  
Panel methods are **inherited from `CollabBase`** — both `CollabTrackChange` and `CollabComments` call `this._getOrCreatePanel()` / `this._renderPanel()` with no peer imports.  
No `components/` subdirectory is created — all files remain flat peers inside `collaborative/`.

---

## 2. New Files

| File | Role |
|------|------|
| `src/js/plugins/collaborative/collab_comments.js` | Comment sub-module (flat peer — no subdirectory) |

All shared component logic (`buildSuggestionCard`, `buildCommentCard`, `_getOrCreatePanel`, `_renderPanel`, `_scrollPanelTo`) lives in the existing `collab_base.js` — no `components/` directory is created.

---

## 3. Files to Modify

| File | Change |
|------|--------|
| `collaborative/collab_base.js` | Add `buildSuggestionCard()`, `buildCommentCard()` as named exports; add `_getOrCreatePanel()`, `_renderPanel()`, `_scrollPanelTo()` as `CollabBase` instance methods |
| `collaborative/collab_track_change.js` | Full implementation — imports builders from `collab_base.js`, calls inherited panel methods |
| `collaborative.js` | Import + instantiate `CollabComments`; expose `addComment()` on public API |
| `src/sass/plugins/collaborative.scss` | Add suggestion / comment / panel styles |
| `src/sass/variables.scss` | Add new design tokens |

---

## 4. Color Scheme for Suggestion Highlights

Derived from the [Advanced-Track-Changes](https://www.figma.com/design/fn6ubzPvadDpcNKNzkXx4O/Advanced-Track-Changes) Figma file and the existing collab token palette (`$ui-color: #0098f7`, `$collab-cursor-default-color: #00B398`).

| State | Text Color | Background | Decoration |
|-------|-----------|------------|------------|
| **Insertion** | `#188038` | `rgba(52, 168, 83, 0.13)` | underline |
| **Deletion** | `#c5221f` | `rgba(217, 48, 37, 0.09)` | strikethrough |
| **Pending card** | — | `#ffffff` / border `#e8f0fe` | left-border `#1a73e8` |
| **Accepted** | `#188038` | `rgba(52, 168, 83, 0.06)` | — |
| **Rejected** | `#999` | none | strikethrough |
| **Comment highlight** | — | `rgba(249, 171, 0, 0.28)` | bottom-border `#F9AB00` |
| **Focused comment** | — | `rgba(249, 171, 0, 0.5)` | bottom-border `#F9AB00` |

### New SCSS tokens (add to `src/sass/variables.scss`)

```scss
// ─── Suggestion highlights ─────────────────────────────────────────────────────
$suggest-insert-color:   #188038;
$suggest-insert-bg:      rgba(52, 168, 83, 0.13);
$suggest-delete-color:   #c5221f;
$suggest-delete-bg:      rgba(217, 48, 37, 0.09);

// ─── Comment highlights ────────────────────────────────────────────────────────
$comment-highlight-bg:   rgba(249, 171, 0, 0.28);
$comment-focused-bg:     rgba(249, 171, 0, 0.5);
$comment-marker-color:   #F9AB00;

// ─── Side panel ───────────────────────────────────────────────────────────────
$collab-panel-width:     280px;
$collab-panel-bg:        $white;
$collab-card-radius:     8px;
$collab-card-padding:    12px 14px;
```

---

## 5. Data Model (Yjs-backed)

Both features store data in named `Y.Map` tables on the shared `Y.Doc`.

### Suggestions — `ydoc.getMap('fr-suggestions')`

```js
{
  id:            'uuid',
  type:          'insert' | 'delete' | 'replace',
  authorId:      'user-id',
  authorName:    'Jane',
  timestamp:     1234567890,
  originalText:  'old text',      // delete / replace only
  suggestedText: 'new text',      // insert / replace only
  anchor: {
    start: [...],                 // Y relative position (serialized as Array<number>)
    end:   [...]
  },
  status: 'pending' | 'accepted' | 'rejected'
}
```

### Comments — `ydoc.getMap('fr-comments')`

```js
{
  id:         'uuid',
  authorId:   'user-id',
  authorName: 'Jane',
  timestamp:  1234567890,
  text:       'comment body',
  anchor: {
    start: [...],
    end:   [...]
  },
  resolved: false,
  replies: [
    { authorId, authorName, text, timestamp }
  ]
}
```

Anchors use `Y.createRelativePositionFromTypeIndex`, serialized via `Y.encodeRelativePosition` as `Array<number>` so they survive JSON persistence and page reloads. This follows the exact same pattern as `CollabCore._domPointToRelPos` (line 557).

**Anchor type depends on where the selection boundary lands:**

| Selection boundary | Yjs anchor type | `typeIndex` meaning |
|--------------------|----------------|---------------------|
| Inside a text run | `Y.XmlText` | character offset |
| At an element boundary (paragraph start/end, before/after inline element) | `Y.XmlElement` | child index inside the element |
| At the document root | `Y.XmlFragment` (`ydoc.getXmlFragment('froala-content')`) | child index |

`Y.XmlElement` anchors are common — any selection that starts or ends at a block boundary produces one. Both `CollabCore._domPointToRelPos` and `_relPosToRange` already handle all three types.

---

## 6. Reusable Component Interface

All shared component logic lives in `collab_base.js`, following the `_buildAvatarHTML` export pattern.

```js
// collab_base.js — named exports (pure HTML builders)
export function buildSuggestionCard({
  id, type, authorName, timestamp,
  originalText, suggestedText, status, canResolve
}) // → HTMLString

export function buildCommentCard({
  id, authorName, timestamp,
  text, resolved, replies, canResolve
}) // → HTMLString

// collab_base.js — CollabBase instance methods (panel management)
// CollabTrackChange and CollabComments inherit these via `extends CollabBase`
CollabBase._getOrCreatePanel()               // → HTMLElement singleton, sibling of $wp
CollabBase._renderPanel()                    // reads ydoc maps, renders all cards
CollabBase._scrollPanelTo(id)               // scrolls panel to card matching id
```

Import pattern in sub-modules:
```js
import CollabBase, { buildSuggestionCard, buildCommentCard } from './collab_base.js'
// then inside the class: this._getOrCreatePanel(), this._renderPanel(), this._scrollPanelTo(id)
```

---

## 7. `collab_track_change.js` — Implementation Detail

### `_init()`

- Guard: check `collabOptions.enabled`; listen to `collab.modeChanged`
- Retrieve `ydoc.getMap('fr-suggestions')` once `CollabCore` has initialized the `Y.Doc`
- Observe the map: `suggestionsMap.observe(() => this._renderAll())`
- **Keydown intercept** (SUGGESTING mode only):
  - Snapshot the current selection range before Froala mutates the DOM
  - After the keystroke, diff old vs new text at that range
  - Wrap diff in `<ins class="fr-suggestion-insert" data-suggestion-id="…">` or `<del class="fr-suggestion-delete" data-suggestion-id="…">`
  - Push suggestion entry into `suggestionsMap` inside `ydoc.transact()`
- Register toolbar commands: `collabAcceptSuggestion`, `collabRejectSuggestion`, `collabAcceptAll`, `collabRejectAll`

### `_renderAll()`

1. Clear all `.fr-suggestion-insert` / `.fr-suggestion-delete` spans from the editor DOM
2. Resolve each suggestion's anchor via `CollabCore._relPosToRange()` (same helper used for remote cursors)
3. Wrap the resolved range in the appropriate `<ins>` or `<del>` span
4. Call `renderPanel(editor, suggestions, comments)`

### Accept / Reject

| Action | Steps |
|--------|-------|
| **Accept** | Resolve anchor → `abs.type` is `Y.XmlText` (direct patch) or `Y.XmlElement` (navigate to child text node at `abs.index`) → call `CollabCore._patchText(yText, originalText, suggestedText)` → mark `status: 'accepted'` → strip span |
| **Reject** | Same anchor resolution → `CollabCore._patchText(yText, suggestedText, originalText)` → mark `status: 'rejected'` → strip span |

`CollabCore._relPosToRange` returns a DOM `Range`; for text mutation, call `CollabCore._domPointToRelPos` on the range boundaries to get back to the Yjs layer. If `abs.type instanceof Y.XmlText`, patch directly. If `abs.type instanceof Y.XmlElement`, the target text node is `abs.type.get(abs.index)` — which must itself be a `Y.XmlText`.

All status updates wrapped in a single `ydoc.transact()` so remote peers see atomic state.

### Role-Based Access Control

| Role | Create suggestion | Accept / Reject own | Accept / Reject others' |
|------|:-----------------:|:-------------------:|:-----------------------:|
| `editor` | ✓ | ✓ | ✓ |
| `suggester` | ✓ | ✓ | ✗ |
| `viewer` | ✗ | ✗ | ✗ |

Enforcement: `canResolve` flag passed into `buildSuggestionCard()` hides action buttons. Server must re-validate on save.

---

## 8. `collab_comments.js` — Implementation Detail

### `_init()`

- Retrieve `ydoc.getMap('fr-comments')`
- Observe map: `commentsMap.observe(() => this._renderAll())`
- On `mouseup` / `selectionchange` inside the editor: if selection is non-empty and mode is not `VIEWING`, show a floating **Add Comment** bubble
- Bubble click → render an inline input (not `window.prompt`) → on submit, create comment entry in map inside `ydoc.transact()`

### `_renderAll()`

1. Clear all `.fr-comment-highlight` marks from editor DOM
2. Resolve each comment's anchor → wrap in `<mark class="fr-comment-highlight" data-comment-id="…">`
3. Click on mark → `scrollPanelTo(id)` + add `.fr-comment-focused` class to the mark
4. Call `renderPanel(editor, suggestions, comments)`

---

## 9. Panel Layout

```
┌─ Editor ($wp) ──────────────────┐ ┌─ Panel ────────────────┐
│                                 │ │                        │
│  text with <ins> highlights     │ │  ┌─ Suggestion card ─┐ │
│  text with <mark> highlights    │ │  │ Jane · 2 mins ago │ │
│                                 │ │  │ + "new text"      │ │
│                                 │ │  │ [Accept] [Reject] │ │
│                                 │ │  └───────────────────┘ │
│                                 │ │  ┌─ Comment card ────┐ │
│                                 │ │  │ Bob · 5 mins ago  │ │
│                                 │ │  │ "check this"      │ │
│                                 │ │  │ [Reply]           │ │
│                                 │ │  └───────────────────┘ │
└─────────────────────────────────┘ └────────────────────────┘
```

The panel is appended as a **sibling** of `$wp` (not inside it) so it does not interfere with the Yjs DOM observer. Both elements sit inside the `$box` container. Panel visibility is toggled by a toolbar button `collabPanel`.

---

## 10. Persistence

### Runtime state (Yjs — in-memory, per session)

- During a live session, suggestions and comments live in `Y.Map` entries on the shared `Y.Doc`.
- All peers share state in real time via the Node SDK WebSocket relay.
- `ydoc.transact()` wraps every write so peers see atomic updates.
- **The `Y.Doc` is not persisted on the server** — it is the live working copy only.

### Durable persistence (Node SDK DB — source of truth)

- Every suggestion / comment mutation (create, accept, reject, resolve, reply) is **also written to the backend DB** via the Node SDK REST API (see Section 13).
- The DB stores plain JSON records — no Yjs dependency on the server.
- On page load, the client **fetches the JSON records from the backend** and re-hydrates the `Y.Map` tables via `ydoc.transact()` before the Yjs sync handshake completes.
- This guarantees state survives server restarts, browser refreshes, and full peer disconnects.

### Real-time relay (Node SDK WebSocket)

- `CollabRealTime` connects to the Node SDK relay at `ws://<host>:<port>/<room>`.
- The relay broadcasts Yjs binary messages between peers — it has no knowledge of suggestions or comments.
- Live peer-to-peer updates propagate through Yjs; DB writes are fire-and-forget, decoupled from the Yjs protocol.

---

## 11. Implementation Phases

| Phase | Files | Work |
|-------|-------|------|
| **P1** | `variables.scss` | Add design tokens |
| **P2** | `collaborative.scss` | SCSS skeleton: panel, cards, highlight spans, comment bubble |
| **P3** | `collab_base.js` | Add `buildSuggestionCard()` + `buildCommentCard()` as named exports |
| **P4** | `collab_base.js` | Add `_getOrCreatePanel()`, `_renderPanel()`, `_scrollPanelTo()` as `CollabBase` instance methods |
| **P5** | `collab_track_change.js` | Yjs map init, mode guard, `_renderAll()`, highlight spans |
| **P6** | `collab_track_change.js` | Keydown intercept + suggestion creation |
| **P7** | `collab_track_change.js` | Accept / reject commands + role gating |
| **P8** | `collab_comments.js` | Full implementation |
| **P9** | `collaborative.js` | Wire `CollabComments`, register `collabPanel` toolbar command |
| **P10** | Node SDK (`D:\Projects\wysiwyg-editor-node-sdk`) | Add persistence routes (see Section 13) |
| **P11** | `demo/index.html` `demo/main.js` | Smoke-test with two user tabs |

---

## 12. Key Constraints

- **No existing track change code is reused** — `collab_track_change.js` is a clean rewrite of its current skeleton.
- **No new npm dependencies** — uses only existing Yjs (`Y.Map`, `Y.XmlText`, `Y.XmlElement`, `Y.XmlFragment`, relative positions), the editor's `$.js`, and native DOM APIs. `Y.XmlElement` is required because selection boundaries at block/inline element edges produce element-typed anchors, not text-typed anchors.
- **Panel lives outside `$wp`** — prevents the Yjs `MutationObserver` in `CollabCore` from picking up panel mutations as document edits.
- **All `Y.Map` updates are transactional** — `ydoc.transact()` wraps every status change so remote peers see atomic state.
- **Anchor serialization matches `CollabCore`** — `_domPointToRelPos` / `_relPosToRange` pattern is borrowed directly; no duplicate logic.
- **SCSS follows existing conventions** — `fr-` BEM prefix, `$collab-*` token namespace, no inline styles in JS.

---

## 13. Node SDK Backend Integration

**Repo:** `D:\Projects\wysiwyg-editor-node-sdk`  
**Entry:** `lib/froalaEditor.js` → `Collaborative` sub-module at `lib/collaborative.js`

> The Node SDK has **no Yjs dependency**. Suggestions and comments are stored as plain JSON records in the database and returned as JSON on load. Yjs is browser-only.

---

### 13.1 What the SDK Provides Today

| Capability | Detail |
|-----------|--------|
| WebSocket relay | `Collaborative.setupWSConnection(conn, req)` — pure Yjs message relay |
| Standalone server | `Collaborative.createServer({ port })` |
| Attach to Express | `Collaborative.attachToServer(httpServer)` |
| Health check | `GET /health` → `{ rooms, clients }` |

The relay is unchanged — it relays live Yjs updates between peers. Persistence is handled separately by the new REST endpoints below.

---

### 13.2 Database Schema

Add two tables to the SDK's database. For local dev, SQLite is sufficient (no extra infrastructure). For production, swap to PostgreSQL.

#### `suggestions` table

| Column | Type | Notes |
|--------|------|-------|
| `id` | TEXT PK | UUID generated by client |
| `room` | TEXT | Room name (document scope) |
| `type` | TEXT | `'insert'` \| `'delete'` \| `'replace'` |
| `author_id` | TEXT | |
| `author_name` | TEXT | |
| `timestamp` | INTEGER | Unix ms |
| `original_text` | TEXT | Null for insert |
| `suggested_text` | TEXT | Null for delete |
| `anchor_start` | TEXT | JSON array — serialized relative position |
| `anchor_end` | TEXT | JSON array — serialized relative position |
| `status` | TEXT | `'pending'` \| `'accepted'` \| `'rejected'` |

#### `comments` table

| Column | Type | Notes |
|--------|------|-------|
| `id` | TEXT PK | UUID generated by client |
| `room` | TEXT | |
| `author_id` | TEXT | |
| `author_name` | TEXT | |
| `timestamp` | INTEGER | Unix ms |
| `text` | TEXT | Comment body |
| `anchor_start` | TEXT | JSON array |
| `anchor_end` | TEXT | JSON array |
| `resolved` | INTEGER | `0` \| `1` (SQLite boolean) |
| `replies` | TEXT | JSON array of reply objects |

---

### 13.3 REST API Contract

All endpoints are added to `lib/collaborative.js` (or a new `lib/collab_persistence.js` imported by `froalaEditor.js`).

#### Suggestions

```
GET  /collab/:room/suggestions
  Response 200: JSON array of all non-deleted suggestion records for the room

POST /collab/:room/suggestions
  Body: { id, type, authorId, authorName, timestamp,
          originalText, suggestedText, anchor, status }
  Response 201: { id }
  Action: insert row; reject if id already exists (idempotent retry safe)

PATCH /collab/:room/suggestions/:id
  Body: { status: 'accepted' | 'rejected' }
  Response 200: { id, status }
  Action: validate requester role (server re-checks COLLAB_ROLES);
          update status column; return updated record

DELETE /collab/:room/suggestions/:id
  Response 204
  Action: hard-delete row (used when a user withdraws a pending suggestion)
```

#### Comments

```
GET  /collab/:room/comments
  Response 200: JSON array of all unresolved + recently resolved comments

POST /collab/:room/comments
  Body: { id, authorId, authorName, timestamp, text, anchor, resolved, replies }
  Response 201: { id }

PATCH /collab/:room/comments/:id
  Body: { resolved: true } | { replies: [...updatedRepliesArray] }
  Response 200: { id }

DELETE /collab/:room/comments/:id
  Response 204
```

---

### 13.4 Client-side Load Flow

On `CollabCore._init()` (after `Y.Doc` is ready, before WebSocket sync):

```js
// Inside collab_track_change.js _init()
async _loadFromBackend() {
  const room = this.editor.opts.collabOptions.room;
  const baseUrl = this.editor.opts.collabOptions.backendUrl; // e.g. 'http://localhost:3000'
  const res  = await fetch(`${baseUrl}/collab/${room}/suggestions`);
  const rows = await res.json();
  const map  = this.editor._collabYdoc.getMap('fr-suggestions');

  this.editor._collabYdoc.transact(() => {
    rows.forEach(row => map.set(row.id, row));
  });
}
```

Same pattern in `collab_comments.js` for `fr-comments`. Call both before `_renderAll()`.

---

### 13.5 Client-side Write Flow

Every `Y.Map` mutation is mirrored to the backend with a fire-and-forget `fetch`:

```js
// After ydoc.transact() that creates/updates the Y.Map entry:
_persistSuggestion(suggestion) {
  const { room, backendUrl } = this.editor.opts.collabOptions;
  fetch(`${backendUrl}/collab/${room}/suggestions`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(suggestion)
  }).catch(err => console.warn('[CollabTrackChange] persist failed', err));
}

_updateSuggestionStatus(id, status) {
  const { room, backendUrl } = this.editor.opts.collabOptions;
  fetch(`${backendUrl}/collab/${room}/suggestions/${id}`, {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ status })
  }).catch(err => console.warn('[CollabTrackChange] status update failed', err));
}
```

---

### 13.6 Editor-side Configuration

```js
new FroalaEditor('#editor', {
  collabOptions: {
    enabled: true,
    room: 'document-abc-123',
    user: { id: 'user-1', name: 'Alice', color: '#0098f7' },
    backendUrl: 'http://localhost:3000'   // ← new: Node SDK base URL
  },
  realTimeConfig: {
    syncUrl: 'ws://localhost:3000'        // WebSocket relay (unchanged)
  }
})
```

`asyncSaveEndpoint` / `asyncFetchEndpoint` are no longer needed — suggestions and comments are persisted independently as JSON records.

---

### 13.7 Node SDK Server Setup (local dev)

```js
// D:\Projects\wysiwyg-editor-node-sdk\examples\server.js  (extend, not replace)
const FroalaEditor  = require('../lib/froalaEditor.js');
const Collaborative = FroalaEditor.Collaborative;
const http          = require('http');
const express       = require('express');
const Database      = require('better-sqlite3');  // dev dependency only

const app = express();
app.use(express.json());
const db = new Database('collab.db');

// ── Schema bootstrap ───────────────────────────────────────────────────────────
db.exec(`
  CREATE TABLE IF NOT EXISTS suggestions (
    id TEXT PRIMARY KEY, room TEXT, type TEXT,
    author_id TEXT, author_name TEXT, timestamp INTEGER,
    original_text TEXT, suggested_text TEXT,
    anchor_start TEXT, anchor_end TEXT, status TEXT DEFAULT 'pending'
  );
  CREATE TABLE IF NOT EXISTS comments (
    id TEXT PRIMARY KEY, room TEXT,
    author_id TEXT, author_name TEXT, timestamp INTEGER,
    text TEXT, anchor_start TEXT, anchor_end TEXT,
    resolved INTEGER DEFAULT 0, replies TEXT DEFAULT '[]'
  );
`);

// ── Suggestions routes ─────────────────────────────────────────────────────────
app.get('/collab/:room/suggestions', (req, res) => {
  res.json(db.prepare('SELECT * FROM suggestions WHERE room=?').all(req.params.room));
});
app.post('/collab/:room/suggestions', (req, res) => {
  const s = req.body;
  db.prepare(`INSERT OR IGNORE INTO suggestions VALUES (?,?,?,?,?,?,?,?,?,?,?)`)
    .run(s.id, req.params.room, s.type, s.authorId, s.authorName, s.timestamp,
         s.originalText ?? null, s.suggestedText ?? null,
         JSON.stringify(s.anchor.start), JSON.stringify(s.anchor.end), s.status);
  res.status(201).json({ id: s.id });
});
app.patch('/collab/:room/suggestions/:id', (req, res) => {
  db.prepare('UPDATE suggestions SET status=? WHERE id=? AND room=?')
    .run(req.body.status, req.params.id, req.params.room);
  res.json({ id: req.params.id, status: req.body.status });
});
app.delete('/collab/:room/suggestions/:id', (req, res) => {
  db.prepare('DELETE FROM suggestions WHERE id=? AND room=?')
    .run(req.params.id, req.params.room);
  res.sendStatus(204);
});

// ── Comments routes ────────────────────────────────────────────────────────────
app.get('/collab/:room/comments', (req, res) => {
  res.json(db.prepare('SELECT * FROM comments WHERE room=?').all(req.params.room));
});
app.post('/collab/:room/comments', (req, res) => {
  const c = req.body;
  db.prepare(`INSERT OR IGNORE INTO comments VALUES (?,?,?,?,?,?,?,?,?,?)`)
    .run(c.id, req.params.room, c.authorId, c.authorName, c.timestamp,
         c.text, JSON.stringify(c.anchor.start), JSON.stringify(c.anchor.end),
         c.resolved ? 1 : 0, JSON.stringify(c.replies ?? []));
  res.status(201).json({ id: c.id });
});
app.patch('/collab/:room/comments/:id', (req, res) => {
  if ('resolved' in req.body)
    db.prepare('UPDATE comments SET resolved=? WHERE id=? AND room=?')
      .run(req.body.resolved ? 1 : 0, req.params.id, req.params.room);
  if ('replies' in req.body)
    db.prepare('UPDATE comments SET replies=? WHERE id=? AND room=?')
      .run(JSON.stringify(req.body.replies), req.params.id, req.params.room);
  res.json({ id: req.params.id });
});
app.delete('/collab/:room/comments/:id', (req, res) => {
  db.prepare('DELETE FROM comments WHERE id=? AND room=?')
    .run(req.params.id, req.params.room);
  res.sendStatus(204);
});

// ── WebSocket relay (unchanged) ────────────────────────────────────────────────
const server = http.createServer(app);
Collaborative.attachToServer(server);
server.listen(3000);
```

---

### 13.8 Integration Constraints

- **No Yjs on the server** — the Node SDK never imports or parses Yjs messages; all persistence is plain JSON SQL.
- **All backend calls go through `collabOptions.backendUrl`** — no hardcoded URLs in plugin source.
- **Idempotent POSTs** — `INSERT OR IGNORE` means a retry after a network blip does not duplicate records.
- **Role validation on PATCH** — the server must re-check `COLLAB_ROLES` before updating `status`; the client-side `canResolve` flag is UI-only.
- **Anchor arrays are opaque to the server** — stored and returned as JSON strings; the server never interprets them.
