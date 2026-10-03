# Ariadne — Software Specification

**Interactive reaction network editor for the browser.**

**Status:** forward-looking specification, not yet implemented. This document is written to be read by an AI coding agent: every requirement is meant to be concrete and testable, and every milestone ends with exit tests (§12).

> **Name:** "Ariadne" (the thread that finds a path through a labyrinth) is a placeholder. The name is used in exactly two places: the repository name and the Vite `base` path (§11.2). Changing it later is a two-line edit.

### Decisions taken

| Decision | Choice | Consequence |
|---|---|---|
| Language / stack | TypeScript + HTML5 Canvas 2D, bundled with Vite (React optional, for the UI chrome only). **Plain JavaScript (ES modules) is an acceptable substitute** if preferred; the module boundaries, invariants and tests in this spec apply either way | Model, geometry and layout are pure modules with no DOM dependency |
| Local development | **Python's built-in web server** (`python -m http.server 8000`, or `py -m http.server 8000` on Windows) serving the built site; develop locally, test, then push to GitHub Pages | The local check must match what Pages serves (§11.2) |
| Backend | **None.** Fully client-side static site | No accounts, no server; user data never leaves the browser |
| Hosting | **GitHub Pages**, project site on a sub-path | Static files only, no custom HTTP headers (§11) |
| SBML / Antimony | **libantimonyjs** (libAntimony + libSBML compiled to WebAssembly), as used by [MakeSBML](https://sys-bio.github.io/makesbml/). **Not** JSBML | No parser to write. Multi-MB, lazy-loaded (§7.2) |
| Layout in SBML | **libsbmljs** for the SBML Layout extension (see open question Q1) | libantimonyjs converts strings only; reading and writing layout needs an SBML object API |
| Primary save format | **SBML with the Layout extension** | Any SBML tool can open the file; styling limits in §7.6 |

### Open questions for Dr. Sauro

- **Q1.** libantimonyjs only exposes `convertAntimonyToSBML()` and `convertSBMLToAntimony()` (strings in, strings out). Reading and writing Layout glyphs needs an SBML object API such as libsbmljs. Is it acceptable to ship both libraries, or is there a libantimonyjs build that exposes layout?
- **Q2.** Is SBML+Layout acceptable as the only save format, or should there be a separate lossless native JSON format that also carries colours and styles? (Until the Render extension is implemented, colours are lost in SBML. See §7.6.)
- **Q3.** Antimony has no layout information, so export writes none by default. §7.5.1 proposes an optional layout comment block (an ignorable comment that only Ariadne reads). Is that format acceptable, or should it be dropped or aligned with what the desktop PathwayDesigner does?

---

## 1. Overview and goals

Ariadne is a web-based graphical editor for constructing, editing and visualising biochemical reaction networks (metabolic pathways, signalling networks, kinetic models). The user draws species and reactions on a canvas, and the app reads and writes SBML (with Layout) and Antimony.

### 1.1 Goals

1. **Draw and edit** reaction networks on a zoomable canvas, with clean, publication-quality curves.
2. **Keep diagrams readable** through alias nodes (§4.3), movable reaction centroids, and Bézier routing.
3. **Interchange** with the rest of the systems-biology ecosystem via SBML+Layout and Antimony.
4. **Zero install.** A static site on GitHub Pages; open the URL and draw. Works offline after first load.
5. **Export** high-resolution PNG and true vector PDF for papers.

### 1.2 Non-goals

Simulation. SBGN compliance. Accounts, cloud storage, collaboration or share links. Multiple documents or tabs. Mobile-first editing (touch is supported for pan and zoom only).

### 1.3 Reference implementation

Dr. Sauro's desktop editor, **PathwayDesigner** (<https://github.com/hsauro/PathwayDesigner>, a Windows release exists), is a working example of this kind of tool and the best source of clues for expected behaviour: how alias nodes, reaction junctions, Bézier handles and node gaps should look and feel. An implementing agent should consult it when this spec is silent on a detail. Where the two differ, **this spec governs**, because it describes what to build in the browser. Ariadne does not need to be file-compatible with PathwayDesigner's native format.

---

## 2. Technology and architecture

| Concern | Choice |
|---|---|
| Language | TypeScript (strict mode) |
| Build | Vite (`npm run build` produces `dist/`). `npm run dev` is an optional convenience |
| Local test server | **`python -m http.server 8000`** run against the build output, served from a sub-path that mimics GitHub Pages (§11.2). This is the official local check before every push |
| Rendering | HTML5 Canvas 2D, imperative; the canvas is never re-rendered by a UI framework |
| Antimony ⇄ SBML | libantimonyjs (WASM), vendored in the repo |
| SBML object model and Layout | libsbmljs (WASM), lazy-loaded |
| PDF | `jsPDF` + `svg2pdf.js`, lazy-loaded; vector output |
| Tests | Vitest (unit, property), Playwright (end-to-end) |

### 2.1 Module layout

The key rule: **nothing below `view/` imports the DOM, a canvas type, or a UI framework.** That is what makes the core testable in Node with no browser.

```
src/
  model/        BioModel types, factories, lookup, cascade delete, selection
  geometry/     vectors, rect/boundary intersection, bezier math, arrowhead polygons,
                world<->screen transforms        (pure arithmetic)
  layout/       force-directed auto-layout, bezier control point computation
  undo/         command objects + UndoManager
  io/
    wasm/       lazy loaders for libantimonyjs and libsbmljs
    antimony/   Antimony <-> SBML string bridge
    sbml/       SBML <-> BioModel bridge, Layout read/write, formula handling
    transforms/ application-level post-import passes
    image/      png.ts, svg.ts, pdf.ts
  view/         DiagramView class: owns the canvas, view state and interaction state machine
    render/     one module per draw pass
    interaction/ stateMachine.ts, hitTest.ts, drag.ts
    navigator/  minimap (§6.4)
  ui/           menus, toolbar, inspector, Antimony panel, dialogs, status bar
  theme/        theme object + presets (§9)
  prefs/        localStorage preferences
```

### 2.2 View ↔ UI boundary

`DiagramView` is a plain class that owns the model reference, the view transform and the interaction state, and paints imperatively to a `CanvasRenderingContext2D`. The UI layer talks to it through two callbacks only:

- `onNeedRepaint` — coalesced into a single `requestAnimationFrame`.
- `onSelectionChanged(info: SelectionInfo)` — the *only* thing that drives UI state. Toolbar enablement, the inspector, the context menu and the status bar all derive from `SelectionInfo`.

The UI never walks the model to compute render state and never stores model objects, only ids.

---

## 3. Domain model

Plain mutable TypeScript objects (the renderer reads them directly every frame). **All cross-references are by id string, never by object pointer.** This is what makes undo/redo, cloning and Web Worker transfer safe.

```ts
type Hex = string;                      // '#rrggbbaa' — also a valid CSS colour
interface Point { x: number; y: number }

interface VisualStyle {
  hasCustomStyle: boolean;              // false ⇒ use theme defaults
  fillColor: Hex; borderColor: Hex; labelColor: Hex;
  borderWidth: number;                  // 0 ⇒ theme default
  lineColor: Hex; lineWidth: number;    // 0 ⇒ theme default
  fontSize: number;                     // 0 ⇒ theme default
}

interface SpeciesNode {
  id: string;                           // unique diagram id; for a primary, also the SBML species id and label
  label: string;                        // displayed text
  center: Point; width: number; height: number;   // defaults 80 x 36
  initialValue: number;
  isBoundary: boolean; isConstant: boolean;
  compartment?: string;
  aliasOf?: string;                     // id of the PRIMARY node; undefined for a primary
  isNullNode: boolean;                  // sink/source "∅" node
  locked: boolean;
  style: VisualStyle;
}

interface Participant {
  speciesId: string;                    // id of the node this leg attaches to (primary OR alias)
  stoichiometry: number;
  ctrl1: Point; ctrl2: Point;           // Bézier handles, valid only if ctrlPtsSet
  ctrlPtsSet: boolean;
}

type ReactionMode = 'straight' | 'direct' | 'bezier';

interface Reaction {
  id: string;
  junctionPos: Point;                   // the centroid
  reactants: Participant[]; products: Participant[]; modifiers: Participant[];
  kineticLaw: string;                   // infix string
  isReversible: boolean;
  mode: ReactionMode;                   // one enum, never two booleans
  hasJunction: boolean;                 // false only for simple 1→1 'direct' reactions
  style: VisualStyle;
}

interface Parameter { id: string; value: number; isConstant: boolean }
interface Compartment { id: string; size: number; /* rendered in a later milestone */ }
```

`BioModel` holds arrays plus `speciesById` and `reactionsById` maps (rebuilt on load, maintained on add/delete/rename). All lookups go through the maps; the render loop does no linear searches.

Selection state is kept as sets of ids in the view, not as flags on the model objects.

---

## 4. Functional requirements

### 4.1 Canvas and workspace

- **Pan**: drag the background, or Space+drag, or middle-button drag, or two-finger trackpad scroll.
- **Zoom**: mouse wheel and trackpad pinch (`wheel` with `ctrlKey`), anchored at the cursor. Normalise `deltaMode`. Zoom range 10%–800%.
- **World ↔ screen**: `screen = world * zoom + offset`. Implemented as pure functions in `geometry/transform.ts`. Hit tolerances are in screen pixels; geometry constants are in world units.
- **Grid**: optional background grid with optional snapping of node and junction positions.
- **HiDPI**: the canvas backing store is scaled by `devicePixelRatio`; all code works in CSS pixels.
- **Viewport indicator**: status bar shows zoom % and cursor world coordinates.
- **Fit to content**: command to zoom and centre on the full extent of the diagram.

### 4.2 Species nodes

- **Create**: "Add species" tool, click on canvas. Ids auto-increment (`S1`, `S2`, …) and never collide with existing ids.
- **Edit**: rename (label and id together for primaries), initial value, boundary/constant flags, compartment.
- **Appearance**: fill colour, border colour, border width, label colour and font size. `fitNodeToText` auto-sizes the node to its label with 10 px padding, using `ctx.measureText` (cached per text and font size).
- **Node gap**: a configurable padding (default 4 world px) between a node's border and the point where a reaction curve ends, so curves never touch the node edge. Curves are stored centre-to-centre and clipped to the boundary at draw time; no boundary point is persisted, so moving or resizing a node never requires recomputing handles.
- **Lock**: a locked node cannot be dragged and is skipped by auto-layout.
- **Delete**: removes the node and cascades (§4.5).

### 4.3 Alias nodes

*This is a core feature and is built early (milestone M3).*

**Problem.** In a real pathway, ATP and ADP appear in dozens of reactions. With a single ATP node, every one of those reaction lines converges on one point and the diagram becomes unreadable.

**Solution.** The user may create **alias nodes**: visual mirrors of an existing node. One species has one **primary** node and any number of **alias** nodes. Each reaction that uses ATP attaches to its own nearby alias instead of to a distant shared node.

Requirements:

1. **Create alias**: right-click a species → "Create alias". The alias is placed near the cursor and is a new `SpeciesNode` with `aliasOf = <primary id>`. Aliasing an alias creates an alias of the *primary*, never a chain.
2. **Identity.** An alias has its own unique diagram `id` (e.g. `ATP_alias1`). It has the same biochemical identity as its primary. Its label, size and style are initially copied from the primary, and **label changes on the primary propagate to all aliases**.
3. **Visual distinction.** Aliases must be recognisable at a glance but still look like the same species (default: same fill with a dashed border, so they are distinguishable even in greyscale print). The style is a theme property (§9).
4. **Reactions attach to either.** A reaction leg's `speciesId` may refer to a primary or an alias. All aliases of a species are treated as the same species by anything biochemical.
5. **Biochemistry ignores aliases.** SBML and Antimony export emit each species once, from the primary, and reactions reference the *primary's* id regardless of which alias the leg attached to. Aliases appear only in the Layout block, each with its own glyph (§7.5).
6. **Select / navigate.** Selecting a primary can highlight all of its aliases; selecting an alias can highlight its primary (inspector button: "Select primary" / "Select all aliases").
7. **Delete rules.** Deleting an alias removes that node and the legs attached to it (the reactions themselves remain if they still have a leg on each side; see §4.5). Deleting a primary that has aliases **promotes** one alias to primary (reassigning `aliasOf` on the rest) instead of deleting the species; the user is asked to confirm, since this is the only way to remove the species entirely. If the user wants the species gone entirely, they choose "Delete species and all aliases".
8. **Import.** On SBML-with-Layout import, a second glyph referencing an already-placed species becomes an alias node (§7.5). Our own exports must round-trip exactly.

### 4.4 Reactions

- **Stoichiometry**: arbitrary N reactants to M products, plus any number of modifiers (activators or inhibitors).
- **Create**: "Add reaction" tool with variants UniUni, BiUni, UniBi and BiBi. The tool collects *n* reactant clicks, then *m* product clicks; pending participants are ringed (green for reactants, red for products). The junction is placed at the participant centroid.
- **Central junction (centroid)**: each reaction has a movable junction point from which the legs branch. It is drawn as a small dot (hidden optionally). Dragging it reshapes all legs.
- **Simple direct reactions**: a 1 → 1 reaction (A → B) may be drawn as a single curve with no junction (`hasJunction = false`, `mode = 'direct'`). The user can toggle a junction on, converting it to a normal reaction.
- **Modes**: `straight` (polyline through the junction), `bezier` (each leg is a cubic curve with two handles), `direct` (see above).
- **Arrowheads (required).** Every reaction draws an **arrowhead at the end of each product leg**, pointing into the product node, with the tip stopping at the node gap boundary. Reversible reactions additionally draw an arrowhead on the reactant legs. Reactant legs have no arrowhead for an irreversible reaction. Arrowhead size scales with line width (default 10 × line width, minimum 8 px). Modifier legs end in a distinct terminator: an open arrowhead for activation, a flat bar (⊣) for inhibition, with a small circle for unspecified modifiers. Arrowhead polygons are computed in `geometry/arrowhead.ts` from the tangent of the curve at its end (for Béziers, the direction from the last handle to the end point), not from the chord.
- **Bézier handles**: when a reaction is selected in `bezier` mode, each leg shows two draggable handles. A leg with `ctrlPtsSet = false` uses **auto handles** computed at render time (never stored); once the user drags a handle, `ctrlPtsSet = true` and the handles persist. "Reset curve" returns the leg to auto.
- **Self-loops** (`A → A + B`): the junction must not sit inside A's bounding box and the two legs to A must be visually separated.
- **Appearance**: per-reaction line colour, line width, junction visibility; kinetic law text and reversibility in the inspector.
- **Rate law and modifiers**: when the user edits the kinetic law, species ids appearing in the law that are not reactants or products are offered as modifiers (once the WASM formula parser is available, §7.4; before then, a plain regex over identifier tokens).

### 4.5 Selection, editing and deletion

- Click selects; Shift+click extends; rubber-band drag selects a region. Hit-test priority: Bézier handle → junction → species node → reaction leg → empty canvas.
- Drag moves the selection. Dragging a node carries its legs' handles (stored handles translate with the node).
- **Cascade delete**: deleting a species removes any reaction leg that references it. A reaction left with no reactants *and* no products is deleted. A reaction left with only one side gets a null node on the empty side if the "show null species" preference is on (§8.1), otherwise it is deleted.
- Delete/Backspace, Esc (cancel the current tool or drag), Ctrl/Cmd+A (select all).
- **Alignment and distribution** of a multi-selection: align left/right/top/bottom/centre-H/centre-V, distribute horizontally/vertically.
- **Context menus** by target kind: empty canvas, primary species, alias, reaction.

### 4.6 Auto-layout

Used when a model is imported without layout, and available on demand. Force-directed (ForceAtlas2 style) with reactions treated as junction nodes in the graph; locked nodes stay fixed. Defaults: repulsion 8000, attraction 0.02, gravity 0.02, 200 iterations, initial speed 4.0 decaying ×0.98 per step. Seeded RNG for reproducibility.

Requirements:

- Runs in a **Web Worker** (structured-clone `postMessage`; no `SharedArrayBuffer`, which GitHub Pages cannot enable, §11.1).
- **Looped reactions** (a species on both sides, e.g. `A → A + B`) must (a) deduplicate springs so the species is not double-pulled and (b) place the junction at the centroid of the *unique* species, otherwise the layout collapses. These are correctness requirements.
- After layout, compute Bézier control points for all reactions (`ctrlPtsSet` stays false, so they remain auto handles).
- The whole operation is a single undoable step.

### 4.7 Undo / redo

Command pattern: `MoveNodesCmd`, `MoveJunctionCmd`, `StyleEditCmd`, `DragCtrlPtCmd`, and a general `SnapshotCmd` (a serialised copy of the model) used for structural edits (add/delete, alias creation, auto-layout, import). Max depth 100; pushing a new command clears the redo stack; total snapshot memory is capped (64 MB, oldest evicted first). Ctrl/Cmd+Z, Ctrl/Cmd+Shift+Z and Ctrl+Y. A drag is one command, not one per frame.

### 4.8 Keyboard and input summary

| Input | Action |
|---|---|
| Wheel | Zoom at cursor |
| Wheel + Ctrl (pinch) | Zoom at cursor |
| Wheel (no Ctrl) / two-finger scroll | Pan |
| Space+drag, middle-drag | Pan |
| Delete / Backspace | Delete selection |
| Esc | Cancel tool or drag, clear selection |
| Ctrl/Cmd+Z / Shift+Z / Y | Undo / redo |
| Ctrl/Cmd+S / O | Save / open |

Use Pointer Events exclusively, with `setPointerCapture` on pointer-down and identical cleanup on `pointercancel` as on a cancelled drag. Use `event.metaKey || event.ctrlKey` for accelerators. Key handlers are bound to the focusable canvas (`tabIndex=0`), not `window`.

---

## 5. Rendering

### 5.1 Draw order

1. Background and grid
2. Compartments (later milestone)
3. Reaction legs (clipped to node boundaries, with arrowheads)
4. Modifier legs and terminators
5. Selection halos on legs
6. Species nodes (primaries and aliases), labels
7. Junction dots
8. Bézier handles and their guide lines (selected reactions only)
9. Pending-reaction rings, rubber-band rectangle
10. Tooltip (drawn on the canvas so it cannot be clipped)

### 5.2 Bézier legs

Each leg is a cubic Bézier from the species centre to the junction. At draw time, binary-search the parameter `t` where the curve crosses the node rectangle expanded by the node gap, split the curve with de Casteljau, and draw only the visible sub-curve with `ctx.bezierCurveTo`. Handle convention: for a reactant leg `ctrl1` is the species-side (outer) handle and `ctrl2` the junction-side (inner) one; product legs are the reverse.

### 5.3 Constants and theming

All sizes (default node size, handle radius, hit tolerances, node gap, arrowhead size, text padding) live in one `view/constants.ts`. All colours, fonts and line widths come from the active **theme object** (§9), never from literals in render code. Use a stable font stack (`system-ui, -apple-system, 'Segoe UI', Roboto, sans-serif`).

### 5.4 Performance budget

| Metric | Target |
|---|---|
| Frame time during drag, 200 species / 250 reactions | ≤ 16 ms |
| Frame time during drag, 1000 species / 1200 reactions | ≤ 33 ms |
| Cold load to interactive (cached), **no WASM loaded** | ≤ 1 s; initial bundle ≲ 300 KB gzipped |
| First import/export including WASM download | ≤ 5 s with visible progress; ≤ 300 ms afterwards |
| Auto-layout, 200 iterations, 200 nodes | ≤ 400 ms, off the main thread |

Order of techniques: coalesce repaints into one rAF → cache text measurement → skip empty passes → hoist context state changes out of loops → only then consider dirty rectangles.

---

## 6. User interface

Single window, single document: menu bar, toolbar, canvas with an optional Antimony side panel, inspector, status bar.

### 6.1 Toolbar and menus

Tools: Select, Add species, Add reaction (UniUni/BiUni/UniBi/BiBi), Pan. Commands: New, Open, Save, Export (PNG, SVG, PDF), Undo, Redo, Auto-layout, Fit to content, Align/Distribute, Create alias.

### 6.2 Inspector

Shows editable properties for the current selection (species, alias, reaction, or multiple). It is fully driven by `SelectionInfo`, with a guard that **suppresses change handlers while fields are being populated** so a selection change can never write the previous selection's values into the new one. Colour pickers use a shared palette plus a `#rrggbb` text field; open the picker → capture the style snapshot; close → commit one `StyleEditCmd`.

### 6.3 Antimony panel

A side panel showing the Antimony text for the current model, editable. "Apply" re-imports. Errors and warnings returned by libantimony are shown **inline** in the panel with line numbers where available.

### 6.4 Navigator (thumbnail preview)

*Optional, but part of the design; built in milestone M11.* Many editors call this a *navigation window* or *minimap*.

- A small panel (default 180 × 120 px) overlaid in the **bottom-right corner of the canvas**, showing a scaled-down rendering of the **entire** diagram extent (all nodes and reactions, not just the visible part).
- A translucent **viewport rectangle** shows the portion currently visible in the main canvas.
- Dragging the rectangle pans the main canvas; clicking elsewhere in the thumbnail recentres the main view there; scrolling the main canvas moves the rectangle live.
- The thumbnail updates when the model changes (throttled, ≤ 10 Hz during drags) and is drawn through the same render passes at a small scale, with text and handles suppressed below a size threshold.
- Collapsible with a small toggle, with the state remembered in preferences. It must not intercept pointer events destined for the main canvas outside its own bounds.

### 6.5 Accessibility

Every command reachable by keyboard; visible focus rings; canvas has `role="application"` and an `aria-label`; an off-canvas live region announces selection changes ("Species ATP selected", "3 items selected"); respect `prefers-reduced-motion`; default palette meets WCAG AA label-on-fill contrast. A keyboard-navigable model tree panel (species and reactions) linked to the canvas selection is a later nicety.

---

## 7. Import and export

### 7.1 Formats

| Format | Import | Export | Notes |
|---|---|---|---|
| **SBML (L3V1/L3V2) + Layout** | yes | yes | Primary save format |
| **Antimony** (`.ant`) | yes | yes — **no layout information by default**; optional layout comment block (§7.5.1) | Via libantimonyjs through SBML |
| **PNG** | — | yes | High resolution |
| **PDF** | — | yes | Vector |
| **SVG** | — | yes | Falls out of the PDF pipeline |

File type is detected by extension, then by content (`<sbml` → SBML; otherwise Antimony).

### 7.2 Loading the WASM libraries

libantimonyjs and libsbmljs are each several MB. Rules:

1. **Lazy.** Neither is loaded until the user first imports, exports, or opens the Antimony panel. Drawing, editing, undo, layout and image export must work with no WASM loaded at all.
2. **Memoise** the load promise so the library is instantiated at most once.
3. **Show progress** (determinate if possible) on first use; the UI must never appear frozen.
4. **Vendor libantimonyjs** (`libantimony.js` and `antimony_wrap.js`) into `web/vendor/libantimony/`, pinned, with a `VERSION` file and the upgrade procedure documented. Use the merged-wasm build so the `.wasm` is embedded in the `.js`. Follow the loading pattern used by MakeSBML (`libantimony().then(lib => new AntimonyWrapper(lib))`). Do not fetch from a CDN at runtime.
5. **Sub-path check**: verify early that any `.wasm` file resolves under the GitHub Pages sub-path (§11.2).
6. **Isolation.** Nothing outside `io/` imports either library. No library handle ever escapes `io/`.

### 7.3 Memory management

Objects returned by libsbmljs are C++ objects behind Emscripten bindings and are **not garbage collected**. Every function in `io/sbml` creates its document inside `try`/`finally`, calls `.delete()` on everything it owns, and returns **plain TypeScript values only** (a `BioModel` or a string). A test imports and discards a model 500 times and asserts the WASM heap does not grow without bound.

### 7.4 One bridge: SBML ⇄ BioModel

```
Antimony text ──libantimony──▶ SBML string ──libsbml──▶ SBML model ──bridge──▶ BioModel
BioModel ──bridge──▶ SBML model ──libsbml──▶ SBML string ──libantimony──▶ Antimony text
```

SBML is the interchange pivot; only one hand-written bridge exists, so Antimony support comes along with it.

**Import**: compartments; global parameters; species (prefer `initialConcentration`, fall back to `initialAmount` when concentration is unset); reactions with reactants, products, modifiers and stoichiometry; kinetic law via libSBML's `formulaToL3String(getMath())` stored as the infix `kineticLaw` string. Identifiers are never silently renamed (not even their casing).

**Export**: the inverse. `sanitizeSBMLId` is applied to identifiers; an empty or unparseable kinetic law emits **no** `<kineticLaw>` element rather than a broken one. Only primary species are emitted biochemically (aliases have no biochemical identity, §4.3).

Kinetic-law parsing in the editor (finding species in a rate law) uses libSBML's `parseL3Formula` when loaded and must never block a keystroke (debounce, or fall back to the regex until the library is ready).

### 7.5 SBML Layout extension

**Export.** Write a `layout:layout` containing: a `SpeciesGlyph` per node (**every** node, primaries and aliases, each with its own glyph id); a `ReactionGlyph` per reaction whose curve is the junction point; and `SpeciesReferenceGlyph`s per leg with `role` (substrate, product, activator, inhibitor) and a curve of cubic Bézier segments. An alias's glyph carries the alias's own diagram id and refers to its primary via `layout:species`. Text glyphs carry labels. Legs are exported clipped at node boundaries (using the same rectangle-intersection as the renderer).

**Import.** Read `SpeciesGlyph` bounding boxes for position and size. Read `ReactionGlyph` and `SpeciesReferenceGlyph` curves for junction position and Bézier control points (reversing the reactant/product handle convention) and set `ctrlPtsSet = true` and `mode = 'bezier'` when curve data exists. **A second glyph referencing an already-placed species becomes an alias node.** Any entity without a glyph is placed by auto-layout. A model with no layout at all is fully auto-laid-out.

### 7.5.1 Antimony export and the optional layout comment block

Antimony describes biochemistry only; it has no place for positions. Therefore:

- **Default export writes no layout.** The output is exactly what libAntimony produces from the SBML.
- **Optional layout block** (export dialog checkbox "Include layout as a comment", default off). Ariadne appends a comment block at the end of the file. Because it is a comment, libAntimony and every other Antimony tool ignore it:

```
/* ARIADNE-LAYOUT v1
{ "nodes": [ {"id":"ATP","alias":false,"x":120,"y":80,"w":80,"h":36},
             {"id":"ATP_alias1","aliasOf":"ATP","x":340,"y":210,"w":80,"h":36} ],
  "reactions": [ {"id":"J0","junction":[200,120],"mode":"bezier",
                  "legs":[{"species":"ATP","ctrl1":[150,90],"ctrl2":[180,110]}] } ] }
ARIADNE-LAYOUT-END */
```

- **Import:** before sending the text to libAntimony, Ariadne searches the raw text for the `ARIADNE-LAYOUT` block, parses it, and strips it. After the model is built, positions, aliases and handles are applied from the block; anything without an entry is auto-laid-out. A malformed or version-mismatched block is ignored with a warning, never an error.
- **Export:** the Antimony text is generated by libAntimony; Ariadne appends the block itself (libAntimony would drop it). The block contains only layout (no styles), is capped at a sensible size, and must not contain the sequence `*/` inside it.
- The block is a proposal (Q3); if it is rejected, this subsection is removed and nothing else changes.

### 7.6 Styling in SBML (limits and plan)

Core SBML and Layout carry geometry but not colours or line widths. Until the **Render extension** is implemented (milestone M13), custom styles are *not* preserved through SBML. The UI must say so on export ("styles are not saved in SBML"). When Render support is added, `VisualStyle` maps to Render primitives (fill, stroke, stroke-width, font-size) via a mapping table agreed in advance. (Q2 above asks whether a lossless native JSON should exist in the meantime.)

### 7.7 Application transforms (post-import passes)

Applied to the `BioModel` after the bridge, in `io/transforms/`, and inverted before export; they apply uniformly to Antimony **and** SBML import:

- **Catalytic rewrite** (preference, default on): a reaction `A -> A + B; k*A` becomes `-> B; k*A` with `A` as a modifier.
- **Null-node materialisation** (preference, default on): reactions with an empty side (`-> A`, `A ->`) get a null node (∅) on that side so the leg has something to attach to.

Both are unit-tested in isolation, in both directions, with the preference on and off.

### 7.8 Error reporting

Surface libantimony's failures and warnings inline in the Antimony panel. Offer **Validate SBML** (libSBML validation, listing severity and line). Malformed input must produce a clean error message, never an uncaught exception or a blank screen.

### 7.9 Image export

| Output | Approach |
|---|---|
| **PNG** | Render to an `OffscreenCanvas` sized `contentBounds × scale` (default scale 2; selectable 1–4), with selection, handle and tooltip passes suppressed; transparent or white background option |
| **PDF** | True **vector**. Render the diagram to SVG through the `Renderer` interface below, then convert with `svg2pdf.js` + `jsPDF`. Page size A4/Letter, portrait/landscape, fit-to-page with margin |
| **SVG** | Exposed directly, as it is produced anyway |
| **Print** | `window.print()` with a print stylesheet showing the diagram fit to the page |

To get vector output from the same drawing code, render passes draw through a narrow `Renderer` interface (`roundRect`, `circle`, `line`, `bezier`, `polygon`, `text`, `measureText`) with two implementations, `CanvasRenderer` and `SvgRenderer`. Keep the interface small; every method added must also be right in the SVG implementation.

### 7.10 Opening and saving files

Use the File System Access API where available (Chromium) so Ctrl+S saves in place. Elsewhere (Firefox, Safari) fall back to `<input type="file">` for open and a blob `<a download>` for save, and say which mode is active in the status bar. Drag-and-drop a file onto the canvas to open it.

**Autosave:** debounce 10 s after any change into IndexedDB; offer recovery on next load if newer than the last explicit save; warn on unload with unsaved changes. Autosave data never goes into the exported file.

---

## 8. Preferences

Stored in `localStorage` under `ariadne.`-prefixed keys (the Pages origin is shared with every other project of the same GitHub account, so unprefixed keys can collide).

```json
{ "convertCatalyticReactions": true, "showNullSpecies": true,
  "gridSnap": false, "showGrid": true, "nodeGap": 4, "navigatorVisible": true }
```

### 8.1 Preference meanings

`convertCatalyticReactions` and `showNullSpecies` gate the transforms in §7.7. The others control the grid, node gap and navigator.

---

## 9. Theming

All colours, fonts, stroke widths, arrowhead sizes, alias styling and node styling come from one **theme object**, never from literals in render code:

```ts
interface Theme {
  name: string;
  background: Hex; gridColor: Hex;
  node: { fill: Hex; border: Hex; label: Hex; borderWidth: number; fontSize: number };
  alias: { fill: Hex; border: Hex; borderDash: number[] };
  boundaryNode: { fill: Hex; border: Hex };
  reaction: { line: Hex; width: number; junction: Hex; arrowheadScale: number };
  modifier: { activation: Hex; inhibition: Hex };
  selection: { halo: Hex; handle: Hex };
  fontFamily: string;
}
```

Ship a `default` theme and a `grayscale/print` theme. An entity's own `VisualStyle` overrides the theme only where `hasCustomStyle` is true (and for numeric fields only when non-zero). Selecting a theme and importing user-defined theme files (JSON) is built in the final milestone, but the object is used from M1 so nothing needs refactoring.

---

## 10. Invariants

1. **Pure core.** No DOM, canvas or framework imports under `model/`, `geometry/`, `layout/`, `undo/` or `io/transforms/`.
2. **References are ids, not pointers.** Everything survives clone, undo and Worker transfer.
3. **One enum for reaction mode**, not multiple booleans.
4. **`ctrlPtsSet` is the single source of truth** for user-placed versus auto handles; auto handles are computed at render time and never stored.
5. **Legs are stored centre-to-centre and clipped at draw time.** No boundary point is persisted.
6. **Aliases are visual only.** They are skipped biochemically on every export, but each gets its own Layout glyph.
7. **User identifiers are never silently renamed**, casing included.
8. **Style fallback is by sentinel** (`hasCustomStyle: false`, or 0 for numeric fields).
9. **No WASM on the core path.** Drawing, editing, undo, layout, image export and autosave work with no WASM loaded.
10. **Static files only.** The build is servable from any directory with no rewrites, redirects or custom headers.
11. **No libSBML/libAntimony handle ever escapes `io/`.**

---

## 11. Hosting and deployment — GitHub Pages

### 11.1 What Pages gives and withholds

HTTPS is automatic (and required for the File System Access API and service workers). **Custom HTTP headers cannot be set**, so no COOP/COEP, no cross-origin isolation, no `SharedArrayBuffer`: workers use structured-clone `postMessage`, and no dependency may require threaded WASM. There are no server-side rewrites (deep links to non-files 404) and no server-side code at all. Soft repository limit about 1 GB, 100 MB per file.

### 11.2 Base path — the thing that silently breaks everything

A project site is served from `https://<user>.github.io/<repo>/`, not from the domain root, so absolute paths (`/assets/...`) 404.

- Set Vite's `base` once (`base: '/<repo-name>/'`) and never hand-write an absolute asset URL.
- Reference assets through the bundler (`import iconUrl from './icon.svg'`).
- Worker URLs, the service worker registration, and the manifest `start_url` and `scope` must all be base-aware (`import.meta.env.BASE_URL`).
- **Test the production build served from a sub-path locally**, not from `/` (for example, build then serve a parent directory containing the output folder named like the repo). A root-path preview will not reproduce this class of bug.

### 11.3 Routing

Single view, no router. If deep-linkable view state is ever wanted, use the URL hash (`#zoom=2`). No path-based routing.

### 11.4 Service worker (milestone M13)

Precache the app shell and all assets **except the WASM libraries**, which are cached on first use (precaching several MB would put the whole cost on the first visit and break the cold-load budget). Use a cache name keyed on the build hash and delete stale caches on `activate`. Because Pages cache headers cannot be overridden, the service worker is the update mechanism: show an explicit "new version available — reload" prompt.

### 11.5 Deployment

GitHub Actions on push to `main`:

1. `npm ci`.
2. Typecheck, lint, and run the full test suite. A test failure **must fail the deploy**.
3. Build with the correct base path.
4. Publish with `actions/upload-pages-artifact` + `actions/deploy-pages`. Build output is generated and never committed.

---

## 12. Development milestones and test anchors

Each milestone is independently shippable, small enough to hand to an AI agent in one session, and ends with **exit tests that must pass before the next milestone begins**. Tests are written first or alongside the code. Never start milestone *N+1* with milestone *N* red.

### M0 — Skeleton and WASM smoke test *(retire the biggest unknown first)*

- Vite + TypeScript project (or plain JS), Vitest, Playwright, CI workflow deploying to GitHub Pages (§11.5), with `base` set. Document the local workflow in the README: build, then `python -m http.server 8000` from the sub-path layout in §11.2, then open it in the browser.
- Throwaway page that lazy-loads libantimonyjs and converts one Antimony string to SBML and back.
- **Exit:** the deployed Pages URL (not localhost) loads, runs the conversion and shows the SBML text. The `.wasm` resolves under the sub-path. CI deploys on push.

### M1 — Model, geometry, canvas, pan/zoom, species

- `model/` types and factories; `geometry/` vector, rect and transform functions; theme object (§9); `DiagramView` with HiDPI canvas, pan, zoom at cursor, grid, status bar.
- Add species tool (auto-ids), select, drag, rubber-band, delete, rename, `fitNodeToText`.
- **Exit tests:** `worldToScreen(screenToWorld(p)) ≈ p` as a property test; zoom keeps the cursor's world point fixed; id generator never collides (including after deletes and renames); drag emits exactly one command; FSM tests with synthetic pointer events (no canvas needed); no stuck drag after `pointercancel`.

### M2 — Reactions: straight lines, junctions, arrowheads

- Reaction data model; Add reaction tool for UniUni/BiUni/UniBi/BiBi with pending-ring feedback; junction at centroid, movable; legs clipped at node boundary with node gap; **arrowheads on product ends** (and reactant ends when reversible); simple 1→1 `direct` reaction without junction; cascade delete.
- **Exit tests:** boundary-intersection function for rectangles, including degenerate cases (external point equals centre); arrowhead polygon tip lies exactly on the gap boundary and points along the leg direction; arrowhead count equals product-leg count (plus reactant legs if reversible); cascade-delete scenarios from §4.5; image-comparison reference render of a small network.

### M3 — Alias nodes *(early, because every later feature must understand them)*

- Create alias from context menu; alias visual style; label propagation; reactions attach to primaries or aliases; select primary/all aliases; delete rules including promotion; `SelectionInfo` target kinds `primary`/`alias`.
- **Exit tests:** aliasing an alias yields an alias of the primary; renaming a primary renames all aliases; deleting a primary with aliases promotes one and leaves all legs valid; every leg's `speciesId` always resolves; a diagram with one primary + 5 aliases keeps all 5 reactions attached to distinct nodes; alias is visually distinguishable in the grayscale theme (image comparison).

### M4 — Bézier curves and handles

- `bezier` mode; auto handles computed at render time with fan-out among legs sharing a node; draggable handles; `ctrlPtsSet` semantics; "reset curve"; clipped sub-curve drawing; arrowhead on the curve tangent; self-loop (`A → A + B`) placement rules.
- **Exit tests:** de Casteljau split reproduces the original curve (property test); `bezierBoundaryT` lands on the boundary within 0.5 px; moving a node does not change stored handles of a `ctrlPtsSet = false` leg; after a handle drag `ctrlPtsSet` is true and the handle survives a node move by translating with it; for `A → A + B` the junction is outside A's rectangle and the two A legs are separated by at least the node gap.

### M5 — Styling and inspector

- Inspector with the `FUpdatingControls`-style guard; colour popover and palette; per-entity styles (fill, border, line colour and width, font size, junction visibility); node gap setting; grid snap; context menus; alignment and distribution; tooltips.
- **Exit tests:** populating the inspector from a selection change fires no change events (regression test for the "writes previous selection's values" bug); style sentinels fall back to theme values; one colour edit = one undoable command; each of the eight align/distribute modes verified numerically.

### M6 — Undo / redo

- Five command types plus `SnapshotCmd`; escalation rule (a drag that also translates stored handles, or touches a smooth junction, takes a full snapshot at drag start); depth and byte caps; keyboard bindings.
- **Exit tests (property test):** a random 50-step edit session undoes step by step and redoes back to **identical serialised models at every step**; alias creation, alias deletion with promotion and cascade deletes all undo cleanly; pushing after undo clears redo.

### M7 — Auto-layout in a Worker

- ForceAtlas2 layout in a Web Worker with both looped-reaction fixes; locked nodes fixed; seeded RNG; bezier control point pass; single undo step; optional animated layout (disabled under `prefers-reduced-motion`).
- **Exit tests:** deterministic output for a fixed seed; locked nodes never move; `A → A + B` junction outside A's box with separated arcs; 200 nodes / 200 iterations ≤ 400 ms in the worker and no main-thread frame over 50 ms; a model of two disconnected components does not drift apart without bound.

### M8 — Native document handling and persistence

- New/Open/Save with File System Access API plus fallback; drag-and-drop; IndexedDB autosave and recovery; unload warning; preferences in `localStorage`.
- **Exit tests:** save → reload → identical model; recovery offered only when the autosave is newer than the last save; fallback path works with the File System Access API disabled (Playwright); all storage keys carry the prefix.

*(If Q2 is answered "yes", this milestone also defines the lossless native JSON format and its round-trip tests.)*

### M9 — Antimony and SBML import/export (no layout yet)

- Lazy loader with progress; memory-management discipline; the SBML ⇄ BioModel bridge; Antimony via the SBML pivot; catalytic-rewrite and null-node transforms; Antimony side panel with inline errors; Validate SBML; auto-layout on import.
- **Exit tests:** run against the real libraries in Node, no mocks. Antimony → BioModel → Antimony and SBML → BioModel → SBML preserve species, reactions, stoichiometry, kinetic laws and parameters; empty kinetic law emits no `<kineticLaw>`; Antimony export contains no layout by default; with the option on, export → import restores positions, aliases and handles from the comment block, and the same file still loads in plain libAntimony (block ignored); a malformed layout block is ignored with a warning; malformed input yields a clean error; transforms invert exactly (`A -> A + B; k*A` ⇄ `-> B; k*A` with modifier A); 500-import heap test; initial bundle still under budget with no WASM loaded.

### M10 — SBML Layout import and export

- Layout export through the SBML API with alias glyph convention; layout import (positions, junctions, Bézier curves, alias detection); fallback auto-layout for unplaced entities.
- **Exit tests:** our own SBML export re-imports with identical node positions, sizes, junction positions, handles and alias structure (semantic comparison, 1e-9 tolerance); a diagram with aliases round-trips to the same number of nodes; a layout-less SBML file imports and is auto-laid-out; a hand-edited file with a missing glyph places only that entity automatically.

### M11 — Navigator (thumbnail preview)

- Bottom-right minimap with viewport rectangle, drag-to-pan, click-to-recentre, throttled refresh, collapse toggle (§6.4).
- **Exit tests:** the thumbnail's content bounds match the model's content bounds; dragging the viewport rectangle by *d* thumbnail pixels pans the main view by *d / thumbnailScale* world units; panning the main canvas moves the rectangle; pointer events outside the thumbnail still reach the main canvas; ≤ 10 Hz refresh during a drag (instrumented).

### M12 — Image export and printing

- `Renderer` interface with `CanvasRenderer` and `SvgRenderer`; PNG (scale option); SVG; vector PDF; print stylesheet.
- **Exit tests:** exported PDF contains vector path operators and no embedded raster of the diagram; PNG dimensions equal `contentBounds × scale`; SVG and canvas outputs agree on node/arrow positions within 0.5 px; selection halos and handles never appear in exports; PDF/SVG libraries are loaded only on first export.

### M13 — Themes, Render extension, offline, polish

- Theme picker and user theme import; SBML **Render extension** export/import mapping `VisualStyle` (§7.6); service worker and PWA manifest with update prompt (§11.4); compartment rendering; accessibility pass (live region, model tree); cross-browser QA (Chromium, Firefox, Safari).
- **Exit tests:** a custom-styled diagram round-trips through SBML with colours and widths intact; app works offline after first load and after one import (WASM cached on first use); all WCAG AA contrast checks pass for shipped themes; Playwright suite green on all three browsers.

### Milestone dependency summary

```
M0 → M1 → M2 → M3 → M4 → M5 → M6 → M7 → M8 → M9 → M10 → M11 → M12 → M13
                 ↑ aliases early so M4–M10 all handle them
```

---

## 13. Testing strategy

| Layer | Approach |
|---|---|
| `geometry/` | Pure unit tests with full branch coverage, including degenerate cases (zero-length vectors, external point equal to rectangle centre) |
| `model/`, `undo/` | Property tests: random models survive serialise → parse; random edit sequences survive undo → redo to identical state |
| `io/` | Real libantimonyjs and libsbmljs in Node. **No mocks of the WASM libraries** (a mock of a C++ binding tests nothing). Include malformed-input, round-trip and heap-leak tests |
| `io/transforms/` | Isolated, both directions, preference on and off |
| `layout/` | Seeded RNG; assert the looped-reaction invariants directly |
| `view/` | FSM tested headlessly with synthetic pointer events; render passes tested by image comparison against stored references with a small per-pixel tolerance |
| End-to-end | Playwright: draw a network, save, reload, verify; import sample files; file fallback path; deployed-URL smoke test after each deploy |

Seed every RNG (layout, random-network generation) and expose the seed in the UI so bug reports are reproducible.

A sample corpus lives in `samples/` (a few `.ant` and `.xml` files, including one with heavy alias use such as an ATP/ADP-rich glycolysis model) and is used by the import, round-trip and render tests.

---

## 14. Third-party software and licensing

List libantimonyjs/libAntimony, libSBML/libsbmljs, jsPDF and svg2pdf.js, with their licences and upstream URLs, in an **About** dialog and a `THIRD-PARTY.md` file. Keep the vendored libantimonyjs unmodified with its version recorded. The project's own licence is chosen by the repository owner (MIT suggested); LGPL components are linked unmodified, which does not impose conditions on the app's own licence.