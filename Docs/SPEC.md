# Gesso Visual Art Specification – Version 1.0.0

**Status:** Normative / Draft  
**License:** MIT  
**Maintainer:** The Gesso Working Group

---

## 1. Introduction & Architectural Philosophy

### 1.1 The Need for a Deterministic Visual Language

Digital art today is trapped between two extremes: raster formats (PNG, JPEG) that preserve a single rendering but discard all creative intent, and proprietary layer‑based formats (PSD, .clip) that are opaque, bloated, and tied to specific software. AI image generators produce beautiful outputs but offer no way to edit the composition, brushwork, or material choices after generation.

**Gesso** is a declarative, deterministic DSL that unifies **compositional logic**, **material physics**, and **process history** into a single human‑readable text format. It acts as a lossless container for artistic intent, allowing the same source file to be rendered to a screen, printed on canvas, milled in 3D, or woven into fabric—faithfully preserving the artist’s decisions.

### 1.2 The Four Core Design Principles

1. **Intent Over Pixels** – The artist describes *what* they want (a warm golden light, a rough impasto stroke, a compositional balance) rather than micromanaging pixels. The compiler translates this into concrete render instructions.
2. **Strict Decoupling (Physics vs. Logic)** – The **material physics** (canvas texture, paint viscosity, brush shape) is declared separately from the **compositional logic** (layout of shapes, color choices, layering order). This allows the same artistic idea to be rendered with different media (oil on linen, digital print, 3D relief).
3. **Deterministic & Archival** – The same source code always yields the same visual result, regardless of rendering engine. This ensures that artwork remains reproducible centuries from now, even as display technology evolves.
4. **AI‑Native & Human‑Readable** – The syntax is concise and structured, making it easy for both humans and generative AI to write, edit, and understand. It leverages inference (sticky state, macros) to reduce redundancy by up to 90% compared to verbose formats like SVG or XML.

### 1.3 The Rendering Pipeline

A compliant compiler implements a 5‑stage transformation:

1. **Lexing** – Tokenization of the source text.
2. **Parsing** – Building an Abstract Syntax Tree (AST) with strict LL(1) lookahead.
3. **Preprocessing** – Macro expansion, variable substitution, import resolution.
4. **Inference** – Resolving sticky state (color, brush, opacity) and projecting onto an absolute coordinate system.
5. **Realization** – Translating the intent into a target medium: raster image, vector graphics, CNC machine path, or physical rendering.

---

## 2. Conformance Profiles

To accommodate different workflows, the specification defines three progressive conformance profiles.

### 2.1 The Golden Rule of Parsing

All compliant compilers **MUST** implement the complete lexer and parser for the entire grammar. Feature exclusion happens only during the Realization phase.

### 2.2 Profile A: Digital Rendering

Minimum viable implementation.  
- **Required:** Composition, color, basic brushwork, layering.  
- **Targets:** PNG, SVG, WebGL canvas.  
- **Degradation:** Unsupported physical material attributes (e.g., canvas texture, varnish) are silently ignored.

### 2.3 Profile B: Physical Output

For high‑fidelity printing, CNC routing, or textile production.  
- **Required:** Profile A plus color management (ICC profiles), resolution, material mapping (canvas, paper, wood).  
- **Targets:** CMYK TIFF, PDF/X, G‑code (CNC), vector cut files.

### 2.4 Profile C: Full Physical Simulation

For advanced rendering engines that simulate light, brush physics, and material aging.  
- **Required:** Profile B plus light simulation, 3D geometry, aging effects.  
- **Targets:** Physically based renderers (e.g., Blender Cycles), VR scenes, robotic painting arms.

---

## 3. Lexical Structure & Tokenization

### 3.1 Character Set & Encoding

- **Encoding:** UTF‑8.
- **Case Sensitivity:** Keywords and built‑in identifiers are case‑insensitive; user‑defined identifiers are case‑sensitive.
- **Comments:** Denoted by `%%` (line comment). Block comments are not allowed.

### 3.2 Operators & Compound Sigils

| Sigil | Name | Purpose |
|-------|------|---------|
| `{ }` | Structural braces | Define scopes (artwork, layer, macro). |
| `@{ }` | Map sigil | Key‑value dictionaries for attributes (color, material). |
| `[ ]` | Array brackets | List of values or geometric coordinates. |
| `:` | Assignment | Bind key to value (e.g., `color: #FF0000`). |
| `.` | Attribute dot | Chain modifiers (e.g., `.rough .glaze`). |
| `$` | Invocation dollar | Reference macros or variables. |

### 3.3 Literals & Domain Primitives

| Type | Pattern | Example |
|------|---------|---------|
| Integer | `[0-9]+` | `1200` |
| Float | `[0-9]+\.[0-9]+` | `0.75` |
| String | `"[^"]*"` | `"Starry Night"` |
| HexColor | `#[0-9a-fA-F]{6}` | `#1E88E5` |
| Angle | `[0-9]+deg` | `45deg` |
| Length | `[0-9]+(px\|mm\|in\|pt)` | `300px`, `10mm` |
| Percent | `[0-9]+%` | `50%` |
| Identifier | `[a-zA-Z_][a-zA-Z0-9_]*` | `background`, `sky` |



## 4. Document Structure & The Absolute Canvas

A well‑formed Gesso document represents a fully encapsulated unit of artistic intent, enforcing a **declaration‑before‑use** policy.

### 4.1 The Root Block & Versioning

The outermost scope **MUST** be enclosed in a `gesso` block declaring the specification version.

```ebnf
Artwork ::= "gesso" String? "{" TopLevel* "}"
```

**Example:**
```
gesso "1.0" {
  %% Internal scopes
}
```

### 4.2 Phase 1: Global Configuration (`meta`)

Establishes the global constants of the physical or digital space using the Map Sigil (`@{}`).

**Syntax:** `meta @{ key: value, ... }`

| Key | Expected Type | Description |
|-----|---------------|-------------|
| `title` | String | Work title. |
| `artist` | String | Creator name. |
| `description` | String | Optional notes. |
| `format` | String | Canvas aspect ratio or dimensions (e.g., `"16:9"`, `"square"`, `"1200x1600px"`). |
| `resolution` | Length | Base resolution for digital renders (e.g., `300px`). |
| `color_profile` | String | ICC profile name (e.g., `"sRGB"`, `"AdobeRGB"`, `"DisplayP3"`). |
| `background` | Color | Global background color or gradient. |
| `units` | String | Default unit for lengths (`px`, `mm`, `in`, `pt`). |

**Example:**
```
meta @{
  title: "Nocturne in Blue",
  artist: "A. Painter",
  format: "1200x1600px",
  color_profile: "DisplayP3",
  background: #0A1928,
  units: px
}
```

### 4.3 Phase 2: Definitions (`def`)

Registers physical materials, brushes, and reusable elements in the Global Symbol Table. An identifier **MUST** be defined before it can be referenced.

**Syntax:** `def [ID] [Label] [Attributes]`

- **ID:** Alphanumeric identifier unique within the namespace.
- **Label:** String literal for human reference.
- **Attributes:** Space‑separated `key=value` pairs; complex configurations use Map Sigil `@{}`.

---

## 5. Material Definitions (The Physics)

Separating material physics from compositional logic ensures the same artistic intent can be rendered across different media.

### 5.1 Surface (`style=surface`)

Defines the substrate (canvas, paper, wood, digital).

**Attributes:**
- `type`: `canvas`, `paper`, `wood`, `metal`, `digital`.
- `texture`: `smooth`, `rough`, `linen`, `watercolor`.
- `color`: Base color (Hex or named).
- `size`: Width x height (e.g., `[1200px, 1600px]`).
- `primed`: Boolean (e.g., `true` for gesso).

**Example:**
```
def canvas "Linen" style=surface @{
  type: canvas,
  texture: linen,
  color: #F5F0E0,
  size: [1200px, 1600px],
  primed: true
}
```

### 5.2 Brush (`style=brush`)

Defines a physical or digital brush.

**Attributes:**
- `type`: `round`, `flat`, `filbert`, `fan`, `digital`.
- `size`: Diameter in pixels or millimeters (e.g., `12px`).
- `stiffness`: `0.0` (soft) to `1.0` (stiff).
- `texture`: `smooth`, `bristle`, `sponge`.
- `opacity`: Initial opacity (`0%` to `100%`).

**Example:**
```
def brush_detail "Riggers" style=brush @{
  type: round,
  size: 4px,
  stiffness: 0.8,
  texture: bristle,
  opacity: 100%
}
```

### 5.3 Paint (`style=paint`)

Defines color with physical properties.

**Attributes:**
- `color`: Base color (Hex, RGB, or named).
- `opacity`: `0%` to `100%`.
- `medium`: `oil`, `acrylic`, `watercolor`, `digital`.
- `gloss`: `matte`, `satin`, `glossy`.
- `transparency`: `0%` (opaque) to `100%` (transparent).
- `flow`: Viscosity factor (`0.0` to `1.0`).

**Example:**
```
def cerulean "Cerulean Blue" style=paint @{
  color: #2A6F8F,
  medium: oil,
  gloss: satin,
  opacity: 100%,
  flow: 0.6
}
```

### 5.4 Palette (Collection)

Groups paints for easy reference.

**Syntax:**
```
palette "Name" = @{ ID1, ID2, ... }
```

**Example:**
```
palette "Seascape" = @{ cerulean, white, naples_yellow, burnt_umber }
```

---

## 6. Composition Primitives (The Logic)

Artistic composition is built from **shapes**, **paths**, **text**, and **image references**. Each primitive has geometric attributes and can carry modifiers.

### 6.1 Shapes

Built‑in primitives: `rect`, `circle`, `ellipse`, `triangle`, `line`.

**Syntax:** `[Shape] [ID?] [Attributes]`

**Attributes:**
- `pos`: `[x, y]` – anchor point (default `[0,0]`).
- `size`: `[w, h]` (for rect/ellipse).
- `radius`: `r` (for circle).
- `points`: `[[x1,y1], [x2,y2], ...]` (for polygon).
- `angle`: rotation in degrees.
- `fill`: Color or reference to paint.
- `stroke`: Color or reference to paint.
- `stroke_width`: Length.

**Example:**
```
rect "sun" @{ pos: [600, 400], size: [200, 200], fill: @{ color: #FFD966, gloss: glossy }, stroke: burnt_umber, stroke_width: 4px }
circle @{ pos: [300, 500], radius: 50, fill: @{ color: #FF6B6B } }
```

### 6.2 Paths

Complex curves defined by cubic Bézier curves.

**Syntax:** `path [ID?] @{ points: [P0, C1, C2, P1, ...] }`

Each segment is `[x0, y0, cx1, cy1, cx2, cy2, x1, y1]`.

**Example:**
```
path "wave" @{ 
  points: [
    [100, 500, 150, 450, 250, 550, 300, 500],
    [300, 500, 350, 450, 450, 550, 500, 500]
  ],
  stroke: cerulean,
  stroke_width: 3px,
  fill: none
}
```

### 6.3 Text

Adds typography.

**Attributes:**
- `text`: String.
- `font`: Font family.
- `size`: Length.
- `align`: `left`, `center`, `right`.
- `fill`: Color.

**Example:**
```
text @{ text: "Dawn", pos: [600, 1400], font: "Georgia", size: 48px, fill: #FFFFFF }
```

### 6.4 Image Reference

Embed external raster images (URI).

**Syntax:** `image [ID?] @{ src: "uri", pos: [x,y], size: [w,h] }`

**Example:**
```
image "reference" @{ src: "./sketch.jpg", pos: [100, 100], size: [400, 300] }
```

---

## 7. Color & Lighting

### 7.1 Color Primitives

Colors can be defined as:
- **Hex:** `#RRGGBB` (e.g., `#FF6600`).
- **RGB:** `rgb(255, 102, 0)`.
- **Named:** Standard CSS color names.
- **Reference:** `$palette_id` or `$paint_id`.

### 7.2 Gradients

Linear or radial gradients.

**Syntax:** `gradient(type) = @{ stops: [[offset, color], ...], angle: deg, pos: [x,y] }`

**Example:**
```
gradient(linear) @{
  stops: [
    [0.0, #FFD966],
    [0.5, #FFAA33],
    [1.0, #FF6B6B]
  ],
  angle: 90deg
}
```

### 7.3 Lighting (Optional)

For 3D renders or simulation.

**Attributes:**
- `ambient`: Intensity `0.0` to `1.0`.
- `directional`: `[angle, intensity]`.
- `point`: `[x, y, intensity]`.

**Example:**
```
light @{
  ambient: 0.3,
  directional: [45deg, 0.7]
}
```

# Addendum A: Ecosystem Integrations (The Zero‑Friction Runtime)

**Version:** 1.0 (Extension to Gesso 1.0.0)  
**Status:** Normative / Final  
**Scope:** WebAssembly Compilation, WebGL Rendering, HTML Custom Elements, and AI Generation.

---

## A.1 Architectural Philosophy

Gesso is designed to be the universal logic layer for visual art. To achieve zero‑friction adoption, the language **MUST** be compilable to WebAssembly (Wasm) and executable directly in the browser, with no external dependencies. This addendum defines the **Embedded Web Runtime**, enabling any web developer to embed, generate, and render Gesso artwork natively.

---

## A.2 The WebAssembly Compiler Target (`--target wasm`)

The core Rust `gesso` compiler **SHALL** support a `wasm32-unknown-unknown` build target, allowing lexing, parsing, and IR unrolling to execute client‑side. This enables:
- **Token efficiency:** Gesso files are typically small (kilobytes), fetching and rendering faster than streaming large raster images.
- **Serverless rendering:** No backend required; all rendering happens in the user’s browser.

---

## A.3 The Embedded WebGL Renderer

When the `gesso` runtime is invoked in a browser environment, it **SHALL** default to a WebGL 2.0 renderer that maps the IR to canvas primitives.

- **`style=surface`:** Creates a WebGL texture representing the canvas.
- **`style=brush`:** Translates brush attributes to WebGL draw calls with appropriate blending.
- **`style=paint`:** Converts color, opacity, and gloss to shader uniforms.
- **Paths & Shapes:** Rendered via WebGL line and triangle primitives with Bézier curve tessellation.

If WebGL 2.0 is unavailable, the runtime **SHALL** fall back to a Canvas 2D renderer, with a user‑visible warning.

---

## A.4 The `<gesso-canvas>` HTML Custom Element

To maximize ease of adoption, the Web Runtime **SHOULD** expose an HTML Custom Element (Web Component).

**Syntax:**
```html
<gesso-canvas src="./artwork.gesso" width="800" height="600" controls autoplay>
  Your browser does not support the Gesso Web Runtime.
</gesso-canvas>
```

**Attributes:**
- `src`: URL of a `.gesso` file.
- `width`, `height`: Dimensions of the canvas.
- `controls`: Show interactive controls (zoom, pan, export).
- `autoplay`: Automatically render on load.

The element encapsulates the compiler, renderer, and user interaction into a single tag, allowing artists to embed interactive, editable artworks on any webpage.

---

## A.5 Local AI Generation (WebGPU)

When a Gesso script references an AI generative plugin (e.g., `src="plugin://ai-diffusion"`), the Web Runtime **MAY** intercept this URI and use WebGPU/ONNX.js to execute lightweight, quantized generative models directly in the browser. This enables:
- Real‑time style transfer.
- In‑painting and out‑painting.
- Interactive co‑creation with AI.

All AI execution remains client‑side, preserving user privacy and eliminating server costs.

---

# Addendum B: The REPL Architecture & Sketch Mode

**Version:** 1.0 (Extension to Gesso 1.0.0)  
**Status:** Normative / Final  
**Scope:** Auto‑Scaffolding, Continuous State Persistence, and the REPL Execution Wrapper.

---

## B.1 Architectural Philosophy

The full Gesso specification mandates a strict declaration‑before‑use hierarchy (`meta` → `def` → layers). While this guarantees archival safety, it introduces friction for rapid sketching. **Addendum B** defines a **Sketch Mode** that automatically scaffolds raw artistic input into a valid Gesso AST, enabling zero‑friction exploration.

---

## B.2 The Auto‑Scaffolding Protocol

When the `gesso` compiler is invoked with the `--sketch` flag, it **MUST** wrap raw input in the following boilerplate:

```
gesso "1.0" {
  meta @{
    title: "Gesso Sketch",
    format: "1200x800px",
    auto_pad_layers: true
  }
  def canvas "Canvas" style=surface @{
    type: digital,
    size: [1200px, 800px],
    color: #F5F0E0
  }
  def default_brush "Sketch Brush" style=brush @{
    type: round,
    size: 8px,
    stiffness: 0.5
  }
  def default_paint "Sketch Paint" style=paint @{
    color: #000000,
    medium: digital
  }
  layer "Sketch" {
    [USER_INPUT]
  }
}
```

- If the user specifies a color or brush ID that is not defined, the wrapper **SHALL** automatically generate a default definition.
- If the user uses multiple distinct identifiers (e.g., `sky` and `mountains`), the wrapper **MAY** create separate layers accordingly.

---

## B.3 Continuous State Persistence (The REPL Cursor)

In a REPL environment (e.g., web console, live coding tool), the compiler state **MUST** persist across executions.

- **Sticky State:** The last used brush, paint, stroke width, and opacity are retained and injected into the next input.
- **Absolute Timeline:** Sequential inputs are appended to the cumulative artwork, not overwritten.

**Example:**
```
> circle radius: 50, fill: #FF0000
> circle pos: [100,100], radius: 30, fill: #00FF00   // second circle added
```

---

## B.4 Ejection (The Export Protocol)

The REPL **MUST** support an `eject` command that materializes the hidden scaffolding and outputs a fully formed, valid Gesso source file.

**Behavior:** The compiler collects all accumulated layers, resolves auto‑generated definitions, and writes a complete `.gesso` file, ready for archival or further refinement.

---

# Addendum C: Generative Ergonomics & Smart Composition

**Version:** 1.0 (Extension to Gesso 1.0.0)  
**Status:** Normative / Final  
**Scope:** Auto‑Padding, Decoupled Control Lanes, and Relative Positioning Heuristics.

---

## C.1 Auto‑Padding of Layers (E3002 Mitigation)

Gesso enforces that each layer must have a defined size. If a layer’s content does not cover the full canvas, the compiler **SHALL** automatically pad it with transparent background when `auto_pad_layers` is enabled.

**Syntax:** `meta @{ auto_pad_layers: true }`

**Execution:** The compiler calculates the bounding box of all primitives in the layer and extends the layer to the canvas dimensions, filling uncovered areas with transparency.

---

## C.2 Decoupled Control Lanes (The Animation Track)

For animation or dynamic art, control data (e.g., brush size over time) can be decoupled from the visual content.

**Syntax:**
```
layer "Animation" {
  control: brush_size = [8px, 20px, 8px], time = [0s, 2s, 4s], curve = "linear"
  rect @{ size: [100,100], pos: [100,100] }   // brush size varies over the animation timeline
}
```

The compiler attaches the control curve to the primitives in the layer, allowing keyframed interpolation without cluttering the visual description.

---

## C.3 Relative Positioning Heuristics (`style=relative`)

When placing elements, absolute coordinates can be verbose. **Relative positioning** allows placement relative to previously placed objects.

**Syntax:** `def canvas style=relative`

**Behavior:** If a primitive’s `pos` attribute is omitted, the compiler places it to the right of the previous primitive (or below if `newline` is implied). This is ideal for generating sequences of shapes or text.

**Example:**
```
circle radius: 50, fill: red
circle radius: 50, fill: blue   // placed to the right of the first
```

---

# Addendum D: Deterministic Semantic Decompilation

**Version:** 1.0 (Extension to Gesso 1.0.0)  
**Status:** Normative / Final  
**Scope:** Reverse‑engineering intent from raster images, vector graphics, and legacy formats.

---

## D.1 Philosophy of Reverse Inference

Gesso is designed to be the archival format for art. **Addendum D** defines how to decompile existing raster images (PNG, JPEG) or vector graphics (SVG, PDF) back into Gesso source, using algorithmic pattern detection to infer brush strokes, color layers, and compositional intent.

---

## D.2 Decompilation Pipeline

1. **Segmentation:** The image is segmented into color regions and edges using computer vision algorithms (e.g., k‑means, edge detection).
2. **Stroke Inference:** Connected regions are analyzed for brush‑like texture, orientation, and opacity to infer brush type and stroke direction.
3. **Layer Separation:** Regions are grouped by color similarity and blending mode to reconstruct layers.
4. **Macro Extraction:** Repeated patterns (e.g., a row of circles) are compressed into macros.
5. **Gesso Generation:** The inferred data is serialized to Gesso syntax.

---

## D.3 Normative Output Guarantee

The decompiler **SHOULD** output Gesso that, when re‑rendered, is visually indistinguishable from the original. While the exact intent may not be perfectly recovered, the result must be a valid, editable Gesso document.

---

# Addendum E: Target Bounds & Fallback Rendering (The Demarcation Pass)

**Version:** 1.0 (Extension to Gesso 1.0.0)  
**Status:** Normative / Final  
**Scope:** AST Pruning, Unprintable Physics, and Target‑Specific Compilation Directives.

---

## E.1 Architectural Philosophy

Gesso unifies physical painting attributes (texture, gloss, impasto) with digital rendering. When targeting a medium that cannot support certain attributes (e.g., printing to paper cannot simulate impasto), the compiler **MUST** execute a Demarcation Pass to either prune, approximate, or error.

---

## E.2 Attribute‑Level AST Pruning

| Attribute | Action (when unsupported) |
|-----------|---------------------------|
| `texture: impasto` | Pruned (ignored) for 2D print; retained for 3D rendering. |
| `gloss: glossy` | Mapped to simulated sheen in PDF; ignored in grayscale. |
| `brush texture` | Pruned if target is vector (SVG). |
| `light` | Pruned for raster output; retained for 3D. |

---

## E.3 Explicit Compilation Directives

Users can control the Demarcation Pass using `@target` directives.

**Syntax:**
```
@target "print" {
  %% Override: convert impasto to bump map for high‑end printing
  impasto -> bump_map
}
```

---

# Addendum F: The Signal Routing & Material Matrix

**Version:** 1.0 (Extension to Gesso 1.0.0)  
**Status:** Normative / Final  
**Scope:** Internal Bus Routing, Material Mixing, and Layered Execution.

---

## F.1 Internal Bus Protocol (`bus://`)

Gesso supports real‑time audio‑like routing for visual elements.

- `src="bus://canvas"`: Captures the final rendered image.
- `src="bus://layer_name"`: Captures the output of a specific layer.

This enables **live feedback effects** (e.g., applying a filter to the entire composition) without duplicating layers.

---

## F.2 Material Mixing

Paints can be mixed using arithmetic.

**Syntax:** `paint: mix(cerulean, white, 0.6)`

**Execution:** The compiler computes the resulting color and physical properties (gloss, opacity) based on the blend ratio.

---

# Addendum G: The Projectional DAW (Bi‑Directional Visual Editor)

**Version:** 1.0 (Extension to Gesso 1.0.0)  
**Status:** Normative / Final  
**Scope:** Projectional Editing, WebGL Timeline Rendering, and Bi‑Directional AST Mutation.

---

## G.1 Architectural Philosophy

Traditional art software ties GUI interactions to a proprietary binary state. Gesso **projectional editing** inverts this: the GUI is a live projection of the Gesso source code. Any interaction (resize, move, recolor) directly edits the source text, which then re‑compiles and re‑renders.

---

## G.2 Mutation Algorithms

- **Resize:** Dragging a shape’s edge updates its `size` attribute.
- **Move:** Dragging updates its `pos` attribute.
- **Recolor:** Picking a color updates the `fill` or `stroke` attribute.
- **Reorder:** Dragging layers in the UI reorders the `layer` blocks in the source.

---

## G.3 Lexical Lock (The AST Guard)

If a user attempts to mutate an element generated by a macro, the GUI **MUST** display a lock icon and disallow editing, preventing ambiguous changes.

---

# Addendum H: Codebase Indexing & AI‑Assisted Orchestration (The RAG Blueprint)

**Version:** 1.0 (Extension to Gesso 1.0.0)  
**Status:** Normative / Final  
**Scope:** AST‑Aware Semantic Chunking, Multi‑Dimensional Metadata Tagging, and Retrieval‑Augmented Generation (RAG).

---

## H.1 Architectural Philosophy

As Gesso implementations scale, the codebase expands beyond the limits of standard LLM context windows. This addendum mandates a RAG pipeline for AI‑assisted development, ensuring that the “narrow waist” of the compiler remains accessible.

---

## H.2 Semantic Chunking & Vectorization

- Use **Tree‑sitter** to parse Rust, TypeScript, and Gesso source files.
- Chunk by logical unit: `struct`, `impl`, `function`, `macro`.
- Embed each chunk using high‑dimensional models (e.g., `text‑embedding‑004`).

---

## H.3 Domain‑Specific Metadata Tagging

| Metadata Key | Values | Purpose |
|--------------|--------|---------|
| `domain` | `compiler`, `renderer`, `ui`, `infra` | Filter search space. |
| `language` | `rust`, `typescript`, `gesso` | Syntax‑specific retrieval. |
| `criticality` | `high`, `medium`, `low` | Prioritize core logic. |

---

## H.4 The Retrieval Pipeline

1. **Query Expansion:** Natural language prompt → technical query.
2. **Filtered Vector Search:** Search only `domain:compiler` for code changes.
3. **Context Injection:** Top K chunks injected into LLM prompt with file paths.

---

## H.5 Hybrid Context Management

- **Exploration:** Use RAG for broad queries.
- **Refactoring:** Load full `domain:compiler` files (1M+ tokens) for surgical changes.

---

This completes the Gesso Visual Art Specification with its foundational addenda. The language is now ready for implementation and adoption as a deterministic, AI‑native, human‑readable format for capturing artistic intent.
