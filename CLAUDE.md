# tlda - Paper Review & Annotation System

Collaborative annotation system for reviewing LaTeX papers. Renders PDFs as SVGs with TLDraw, supports KaTeX math in notes, real-time sync, and source-anchored annotations that survive document rebuilds.

## Quick Reference

| Task | Command |
|------|---------|
| **Start the server** | `tlda server start` |
| **Start all watchers** | `tlda watch-all start` |
| **Open in browser** | `tlda open <name>` |
| List projects | `tlda list` |
| Build status | `tlda status <name>` |
| LaTeX errors | `tlda errors <name>` |
| Visual check | `tlda preview <name> [page ...]` |
| Push files manually | `tlda push <name> --dir /path/to/project` |
| Monitor doc | `tlda monitor add <doc>` (auto-detect feedback via hook) |
| Block for feedback | `tlda listen <doc>` (one-shot, for idle agents) |
| Publish snapshot | `npm run publish-snapshot -- doc-name` |

**`tlda watch-all start`** auto-discovers all projects with a `sourceDir` and watches them. It polls for new projects every 30s, so `tlda create` picks them up automatically. This is the standard way to run watchers — no per-project `./watch` scripts needed.

**Never use `tlda build` to work around pipeline issues.** It bypasses change detection and masks bugs. If something isn't rebuilding when it should, fix the pipeline.

**IMPORTANT: Always use `tlda server start` to start the server.** It daemonizes properly and writes a PID file. NEVER use `node server/unified-server.mjs &` or run it in a background task — the server dies when the parent exits, leaving a zombie that holds the port but doesn't serve requests. Use `tlda server stop` to stop, `tlda server status` to check.

**If something goes wrong** (services won't start, build fails, viewer not loading, ports in use), delegate to the **ops agent** (`subagent_type: "ops"`). It knows the full build pipeline, service architecture, health checks, and common fixes.

## Markdown Format

tlda supports a `markdown` format for lightweight notes and scratch documents. No LaTeX build pipeline — the server renders the `.md` file with markdown-it + KaTeX and serves it as an HTML iframe page.

```bash
# Create a markdown project
tlda create my-notes --dir ~/work/notes/ --format markdown --title "My Notes"
# --main defaults to the first .md file found in the dir

# The watcher auto-detects .md changes and rebuilds
tlda watch-all start
```

Math works the same as in LaTeX: `$inline$` and `$$display$$`. KaTeX renders server-side; CSS served from `/katex/`.

The viewer uses the same `html-page` shape and iframe machinery as HTML/Quarto projects. All MCP annotation tools (`add_annotation`, `wait_for_feedback`, etc.) work normally. Source-line anchoring is not yet implemented for markdown — notes are placed visually on the canvas.

## TLDraw-Native UI Rule

**All UI that lives on the TLDraw canvas MUST use TLDraw-native patterns** unless there's a specific, documented reason not to. This means:

- **Shape state lives in shape props**, not in meta fields coordinated across multiple shapes
- **One shape = one visual unit.** Don't use N shapes with opacity toggling to simulate tabs/states. Use a single shape with data props (arrays, indices) instead.
- **Use TLDraw's event system** (`stopEventPropagation` from tldraw, not bare `e.stopPropagation()`). TLDraw uses capture-phase listeners; bare stopPropagation doesn't prevent TLDraw from intercepting events.
- **Don't fight TLDraw's selection/editing model.** If your component needs click handling, make sure it works *with* TLDraw's pointer state, not around it.

Deviations from this rule require justification in a code comment explaining why the TLDraw-native approach doesn't work. "It was easier" is not a justification.

**Visual design is deliberately subtle.** UI chrome should be nearly invisible until hovered or needed. Follow the conventions established by existing elements (e.g., `.build-warning-badge`): 10% opacity default, 60% on hover, 0.3s transition. Use CSS classes with `.tl-theme__dark` variants — never hardcode colors inline. New UI elements should look like they belong next to existing ones in size, weight, and opacity.

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
├── tlda.mjs                    # CLI entry point (installed as `tlda`)
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
├── data-source.mjs            # Reads doc assets from disk or HTTP (TLDA_SERVER)
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
│ tlda watch        │──POST /push───→ │ Project API → Build runner   │
│                  │                 │   latexmk → dvisvgm → etc.  │
│ Claude Code      │                 │   ↓                          │
│ └─ MCP (stdio)   │──Yjs WS──────→ │ Yjs sync + signal:reload     │
│                  │                 │   ↓                          │
│ iPad viewer      │←─Yjs WS───────│ Viewer SPA (/docs/* assets)  │
└──────────────────┘                 └──────────────────────────────┘
```

**Server URL resolution:** `TLDA_SERVER` env → `--server` flag → `~/.config/tlda/config.json` → `http://localhost:5176`

**Split sync server:** Set `TLDA_SYNC_SERVER` to route shapes/signals to a different server (e.g. Fly) while reading doc assets from `TLDA_SERVER` or local disk. Used for running Todd against the published version.

### Publishing and Todd

`npm run publish-snapshot -- <doc>` syncs the working copy to `~/work/published/tlda/`, builds the viewer, and deploys to GitHub Pages + Fly. The published clone is a frozen snapshot — safe for Todd to read from while the working copy keeps changing.

To run Todd against the published version:
```bash
cd ~/work/published/tlda
TLDA_SYNC_SERVER=https://tldraw-sync-skip.fly.dev node cli/lib/triage-agent.mjs
```

Todd reads doc assets (lookup tables, macros, page data) from the published clone on disk. Shapes and signals sync through Fly — the same room students are connected to.

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

1. Make sure the server is running: `tlda server start`
2. Start all watchers: `tlda watch-all start`
3. Open in browser: `tlda open <name>`

**If you'll be doing other work while the doc is open** (editing code, running sims, writing), enable background monitoring so feedback appears automatically:
```bash
tlda monitor add <name>
```

For an **iPad review session** (dedicated to review, not multitasking):
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

### Background listening (work + monitor)

When you need to do other work while monitoring a document, use `tlda monitor` to enable automatic feedback detection via a PostToolUse hook:

```bash
tlda monitor add spinoff3    # start monitoring (uses $AGENT_WIN)
tlda monitor list            # show what's monitored
tlda monitor remove spinoff3 # stop
tlda monitor clear           # stop all
```

Monitoring is scoped per agent via `$AGENT_WIN` — each agent has its own watch list and state. If `$AGENT_WIN` isn't set, pass `--id <name>` explicitly.

Once monitoring is active, the hook checks for new annotations, pings, and drawn shapes after every tool call (throttled to every 10s). Feedback appears automatically between tool calls — no polling, no re-launching, no background tasks. Just work normally and feedback shows up as `[tlda feedback] New note on spinoff3: "..."`.

For **idle agents** (nothing to do, waiting for input), use `tlda listen` instead — it blocks until feedback arrives, suitable for `run_in_background`:

```bash
tlda listen spinoff3 --timeout 600
```

**Summary:**
- **`tlda monitor`** — hook-based, automatic, for agents actively working
- **`tlda listen`** — blocking CLI, for idle agents or scripts
- **`wait_for_feedback`** — MCP tool, for dedicated review sessions (richer output, heartbeat)

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

**NEVER tell the user to check something.** Do not say "reload and check," "try it on the iPad," "go verify," "see if that works," or any variant. You have puppeteer, MCP tools, `tlda preview`, and screenshots. Use them. If you can't verify it yourself, say so explicitly — don't punt to the user.

**Verify before declaring success.** After deploying changes (server restart, SPA rebuild, viewer fix), open the viewer in playwright/puppeteer and confirm it actually works. Don't guess at CSS fixes — load the page and look.

**Look at layout, not just functionality.** When taking verification screenshots, actually examine proportions, spacing, and visual balance — don't just confirm that elements exist and render. A sidebar that's 80/20 instead of 50/50, text crammed into a sliver, an overlay that's misaligned by 100px — these are obvious to a human glancing at the screenshot. Check: Are columns balanced? Does text have room to breathe? Are things where they should be relative to each other? If you changed something that affects sizing or positioning, measure the actual computed values (grid columns, bounding rects, offsets) rather than eyeballing.

**Test in WebKit.** The user views on Safari/iPad. If you can't reproduce a reported problem, test in WebKit — Chromium passing doesn't mean Safari passes. Playwright supports WebKit: `playwright.webkit.launch()`.

**Never tell the user to force-refresh.** Open a new tab instead: `open -a Safari http://localhost:5176/?doc=NAME` or use playwright to open a fresh page. A new tab has no cache to worry about.

**Don't claim it'll work in Safari without justification.** If WebKit playwright fails, don't assert "that's just a playwright quirk, real Safari will be fine" unless you have a concrete reason. That's punting with extra steps.

If something works in Chromium but fails in WebKit playwright, and you genuinely believe it's a playwright-specific issue: explain why (e.g. known TDZ bug in minified bundles under strict mode). If you can't explain it, don't claim it.

If a bug isn't reproducible in playwright: try both Chromium and WebKit to narrow it down. If still not reproducible after trying both, you can involve the user — but set it up first: open the page, use MCP tools to scroll and screenshot as much as possible, and give them a specific thing to confirm rather than "go check if it works."

**Debug with live tools.** When something is visually broken in the viewer, use playwright/puppeteer to inspect the live page (console errors, DOM state, network requests). `tlda preview` renders static SVGs — it can't diagnose viewer runtime issues like blank pages, broken WebSocket, or CSS problems.

**If headless can't verify it, go headed.** If iframes, canvas rendering, or animations don't work in headless playwright, launch headed (`headless: false`), take screenshots at each step, and read them yourself. Don't punt to the user because your default verification tool has limits.

**For motion/interaction issues, record a video.** If the bug is about how something animates, transitions, or responds to a sequence of interactions, screenshots won't capture it. Use playwright's video recording:

```js
const context = await browser.newContext({ recordVideo: { dir: '/tmp/tlda-video/' } });
const page = await context.newPage();
// ... your test ...
await context.close(); // flushes the video
```

Then extract frames and read them:
```bash
ffmpeg -i /tmp/tlda-video/*.webm -vf fps=15 /tmp/frames/frame-%03d.png 2>/dev/null
```

Read the frames as images to see the full interaction sequence. For a specific moment, seek to a timestamp: `ffmpeg -ss 2.5 -i video.webm -frames:v 1 /tmp/frame.png`.

**When a feature is built, fixed, and verified, offer a tour.** After you've confirmed it works yourself, offer to run a headed playwright walkthrough — so the user sees the same thing you saw. This is confirmation, not verification. Don't offer before you've verified it yourself, and don't kick it off without asking.

**Read this file before starting any tlda session.** The self-service rule, verification patterns, TLDraw-native UI rules, and tool permissions are all here. Don't wait to be corrected on something that's already documented.

**Test exactly what the user said is broken.** If the user says "button X doesn't navigate to a new page," the test is: click button X, assert page changed. Not a broader test suite that touches the same code path. Don't test something adjacent and declare the reported issue fixed.

## Permissions

These operations are pre-approved for autonomous work:

- **Bash**: `npm run *`, `node`, `tlda`, shell scripts in this project, `curl` for local API testing, `open` for browser, process management (`pkill`, `lsof`)
- **Edit/Write**: Any file in this project
- **Git**: All operations within this repo (commit, push, branch, etc.)

**Restriction**: Git write operations (commit, push) in other repos require approval.
