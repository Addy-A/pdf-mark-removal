# Pdfium & Pdfium-Render Crate Research

Research into whether the `pdfium-render` and/or `pdfium` Rust crates are
suitable for a new PDF-to-image export tool (JPG / PNG / WebP output based on
user args).

---

## Project Goal

Build a CLI tool that:

1. Accepts a PDF file as input.
2. Renders each page to a raster image.
3. Exports the result in a user-chosen format via flags (`-jpg`, `-png`,
   `-webp`).

The tool would sit alongside the existing `pdf-mark-removal` project as part of
a broader "PDF manipulation" series and should leverage a crate that provides
high-quality rendering without requiring the user to write low-level PDF
parsing code.

---

## Crates Evaluated

### 1. `pdfium-render`

| Attribute | Detail |
|---|---|
| **Crate** | [`pdfium-render`](https://crates.io/crates/pdfium-render) |
| **Source** | <https://github.com/ajrcarey/pdfium-render> |
| **Docs** | <https://docs.rs/pdfium-render/latest/pdfium_render/> |
| **Latest version** | 0.8.37 (Nov 2025) |
| **Downloads** | ~66,000 / month |
| **Used by** | 36 crates (29 direct) |
| **License** | MIT / Apache-2.0 |
| **Maintenance** | Actively maintained, frequent releases throughout 2024-2025 |

#### What it does

`pdfium-render` is a high-level, idiomatic Rust wrapper around Google's
[PDFium](https://pdfium.googlesource.com/pdfium/) C++ library — the same engine
that powers Chrome's built-in PDF viewer. It exposes rendering, editing,
creation, form-filling, annotation, text/image extraction, and more through a
safe Rust API.

#### Key capabilities relevant to the project

- **PDF → image rendering.** Render whole pages or regions to bitmaps, then
  convert to `image::DynamicImage` for saving in any format supported by the
  `image` crate (JPG, PNG, WebP, etc.).
- **Configurable render settings.** Target width/height, DPI, rotation, and
  transparency handling via `PdfRenderConfig`.
- **`image` crate integration.** The rendered bitmap converts directly to a
  `DynamicImage`, so saving as JPG/PNG/WebP is a single
  `save_with_format(...)` call.
- **Extensive example suite.** The repo ships with commented examples covering
  the exact use case we need (page-to-JPEG export), plus advanced scenarios.
- **WASM support.** If the project ever targets the browser, `pdfium-render`
  can compile to WebAssembly.

#### Example: render every page to JPEG

```rust
use pdfium_render::prelude::*;

fn export_pdf_to_jpegs(path: &impl AsRef<Path>, password: Option<&str>) -> Result<(), PdfiumError> {
    let pdfium = Pdfium::default();
    let document = pdfium.load_pdf_from_file(path, password)?;

    let render_config = PdfRenderConfig::new()
        .set_target_width(2000)
        .set_maximum_height(2000)
        .rotate_if_landscape(PdfPageRenderRotation::Degrees90, true);

    for (index, page) in document.pages().iter().enumerate() {
        page.render_with_config(&render_config)?
            .as_image()?
            .into_rgb8()
            .save_with_format(
                format!("page-{}.jpg", index),
                image::ImageFormat::Jpeg,
            )
            .map_err(|_| PdfiumError::ImageError)?;
    }
    Ok(())
}
```

Swapping `ImageFormat::Jpeg` for `ImageFormat::Png` or `ImageFormat::WebP`
(with `image`'s `webp` feature enabled) is the only change needed to support
all three output formats.

#### Pros

- High-level API — minimal boilerplate to go from PDF to image file.
- `image` crate integration means JPG/PNG/WebP are trivially supported.
- Actively maintained with ~66k downloads/month; mature and battle-tested.
- Extensive documentation and examples.
- WASM-capable for future flexibility.

#### Cons

- **External binary dependency.** Requires the Pdfium shared library
  (`libpdfium.so` / `pdfium.dll` / `libpdfium.dylib`) at runtime.
  Pre-built binaries are available from
  [bblanchon/pdfium-binaries](https://github.com/bblanchon/pdfium-binaries),
  but the user must download and place them manually (or the project must
  bundle them).
- Thread safety is continually improving but may need care in concurrent
  scenarios.
- Dynamic linking adds a deployment consideration (library path setup).

---

### 2. `pdfium` (by newinnovations)

| Attribute | Detail |
|---|---|
| **Crate** | [`pdfium`](https://crates.io/crates/pdfium) |
| **Source** | <https://github.com/newinnovations/pdfium-rs> |
| **Docs** | <https://docs.rs/pdfium/latest/pdfium/> |
| **Latest version** | 0.10.3 (Feb 2026) |
| **Downloads** | ~433 / month |
| **Used by** | A handful of projects (e.g. MView6 viewer) |
| **License** | MIT / Apache-2.0 |
| **Maintenance** | Active (50 releases), though narrower adoption |

#### What it does

A modern, thread-safe Rust interface to the PDFium C library. It focuses on
ergonomics — no complex lifetime annotations — and gives direct access to the
full 440+ function PDFium C API alongside safe high-level wrappers.

#### Key capabilities relevant to the project

- Full rendering via PDFium (same engine as `pdfium-render`).
- Thread-safe static library access (`parking_lot::ReentrantMutex`).
- Renderer selection between Skia and AGG backends.
- No lifetime annotations on core types — simpler to integrate into GUI or
  long-lived application state.

#### Pros

- Thread-safe by design — cleaner for multi-threaded / interactive apps.
- No complex lifetime plumbing.
- Full low-level C API accessible when needed.

#### Cons

- **Much lower adoption** (~433 downloads/month vs ~66k for `pdfium-render`).
- Fewer examples and less community support.
- Still requires the external PDFium binary.
- Less documentation around image-export workflows specifically.

---

## Side-by-Side Comparison

| Criterion | `pdfium-render` | `pdfium` |
|---|---|---|
| **Rendering quality** | High (Chrome's engine) | High (same engine) |
| **API level** | High-level, idiomatic Rust | Mid-level, C API exposed |
| **`image` crate integration** | Built-in (`as_image()` → `DynamicImage`) | Manual conversion required |
| **JPG / PNG / WebP export** | Trivial via `image::ImageFormat` | Possible but more manual |
| **Downloads / month** | ~66,000 | ~433 |
| **Example coverage** | Extensive (rendering, editing, WASM) | Decent but narrower |
| **Thread safety** | Improving; needs care | Robust out of the box |
| **External binary** | Required (Pdfium `.so`/`.dll`) | Required (Pdfium `.so`/`.dll`) |
| **WASM support** | Yes | Not emphasized |
| **Lifetime complexity** | Low (simplified over time) | Very low (no lifetimes) |

---

## How `lopdf` Fits In

The existing `pdf-mark-removal` project uses
[`lopdf`](https://crates.io/crates/lopdf) for low-level PDF manipulation
(parsing objects, editing content streams, modifying page dictionaries). `lopdf`
**cannot render** PDFs — it has no concept of turning pages into images. For
the new image-export tool, `lopdf` alone is not an option; a rendering crate
is needed.

However, `lopdf` and `pdfium-render` serve complementary purposes:

| Concern | `lopdf` | `pdfium-render` |
|---|---|---|
| Parse / edit PDF structure | ✅ | ✅ (but higher-level) |
| Render pages to images | ❌ | ✅ |
| Pure Rust (no C deps) | ✅ | ❌ (needs Pdfium binary) |

The new project would use `pdfium-render` (or `pdfium`) for rendering, **not**
as a replacement for `lopdf` in the existing project.

---

## WebP Support Details

WebP export is **not** a feature of the Pdfium crates themselves — it comes
from the Rust [`image`](https://crates.io/crates/image) crate. Since
`pdfium-render` outputs `image::DynamicImage`, you can save in any format
`image` supports. To enable WebP encoding, add the `webp` feature:

```toml
[dependencies]
pdfium-render = "0.8"
image = { version = "0.25", features = ["webp"] }
```

Then save with `ImageFormat::WebP`:

```rust
img.save_with_format("page-0.webp", image::ImageFormat::WebP)
    .map_err(|_| PdfiumError::ImageError)?;
```

No extra crates or manual encoding are needed.

---

## Recommendation

**Use `pdfium-render`** for the new PDF-to-image export project.

### Rationale

1. **Direct fit for the use case.** The project needs to render PDF pages to
   JPG/PNG/WebP. `pdfium-render` does exactly this with minimal code — load
   PDF, configure render, call `as_image()`, save.

2. **`image` crate integration.** Switching between output formats is a single
   enum variant change (`ImageFormat::Jpeg` / `Png` / `WebP`), which maps
   cleanly to the planned CLI flags.

3. **Community & stability.** ~66k downloads/month, active maintenance, and
   extensive examples reduce risk and speed up development.

4. **Lower barrier to entry.** The high-level API means less boilerplate
   compared to the `pdfium` crate's more direct C-API approach.

5. **Future flexibility.** WASM support opens the door to a browser-based
   version if desired later.

### Trade-off to accept

The external Pdfium binary dependency is the main downside. This can be
mitigated by:

- Documenting the setup steps clearly in the new project's README.
- Pointing users to the
  [bblanchon/pdfium-binaries](https://github.com/bblanchon/pdfium-binaries)
  pre-built releases.
- Optionally bundling the binary in CI artifacts.

### Suggested `Cargo.toml` for the new project

```toml
[package]
name = "pdf-to-image"
version = "0.1.0"
edition = "2024"

[dependencies]
pdfium-render = "0.8"
image = { version = "0.25", features = ["jpeg", "png", "webp"] }
clap = { version = "4", features = ["derive"] }
```

`clap` is included for ergonomic CLI argument parsing (`-jpg`, `-png`,
`-webp` flags).

---

## References

- [pdfium-render — GitHub](https://github.com/ajrcarey/pdfium-render)
- [pdfium-render — crates.io](https://crates.io/crates/pdfium-render)
- [pdfium-render — docs.rs](https://docs.rs/pdfium-render/latest/pdfium_render/)
- [pdfium — GitHub](https://github.com/newinnovations/pdfium-rs)
- [pdfium — crates.io](https://crates.io/crates/pdfium)
- [pdfium — docs.rs](https://docs.rs/pdfium/latest/pdfium/)
- [pdfium-binaries (pre-built)](https://github.com/bblanchon/pdfium-binaries)
- [image crate — crates.io](https://crates.io/crates/image)
- [lopdf — crates.io](https://crates.io/crates/lopdf)
