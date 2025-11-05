# graph explore — README

A single-file, browser-only tool to **visualize small colored graphs** (≤ ~20 vertices), interactively **move/fix vertices**, and **analyze perfect matchings**—including **multi-edges** and **edge colors**. No build step, no dependencies: open the HTML in any modern browser.

---

## Table of contents

- [What this tool does](#what-this-tool-does)
- [Quick start](#quick-start)
- [Input format](#input-format)
- [UI tour](#ui-tour)
- [Perfect matching panel](#perfect-matching-panel)
- [Edge semantics (colors, weight, multi-edges)](#edge-semantics-colors-weight-multi-edges)
- [Physics layout](#physics-layout)
- [Algorithm details](#algorithm-details)
- [Code structure](#code-structure)
- [Key data structures](#key-data-structures)
- [Performance notes & limits](#performance-notes--limits)
- [Customization](#customization)
- [Troubleshooting](#troubleshooting)
- [Ideas for extension](#ideas-for-extension)
- [License](#license)

---

## What this tool does

- **Draws colored, directed-by-half edges**: each edge is split at its midpoint with two colors (one per incident vertex half).
- **Marks negative weights** with a small diamond at the edge midpoint.
- **Handles multi-edges**: parallel edges between the same vertex pair are automatically rendered as separated curves.
- **Interactive layout**:
  - Drag a vertex to a new position → it becomes **fixed** (highlighted ring).
  - Click a fixed vertex to **release** it.
  - Click **Reposition** to randomize **non-fixed** vertices and let the spring model settle.
- **Perfect matchings (PMs)**:
  - Computes **all** perfect matchings (with multi-edge expansion).
  - Sorted by the **vertex coloring they induce** (lexicographic on color string like `00012034`).
  - Right panel shows one row of **colored dots** per PM (one dot per vertex, color = vertex color in that PM).
  - PM rows with the **same coloring** are wrapped by a subtle, rounded **group box** (size-1 groups included).
  - **Active box** (rounded, semi-opaque) acts as a **custom slider thumb**:
    - Drag the box or **click a PM row** to jump.
    - Top position is **“full”** (no PM filtering on the drawing).
  - **Edge selection** filters PMs:
    - Click an edge to toggle it as **required**; only PMs containing all selected edges remain **fully opaque**.
    - Non-qualifying PM rows remain visible but **translucent** (positions don’t shift).
    - Dots for vertices **incident to selected edges** get a subtle **glow**.
- **Export**: Download the current **SVG** or a **JSON** bundle (graph + vertex positions).

---

## Quick start

1. Open `graph-explore.html` in a modern browser (Chrome, Firefox, Safari, Edge).
2. Paste or edit the graph JSON in the left panel and click **Load Graph**.
3. Drag vertices to arrange; release to let physics settle; drag to fix.
4. Click edges to constrain perfect matchings; use the **active box** to browse PMs.
5. **Download SVG** or **Download JSON** when you like the view.

> Tip: You can also **Upload JSON** with a top-level `"graph"` object (see format below). If the file also includes `"vertexpositions"`, all vertices will load at those coordinates and start fixed.

> Saved graphs dropdown: Put your graph files in `graphs/` and list them in `graphs/manifest.json`. Supported shapes:

```json
// simplest: array of filenames
["example1.json", "example2.json"]

// or
{ "files": ["example1.json", "example2.json"] }

// or
{ "items": [ { "path": "example1.json", "label": "Example 1" } ] }
```

---

## Input format

Edges live in a JSON object mapping **edge keys** to **weights**:

```json
{
  "(u, v, c1, c2)": weight,
  "(0, 4, 1, 0)": 1.0,
  "(1, 3, 1, 1)": -1.0
}
```

- `u`, `v` — integer vertex ids (0-based is fine). `u != v`.
- `c1`, `c2` — **half-edge colors** (per endpoint). Supported values:
  - `0 → blue`, `1 → red`, `2 → green`, `3 → orange`, `4 → purple`,
    `5 → cyan`, `6 → yellow`, `7 → pink`.
- `weight` — any number; **negative** draws a diamond at edge midpoint.

You can either:
- paste this mapping directly in the textarea, or
- upload a file with either of the following shapes:
  ```json
  { "graph": { "(u, v, c1, c2)": weight, ... } }
  ```
  or with saved layout:
  ```json
  {
    "graph": { "(u, v, c1, c2)": weight, ... },
    "vertexpositions": {
      "0": { "x": 120.5, "y": 340.0 },
      "1": { "x": 220.0, "y": 180.2 }
      /* one entry per vertex id */
    }
  }
  ```

> **Multi-edges**: just repeat the same `(u, v, c1, c2)` with different color tuples (or same), and/or multiple entries for the same `(u, v, …)` key with different colors. They will render as separate curves and are fully included in PM enumeration.

---

## UI tour

**Header buttons**
- **Load Graph** — parse textarea JSON and (re)build.
- **Reposition** — randomize only **non-fixed** vertices then let physics re-settle.
- **Clear Fixed** — unfix all vertices.
- **Clear Edges** — clear required edge set (PM filter).
- **Download SVG** — exports current drawing as `graph-explore.svg`.
- **Download JSON** — exports `{ graph: { ... }, vertexpositions: { ... } }` as `graph-explore.json`.
- **Upload JSON** — pick a file with `{ "graph": { ... } }`.
  - If it has `"vertexpositions"`, vertices are placed accordingly and start fixed.
- **Saved graphs dropdown** — choose from entries in `graphs/manifest.json`; selecting one loads it immediately.
  - If it has `"vertexpositions"`, vertices are placed accordingly and start fixed.

**Canvas**
- **Drag a vertex** → it **fixes** at the drop position and gains a blue ring.
- **Click a fixed vertex** → **release** (physics will move it).
- **Edges**:
  - Click to **toggle required**. Selected edges get thicker + glow.
  - Negative weight shows a diamond at the midpoint.
  - Each edge half is colored by its endpoint’s color (`c1` near `u`, `c2` near `v`).
  - Multi-edges are gracefully **curved** to avoid overlap.

**Right panel (PM slider)**
- Top label **“full”** shows the entire graph (no PM emphasis).
- One **row per PM**: horizontally centered **colored dots** (one per vertex).
- **Group boxes** wrap contiguous rows sharing the same coloring.
- **Active box** is the **draggable thumb** centered on the active row:
  - **Drag** to browse PMs (snaps to nearest **allowed** row).
  - **Click any row** to jump to that PM.
  - Rows filtered out by selected edges remain in place but are **translucent** (non-interactive).

**Footer**
- Live count: `there are N perfect matchings for this graph[, M perfect matchings with the selected edges]`
- `(truncated)` appears if enumeration hit the limit (see below).

---

## Edge semantics (colors, weight, multi-edges)

- **Half-edge colors**: Edge is split at midpoint; the `u`-side half uses color `c1`, the `v`-side half uses `c2`.
- **Negative weight**: draws a **diamond** at the midpoint.
- **Multi-edges**: Parallel edges are rendered as **offset quadratic curves**. Matching enumeration treats each actual edge as distinct; a matching that uses `(u,v)` can use **any** one of the parallel edges, and all such choices are **expanded**.

---

## Physics layout

A lightweight force-directed layout:
- **Repulsion** between all vertices (`CHARGE`).
- **Springs** along edges targeting `SPRING_LENGTH`.
- **Damping** and **annealing** (`temperature`) stabilize the layout.
- **Fixed vertices** are excluded from forces and remain stationary.

All constants are in the script header; see [Customization](#customization).

---

## Algorithm details

### Enumerating perfect matchings

1. Build the set of **unique unordered vertex pairs** appearing in the graph (`uniquePairs`).
2. On the vertex set `V` (size `n`):
   - If `n` is odd → no PMs.
   - Build bit-mask adjacency `adj[i]` from `uniquePairs`.
   - Use recursive DFS with bitmasks to enumerate **pairings of vertices** (each vertex used exactly once), respecting adjacency.
3. **Expand** each pairing into **actual edge choices** across **multi-edges**:
   - For each paired vertex pair, take the **list of edge IDs** connecting that pair.
   - Do a product over these lists to generate **concrete matchings** (edge-level).
4. For each concrete matching, compute its **vertex coloring**:
   - For every chosen edge `(u,v,c1,c2)`, set `color[u]=c1`, `color[v]=c2`.
   - Concatenate as a string key (e.g., `"012430..."`) and **sort** all matchings by this key.

> **Constraint filtering**: when you **select edges**, a matching is eligible iff it **contains all selected edge IDs**. Eligible rows remain at their original fixed positions; **ineligible rows become translucent**.

### Truncation

To avoid combinatorial blow-ups, enumeration is capped at `MATCH_LIMIT = 50,000` concrete matchings. If reached, the footer shows `(truncated)`.

---

## Code structure

Everything lives in one HTML file. Primary DOM IDs:

- `#svg` — main canvas; groups `#edges`, `#nodes`.
- `#edgeInput` — textarea for inline JSON.
- Buttons: `#loadBtn`, `#repositionBtn`, `#clearFixBtn`, `#clearSelBtn`, `#downloadBtn`, `#downloadJsonBtn`, file `#fileInput`.
- PM UI: `#ticks` (rows & group boxes), hidden `#matchSlider` (state), draggable `.active-box`.

Key functions (search by name):

- **Parsing & build**
  - `parseGraphObject`, `parseEdgesFromMap`
  - `buildNodes`, `buildEdges` (sets multi-edge order `ord`/`ordCount`)
- **Layout & render**
  - `curvedSplit` (quadratic curve control for multi-edges)
  - `draw()` (edges, diamonds, nodes, labels, hit targets)
  - Force sim: `tickLayout`, `animate`, `startSimulation`
- **Interaction**
  - Node drag/fix: `startDrag`, `onDragMove`, `onDragEnd`
  - Edge toggle: `onEdgePointer` (single `pointerdown` listener)
  - PM UI: `renderTicksFixed`, `positionActiveBox`, `nearestAllowed`
  - Filtering: `refilterMatchings`, `applyMatchingSliderLimitsFixed`
- **PM enumeration**
  - `computePairings` (bitmask DFS on vertex pairs)
  - `expandAndSort` (multi-edge expansion + vertex coloring + sort)
- **Utilities**
  - `downloadSVG` (export current SVG)
  - `downloadGraphJSON` (export graph + vertex positions)
  - `showToast` (lightweight status)

---

## Key data structures

```ts
// Node
{ id: number, x: number, y: number, vx: number, vy: number, fixed: boolean }

// Edge
{
  id: number, u: number, v: number,
  c1: number, c2: number, w: number,          // colors & weight
  pk: string,                                  // pairKey(u,v)
  ord: number, ordCount: number                // multi-edge offset ordering
}

// Perfect matching (concrete, expanded)
{
  eids: number[],      // edge ids used by the matching
  colors: number[],    // per-vertex color result
  key: string          // colors.join('') for sorting/grouping
}

selectedEdges: Set<number>        // required edges
filteredIndices: number[]         // 1-based row indices that are allowed
allData: Array<PM>                // all perfect matchings (expanded)
pairToEdges: Map<pairKey, number[]>
uniquePairs: Set<pairKey>         // undirected adjacency for pairing DFS
vertexOrder: number[]             // sorted vertex ids (defines dot order)
```

---

## Performance notes & limits

- Intended for small graphs (≤ ~20 vertices). PM enumeration is **exponential**; multi-edges multiply results.
- `MATCH_LIMIT` safeguards runaway enumeration; increase cautiously if needed.
- Rendering is lightweight SVG; thousands of edges may start to feel heavy.

---

## Customization

Search in the script for these constants:

```js
// Physics
const SPRING_LENGTH = 120, SPRING_K = 0.08, CHARGE = 14000;
const DAMPING = 0.85, MAX_SPEED = 12;

// PM cap
const MATCH_LIMIT = 50000;

// Colors
const colorMap = {
  0:'#4ea1ff', 1:'#ff6b6b', 2:'#6ede8a', 3:'#ffa94d', 4:'#b28dff',
  5:'#2ed3ea', 6:'#ffd166', 7:'#ff85c0'
};
```

- **Add a color**: extend `colorMap` and allow its index in `parseEdgesFromMap`.
- **Edge thickness/glow**: tweak `--sel-thick` in `:root` and the softGlow filter.
- **Group/row visuals**: CSS classes `.tick-group`, `.tick-row`, `.active-box`.

---

## Troubleshooting

- **“Invalid JSON” / “Bad edge key”** — Check the textarea or uploaded file syntax.
- **0 perfect matchings** — Graph order must be even; ensure connectivity supports pairing; check for typos in vertex ids.
- **Rows look faded** — You’ve selected required edges; only PMs containing them are fully opaque.
- **Active box looks offset at ends** — Code centers by `translateY(-50%)`; ensure the `.ticks` container has `overflow: visible`.

---

## Ideas for extension

- Keyboard browse: **↑/↓** to move to previous/next allowed PM.
- Save/restore **vertex positions** and **selected edges** (localStorage).
- Export/import **graph + layout** bundle.
- **Tooltips** on PM rows (edge list summary).
- **Weighted** PM analysis (e.g., min/max weight selection).
- Toggle to show **only** eligible rows (currently we keep others translucent by design).
- Optional **labels** for colorings/groups.

---

**Happy exploring!** If you add features, keep the slider’s **fixed row positions** invariant under filtering—it’s a core UX choice that preserves spatial memory.
