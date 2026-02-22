# Claude TLDraw - Paper Review & Annotation System

Collaborative annotation system for reviewing LaTeX papers. Renders PDFs as SVGs with TLDraw, supports KaTeX math in notes, real-time sync, and source-anchored annotations that survive document rebuilds.

## Quick Reference

| Task | Command |
|------|---------|
| **Start the server** | `ctd server start` |
| **Start all watchers** | `ctd watch-all start` |
| **Open in browser** | `ctd open <name>` |
| List projects | `ctd list` |
| Build status | `ctd status <name>` |
| LaTeX errors | `ctd errors <name>` |
| Visual check | `ctd preview <name> [page ...]` |
| Push files manually | `ctd push <name> --dir /path/to/project` |
| Publish snapshot | `npm run publish-snapshot -- doc-name` |

**`ctd watch-all start`** auto-discovers all projects with a `sourceDir` and watches them. It polls for new projects every 30s, so `ctd create` picks them up automatically. This is the standard way to run watchers — no per-project `./watch` scripts needed.

**Never use `ctd build` to work around pipeline issues.** It bypasses change detection and masks bugs. If something isn't rebuilding when it should, fix the pipeline.

**IMPORTANT: Always use `ctd server start` to start the server.** It daemonizes properly and writes a PID file. NEVER use `node server/unified-server.mjs &` or run it in a background task — the server dies when the parent exits, leaving a zombie that holds the port but doesn't serve requests. Use `ctd server stop` to stop, `ctd server status` to check.

**If something goes wrong** (services won't start, build fails, viewer not loading, ports in use), delegate to the **ops agent** (`subagent_type: "ops"`). It knows the full build pipeline, service architecture, health checks, and common fixes.

## TLDraw-Native UI Rule

**All UI that lives on the TLDraw canvas MUST use TLDraw-native patterns** unless there's a specific, documented reason not to. This means:

- **Shape state lives in shape props**, not in meta fields coordinated across multiple shapes
- **One shape = one visual unit.** Don't use N shapes with opacity toggling to simulate tabs/states. Use a single shape with data props (arrays, indices) instead.
- **Use TLDraw's event system** (`stopEventPropagation` from tldraw, not bare `e.stopPropagation()`). TLDraw uses capture-phase listeners; bare stopPropagation doesn't prevent TLDraw from intercepting events.
- **Don't fight TLDraw's selection/editing model.** If your component needs click handling, make sure it works *with* TLDraw's pointer state, not around it.

Deviations from this rule require justification in a code comment explaining why the TLDraw-native approach doesn't work. "It was easier" is not a justification.

## Architecture

```
server/
├── unified-server.mjs        # Single process: Express + Yjs WS + SPA + API
├── lib/
│   ├── yjs-sync.mjs           # Yjs doc management + persistence
│   ├── project-store.mjs      # Project CRUD (server/projects/{name}/)
│   └── build-runner.mjs       # Build pipeline (latexmk → dvisvgm → synctex → proof-pairing)
├── routes/
│   └── projects.mjs           # REST API: /api/projects/*
├── projects/                  # Per-project storage
│   └── {name}/
│       ├── project.json       # Metadata (name, title, pages, buildStatus)
│       ├── source/            # Uploaded tex/bib/sty/cls/figure files
│       ├── output/            # Build output (SVGs, lookup, macros, proof-info)
│       └── build.log
├── data/{room}.yjs            # Persisted annotations per room
└── sync-server.js             # Legacy standalone Yjs server (still works)

cli/
├── ctd.mjs                    # CLI entry point (installed as `ctd`)
└── lib/
    └── watcher.mjs            # File watcher → HTTP push to server

src/                           # Viewer SPA (React + TLDraw)
├── SvgDocument.tsx            # SVG page loading, layout, reload handling
├── MathNoteShape.tsx          # KaTeX-enabled sticky notes
├── ProofStatementOverlay.tsx  # Proof reader overlays
├── useYjsSync.ts              # Real-time Yjs sync hook
├── synctexAnchor.ts           # Source-anchored annotation resolution
└── svgDocumentLoader.ts       # Document loading, manifest, proof-info

mcp-server/
├── index.mjs                  # MCP tools (wait_for_feedback, annotations, etc.)
├── data-source.mjs            # Reads doc assets from disk or HTTP (CTD_SERVER)
└── svg-text.mjs               # SVG text extraction for shape interpretation

public/docs/                   # Legacy doc storage (served as fallback)
├── manifest.json              # Legacy document registry
└── {doc-name}/                # SVGs + metadata
```

### How it fits together

```
Author's machine                     Server (localhost or remote, port 5176)
┌──────────────────┐                 ┌──────────────────────────────┐
│ Editor (Zed)     │                 │ unified-server.mjs           │
│     ↓ save       │                 │                              │
│ ctd watch        │──POST /push───→ │ Project API → Build runner   │
│                  │                 │   latexmk → dvisvgm → etc.  │
│ Claude Code      │                 │   ↓                          │
│ └─ MCP (stdio)   │──Yjs WS──────→ │ Yjs sync + signal:reload     │
│                  │                 │   ↓                          │
│ iPad viewer      │←─Yjs WS───────│ Viewer SPA (/docs/* assets)  │
└──────────────────┘                 └──────────────────────────────┘
```

**Server URL resolution:** `CTD_SERVER` env → `--server` flag → `~/.config/ctd/config.json` → `http://localhost:5176`

### For viewer development only

Working on the React/TLDraw code (not normal paper review):

```bash
node server/unified-server.mjs   # API + Yjs on 5176
npx vite                          # HMR on 5173, proxies /api and /docs to 5176
```

## Math Notes

Press `m` or click the note tool to create a math note.

Syntax:
- `$x^2$` - inline math
- `$$\int_0^1 f(x) dx$$` - display math

Custom macros from the paper's preamble are automatically available (e.g., `$\E[X]$`, `$\chis$`).

## iPad Review via MCP

### Starting a session
When the user asks to review or view a paper (e.g. "let's review this", "review bregman", "pull up the paper"):

1. Make sure the server is running: `ctd server start`
2. Start all watchers: `ctd watch-all start`
3. Open in browser: `ctd open <name>`

For an **iPad review session** (not just viewing), also:
1. Print a QR code: `node -e "import('qrcode-terminal').then(m => m.default.generate('http://IP:5176/?doc=DOC', {small: true}))"`
   - Get IP from `ifconfig | grep 'inet 100\.'` (Tailscale) or LAN
2. Open the tex file in Zed: `open -a Zed /path/to/file.tex`
3. Enter the listen-respond loop with `wait_for_feedback(doc)`

### Listening for feedback
Call `wait_for_feedback(doc)` in a loop. It blocks until:
- Ping (user tapped share) — immediate
- Text selection — 2s debounce
- Drawn shape (pen, highlight, arrow, geo) — 5s debounce
- Annotation edit — 5s debounce

### Reading annotations
- `read_pen_annotations(doc)` — all drawn shapes with source line mapping
- `list_annotations(doc)` — all math-note stickies

### Responding
- `add_annotation(doc, line, text, file?)` — persistent note anchored to source line
- `send_note(doc, line, text, file?)` — quick note via WebSocket + Yjs
- `reply_annotation(doc, id, text)` — create a reply in the note's thread (new tab)
- `highlight_location(file, line)` — flash red circle at source line
- `scroll_to_line(doc, line, file?)` — scroll viewer to source line

**Multi-file projects:** For documents that use `\input{}`/`\include{}`, pass the `file` parameter (e.g. `file="appendix.tex"`) to target lines in input files. Without `file`, tools default to the main tex file. The `lookup.json` keys input file lines as `"filename.tex:N"`.

### Note threading
Notes support reply chains via **threads**. A thread is a group of notes sharing the same canvas position, displayed as stacked tabs.

- `reply_annotation(doc, id, text)` adds a new tab to the note's `tabs` array and switches to it.
- `list_annotations(doc)` returns `tabCount`, `activeTab`, and `tabs` fields when a note has multiple tabs.
- `delete_annotation(doc, id)` deletes the entire note shape (all tabs).

On the viewer canvas, multi-tab notes show numbered tab handles above the note. The user can merge notes by dragging one onto another (tabs combine), or detach a tab via right-click.

The Notes tab in the panel has sort (document order / recency) and filter (all / pending MC / plain notes) controls.

### Cleanup
- `delete_annotation(doc, id)` — remove a note (deletes all tabs)

### Review loop behavior
When the user says they're reviewing a document, enter a listen-respond loop:
1. Call `wait_for_feedback(doc)` to block for the next annotation
2. Interpret what came in (pen stroke, highlight, text selection, etc.)
3. Scroll Zed to the relevant source line: `zed /path/to/file.tex:LINE`
4. Respond — drop a note, reply, answer the question, edit tex, whatever's needed
5. Call `wait_for_feedback(doc)` again automatically

Always keep Zed in sync: whenever you're discussing, highlighting, or responding to a specific source line, scroll Zed there with `zed file.tex:LINE`. This is the default behavior, not something the user should have to ask for.

If the user interrupts with a chat message, handle it, then resume `wait_for_feedback`. The default is to stay in the loop until the user says they're done.

### Diff review workflow

When starting a review of a diff document (`format: "diff"` in manifest):

1. **Populate summaries at session start.** Read `diff-info.json` and git diff to write a one-line summary per changed page:
   - Read `public/docs/{doc}/diff-info.json` to get page pairs and the git ref
   - Run `git diff {ref} -- {texfile}` in the tex repo to get the actual hunks
   - Map hunks to pages using the line ranges in diff-info
   - Write summaries to Yjs `signal:diff-summaries` via a Node one-liner:
     ```bash
     node -e "
     import WebSocket from 'ws'; import * as Y from 'yjs';
     const doc = new Y.Doc(); const ws = new WebSocket('ws://localhost:5176/DOC');
     ws.on('message', d => Y.applyUpdate(doc, new Uint8Array(d)));
     setTimeout(() => {
       const m = doc.getMap('records');
       doc.transact(() => m.set('signal:diff-summaries', {
         summaries: { PAGE: 'summary text', ... }, timestamp: Date.now()
       }));
       setTimeout(() => { ws.close(); process.exit(); }, 500);
     }, 1000);
     "
     ```
   - Keep summaries short: ~35 chars for simple changes, bullets with `\n` for complex ones
   - Focus on *what* changed semantically ("tightened bound in Prop 2.1"), not mechanically ("changed page 5")

2. **Triage with the user.** The Changes tab shows three status dots per change:
   - Blue = keep new version, Red = revert to old, Violet = discuss
   - Review state syncs via Yjs and adjusts highlight opacity on canvas
   - `n`/`p` keyboard shortcuts jump between changes with a pulse animation

3. **Don't redo decided changes.** When summaries and triage state already exist (from a previous session or earlier in the current one), respect them. Only update summaries if the diff itself changes (reload signal clears both).

### Proof reader

Press `r` to toggle proof reader mode. This highlights proof regions and shows a statement overlay panel (bottom-right) when scrolled to a cross-page proof.

**Statement panel** (green): shared-store TLDraw showing the theorem statement. Click header to jump to the statement page. Annotations drawn in the panel appear in the main view.

**Definition panel** (blue/indigo): appears above the statement panel when the proof references definitions, lemmas, or equations from other pages. Auto-selects the furthest-away dependency. Clickable badges in the statement header swap which dependency is shown; click the active badge to dismiss.

Data flow:
- `compute-proof-pairing.mjs` scans proof bodies for `\ref{}`/`\eqref{}`, builds a global label map, resolves to page regions, outputs `dependencies` array in `proof-info.json`
- `svgDocumentLoader.ts` loads `ProofDependency[]` per pair
- `ProofStatementOverlay.tsx` renders stacked panels with two shared-store TLDraw editors

Dependencies are sorted by page distance descending (furthest first). Same-page deps (dist=0) are filtered out. Section, figure, and table labels are excluded.

## Self-Service Rule

You have puppeteer and MCP tools available. Use them to check console output, read screen content, take screenshots, verify UI state, count elements, read error messages, etc. Never ask the user to report what they see on screen — do it yourself.

**Verify before declaring success.** After deploying changes (server restart, SPA rebuild, viewer fix), open the viewer in playwright/puppeteer and confirm it actually works before telling the user to look. "Go try it on the iPad" is not acceptable — you try it first.

**Debug with live tools.** When something is visually broken in the viewer, use playwright/puppeteer to inspect the live page (console errors, DOM state, network requests). `ctd preview` renders static SVGs — it can't diagnose viewer runtime issues like blank pages, broken WebSocket, or CSS problems.

## Permissions

These operations are pre-approved for autonomous work:

- **Bash**: `npm run *`, `node`, `ctd`, shell scripts in this project, `curl` for local API testing, `open` for browser, process management (`pkill`, `lsof`)
- **Edit/Write**: Any file in `/Users/skip/work/claude-tldraw/`
- **Git**: All operations within this repo (commit, push, branch, etc.)

**Restriction**: Git write operations (commit, push) in other repos require approval.
