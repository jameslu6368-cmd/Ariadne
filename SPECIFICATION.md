# Software Requirements Specification (SRS)
## Project Name: ReactRespond (Interactive Reaction Network Editor)

### 1. Project Overview & Purpose
ReactRespond is a web-based, interactive graphical editor for constructing, modifying, and visualizing biological and chemical reaction networks (metabolic pathways, signal transduction networks, and kinetic models). It provides an intuitive canvas interface supporting SBML (with SBML Layout Extension) and Antimony formats for computational modeling workflows.

The application is engineered as a single-page application (SPA) designed for deployment on GitHub Pages.

---

### 2. Target Technology Stack
* **Language:** TypeScript / JavaScript (ES6+)
* **Rendering Engine:** HTML5 Canvas API (with support for Bézier curve rendering, PDF export engines, and theme styling hooks)
* **External Libraries:**
  * JS SBML / Antimony parser & generator library (e.g., libAntimony / JSBML web assemblies or wrappers)
  * PDF generation library (e.g., `jspdf` or `pdfkit`)
* **Local Development:** Python HTTP Server (`python -m http.server 8000`)
* **Deployment Platform:** GitHub Pages (`index.html` entry point)

---

### 3. Core Functional Requirements

#### 3.1 Interactive Canvas & Workspace
* **Pan & Zoom:** Smooth viewport translation (drag background / Shift+drag) and scaling (mouse wheel).
* **Grid Background & Auto-Layout:**
  * Toggleable grid snapping.
  * **Auto-Layout Algorithm:** Force-directed or hierarchical layout engine for models imported without spatial layout metadata.
* **Selection & Dragging:** Click-and-drag node positioning with real-time recalculation of reaction curves.

#### 3.2 Species / Node Editing
* **Node Creation:** Add species nodes with auto-incremented IDs (`S1`, `S2`) and custom names.
* **Aesthetics:** Custom colors, border thicknesses, and boundary condition styling.
* **Node Gaps:** Adjustable padding/gap around node borders so connecting Bézier curves do not touch node edges.

#### 3.3 Reaction / Edge & Centroid Mechanics
* **Arbitrary Stoichiometry:** Support N-substrates to M-products per reaction.
* **Central Reaction Point:** Each multi-reactant/product reaction features a movable central junction node from which Bézier curves branch.
* **Simplified Direct Reactions:** For simple 1-to-1 ($A \rightarrow B$) reactions, optionally use a single direct Bézier curve without a central centroid point.
* **Bézier Curves & Handles:** Straight lines or Bézier curves with interactive control handles for customized line paths.
* **Visual Customization:** Per-reaction stroke color, line thickness, and arrowhead/modifier shapes.

#### 3.4 Import & Export Capabilities
* **SBML Support:**
  * Import/Export SBML files including **SBML Layout Extension** metadata.
* **Antimony Support:**
  * Import Antimony string representations.
  * Export Antimony code (layout coordinates stored optionally within commented metadata blocks).
* **Image & Publication Export:**
  * Export high-resolution **PNG** for quick previews.
  * Export vector **PDF** for academic publication standards.

#### 3.5 Architecture & Theming Hooks
* **Theme-Ready State:** Abstract color palettes, font settings, and element styles into a central theme configuration object to support preset selection and user-defined custom stylesheets in future releases.

---

### 4. Phased Development Milestones (Testing Anchors)

* **Phase 1: MVP Canvas & Basic Topology**
  * Canvas pan/zoom, species node creation/dragging, simple straight-line directed edges, and viewport status indicators.
* **Phase 2: Bézier Curves, Centroids & Styling**
  * Central reaction points, Bézier curves with draggable handles, node-edge gap clearance, stroke colors, and line thickness controls.
* **Phase 3: Auto-Layout Engine & Advanced Canvas**
  * Automatic graph layout algorithms for unpositioned networks and multi-reactant hyper-graph routing.
* **Phase 4: SBML & Antimony Import/Export**
  * Integration of JavaScript libraries for SBML Layout Extension and Antimony import/export (with commented layout metadata).
* **Phase 5: Publication Export & Custom Theming**
  * High-resolution PDF export pipeline and user-configurable theme/style management system.