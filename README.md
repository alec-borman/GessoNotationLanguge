# Gesso – The System of Record for Visual Art

**Status:** Specification Draft (v1.0.0)  
**License:** MIT  
**Maintainer:** The Gesso Working Group

---

Gesso is a deterministic, declarative domain‑specific language for capturing artistic intent in painting, digital art, and physical fabrication. It separates **material physics** (canvas, brush, paint) from **compositional logic** (shapes, layers, color), allowing the same source to render faithfully to screen, print, 3D relief, or CNC.

Like its musical counterpart Tenuto, Gesso is designed to be:

- **Human‑readable & AI‑native** – Concise syntax with sticky state and macros.
- **Deterministic & archival** – The same source always yields the same result, preserving intent for centuries.
- **Lossless** – Captures not just pixels, but the *why* behind every stroke.

---

## Repository Contents

This repository contains the **normative specification** and a **reference prototype compiler**. The full Gesso Studio (IDE, renderer, AI tools) lives in a [separate repository](https://github.com/your-org/gesso-studio).

- **`/spec`** – The complete language specification (in Markdown).
- **`/prototype`** – A minimal Rust/Wasm compiler demonstrating the core pipeline.
- **`/examples`** – Example Gesso artworks.

---

## Quick Example

```gesso
gesso "1.0" {
  meta @{ title: "Evening Glow", format: "1200x1600px" }

  def canvas style=surface @{ type: canvas, texture: linen, color: #F5F0E0 }
  def brush_soft style=brush @{ type: round, size: 20px, stiffness: 0.3 }
  def cerulean style=paint @{ color: #2A6F8F, medium: oil }

  layer "Sky" {
    rect @{ size: [1200, 800], fill: #0A1928 }
    gradient(radial) @{ stops: [[0.0, #1E3A8A], [1.0, #0A1928]] }
  }

  layer "Mountains" {
    brush: brush_soft
    paint: cerulean
    path @{ points: [[100,800, 150,720, 250,760, 300,800]] }.impasto
  }
}
