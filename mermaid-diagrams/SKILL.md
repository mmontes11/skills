---
name: "mermaid-diagrams"
description: "Author and render/verify mermaid diagrams (flowchart, sequence, state, class, ER, C4, gantt) using mermaid-cli (mmdc). Use whenever the user wants to create, update, verify, or render a mermaid diagram, or wants an architecture/flow/sequence/state/ER diagram in Markdown, or when a ```mermaid block needs checking for validity before it lands in a file. Covers the turnkey `mermaid-render` wrapper, the required headless-Chromium no-sandbox config, and common mermaid authoring pitfalls."
license: MIT
allowed-tools: Bash, Read, Write, Glob, Grep
---

# Mermaid: Author and Render Diagrams

## Overview

Create and verify mermaid diagrams. Mermaid is a text-to-diagram language: fenced ` ```mermaid ` blocks in Markdown are rendered natively by GitHub (and most other hosts), so **diagrams are committed as text, never as images**. The one place an image is produced is *local verification* — you render the block to SVG/PNG with `mmdc` to confirm it is syntactically valid and legible before pushing.

> Validated against `@mermaid-js/mermaid-cli` (mmdc) v12.0.0 and headless Chromium 154. Flags are stable across recent releases, but run `mmdc --help` if a command behaves unexpectedly.

**Key insight:** the hard part is not the mermaid syntax — it is *rendering headless Chromium in a container*. `mmdc` drives a real Chromium browser to rasterize the diagram. Chromium's sandbox requires user namespaces, which are not available to a non-root container user, so **bare `mmdc` fails to launch**. A `--no-sandbox --disable-setuid-sandbox --disable-dev-shm-usage` Puppeteer config is mandatory. The [docker-opencode](https://github.com/mmontes11/docker-opencode) image bakes this in and ships a `mermaid-render` wrapper that applies it automatically.

## When to use / not use

- **Use** when creating, editing, or verifying any ` ```mermaid ` block, or when asked for an architecture / flow / sequence / state / ER diagram in Markdown.
- **Don't** reach for mermaid when the deliverable is a standalone image file, a C4 model the team manages in a dedicated tool, or a diagram that needs precise manual layout (mermaid auto-lays-out; use a dedicated diagramming tool instead).
- Prefer mermaid over committed PNG/SVG images: it diffs, it is reviewable, and it re-renders on the host for free.

## Prerequisites

- `mmdc` on PATH — `@mermaid-js/mermaid-cli` (baked into the docker-opencode image; otherwise `npm install -g @mermaid-js/mermaid-cli`).
- A no-sandbox Puppeteer config (baked at `~/.config/mermaid/puppeteer.json` in the image):
  ```json
  {"args":["--no-sandbox","--disable-setuid-sandbox","--disable-dev-shm-usage"]}
  ```
- The `mermaid-render` wrapper (baked on PATH in the image) — it wraps `mmdc -p <config>` so you do not repeat the flags.

## Render & verify workflow

mmdc takes a **file**, not a fenced block — so extract each ` ```mermaid ` block to a temp `.mmd` file, render it, and check the output. This loop handles any number of diagrams (validated against a real multi-diagram `architecture.md`):

1. **Write the block(s)** in the Markdown file (fenced `mermaid`).
2. **Split every mermaid block** in the file into `/tmp/d-<n>.mmd`:
   ```bash
   awk '
     /^```mermaid[[:space:]]*$/ { n++; inblk=1; next }
     /^```[[:space:]]*$/        { inblk=0 }
     inblk                       { print > ("/tmp/d-" n ".mmd") }
   ' FILE
   ```
3. **Render each** and check it came out non-empty:
   ```bash
   for f in /tmp/d-*.mmd; do
     if mermaid-render "$f" && [ -s "${f%.mmd}.svg" ]; then
       echo "OK:   $f"
     else
       echo "FAIL: $f"
     fi
   done
   ```
   A single diagram? Just `mermaid-render /tmp/d-1.mmd`. To render directly without the wrapper:
   `mmdc -i /tmp/d-1.mmd -o /tmp/d-1.svg -p ~/.config/mermaid/puppeteer.json`. PNG: `mermaid-render -i /tmp/d-1.mmd -o /tmp/d-1.png`.
4. **Verify**: a block is good only when its render exits 0 **and** writes a non-empty file. A parse error prints to stderr and produces no/empty output — treat any `FAIL` as "the block is broken", open the SVG, and fix it.
5. Iterate until every block is `OK`, then push. Overwrite temp files rather than `rm` (may be blocked): `echo "" > /tmp/d-*.mmd`.

## Choosing a diagram type

| Need | Mermaid type |
|------|--------------|
| Components + how they interoperate, request/data path, control plane | `flowchart TD` (or `LR`) |
| Message exchange between named participants over time | `sequenceDiagram` |
| States + transitions | `stateDiagram-v2` |
| Class / object relationships | `classDiagram` |
| Data model / tables | `erDiagram` |
| System context / C4 level 1 | `C4Context` (via the `c4-mermaid` include), or a labelled `flowchart` |
| Roadmap / schedule | `gantt` |
| Proportions | `pie` |

For a high-level "map of the system and how the parts talk" (the `architecture.md` use case), `flowchart TD` or `flowchart LR` is usually right, with `subgraph` to group components by layer (edge / data plane / control plane / storage) and edges labelled with the interaction (`-->|"SQL"|`).

## Command reference

| Command | Purpose |
|---------|---------|
| `mermaid-render file.mmd` | Render to `file.svg` (wrapper; auto no-sandbox config) |
| `mermaid-render -i in.mmd -o out.svg` | Explicit in/out |
| `mermaid-render -i in.mmd -o out.png` | Raster output |
| `mermaid-render -i in.mmd -o out.svg -b transparent -s 2` | Transparent background, 2× scale |
| `mmdc -i in.mmd -o out.svg -p ~/.config/mermaid/puppeteer.json` | Direct, no wrapper (the config flag is required) |
| `mmdc --help` | Full flag reference |

Common `mmdc` flags: `-i/--input`, `-o/--output`, `-b/--backgroundColor` (`white` / `transparent`), `-s/--scale`, `-w/--width`, `-p/--puppeteerConfigFile`, `-f/--find` (render only a specific diagram by id).

## Authoring pitfalls

1. **Quote node labels with special characters.** Any label containing `()`, `{}`, `[]`, `#`, quotes, or more than the simple case should be wrapped in double quotes: `A["VTGate (edge)"]`. Unquoted special characters break the parse.
2. **`<br/>` for line breaks inside labels.** `A["line one<br/>line two"]`. A raw newline inside an unquoted label is a syntax error.
3. **Pick a node shape deliberately.** `[rect]`, `(round)`, `((circle))`, `[[subroutine]]`, `>flag]`, `{decision}`. Do not mix shapes inconsistently — shapes carry meaning.
4. **`subgraph` titles with spaces need quotes** — `subgraph sg1["Data plane"]`.
5. **Edge labels** use `-->|"label"|` or `-- text -->`. Keep them short; long prose belongs in the surrounding text, not the diagram.
6. **`#` is a comment-ish character.** Avoid a raw `#` in a label (use a quoted label or reword).
7. **One diagram per concern.** A 30-node tangle is not high-level. Split into "overview", "data path", "control path" rather than one giant graph — this also keeps each render fast and legible.
8. **IDs must be unique and ASCII.** Use short node ids (`v1`, `vg`) and put the human name in the label.
9. **`%%` is the comment** — safe to leave in, stripped before render.
10. **C4 requires the include** at the top of the block. When in doubt, a labelled `flowchart` communicates the same context with zero extra dependency.

## Verification

- **Syntax + render**: `mermaid-render /tmp/d.mmd && [ -s /tmp/d.svg ]` — both must be true.
- **Legibility**: open the SVG (or render a small PNG) and confirm labels are readable and not overlapping. A diagram that *renders* but *overlaps* is still a failure.
- **GitHub preview**: a block that renders cleanly with `mmdc` will render on GitHub, since GitHub uses the same mermaid engine. No committed image is needed.

## Checklist

- [ ] Block is fenced ` ```mermaid ` in the Markdown file
- [ ] Node / edge labels with special characters are double-quoted
- [ ] `subgraph` titles with spaces are quoted
- [ ] Each diagram renders: `mermaid-render` exits 0 and the output is non-empty
- [ ] Diagram is legible (no overlapping labels); split it if it is too dense
- [ ] Diagram stays high level; detail lives in the linked prose / feature files
- [ ] No PNG/SVG committed — only the text block
- [ ] Temp `.mmd` / output files overwritten (not just deleted)
