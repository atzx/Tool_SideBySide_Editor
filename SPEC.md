# SPEC — SideBySide Editor

## Overview
A single-page HTML application for composing images with overlaid text side-by-side. Users import an image on the left panel, edit text on the right panel, and export a combined high-resolution canvas as WebP/JPG/PNG or copy to clipboard.

---

## 1. Layout

### 1.1 Main Structure
- Full viewport height, column flex layout.
- **Top bar**: App title, Config GUI button with dropdown, Undo (↩), Redo (↪), dark/light mode toggle (🌙/☀️).
- **Body**: Two resizable panels side-by-side with a draggable divider. Left panel contains one or more drop zones, a Preview section, and image tools. Right panel contains a text formatting toolbar and one or more text editors.
- **Bottom bar**: IMG-Height toggle, aspect ratio selector, custom dims inputs, invert-panels button, export buttons (WebP, JPG, PNG, Copy).

### 1.2 Resizable Panels
- Left panel (image editor, with 1–2 drop zones) and right panel (text editor, with 1–2 editors).
- Draggable vertical divider between them.
- A toggle button to invert panel order (swap left/right).
- Content is dynamically rebuilt when the user selects a different layout configuration.

### 1.3 Aspect Ratio & IMG-Height
- **IMG-Height button** (default active): makes the output canvas height match the loaded image's natural height. Width is recalculated from the selected aspect ratio. When active, no aspect-ratio button shows as highlighted.
- **Button group**: 1:1, 16:9, 21:9, 4:5, Custom.
- Custom mode exposes width/height inputs (default 1920×1080).
- **Mutual exclusion**: clicking any Ratio button deactivates IMG-Height; clicking IMG-Height deactivates any Ratio button.
- **Canvas Scale**: fixed at 2× (no slider — removed). Multiplies the base resolution for high-resolution output.

### 1.4 Layout Configuration (Config GUI)
- **Config GUI button** in the top bar opens a dropdown with 4 layout presets:
  - **1 Image + 1 Text** (default): single drop zone, single text editor.
  - **1 Image + 2 Texts**: single drop zone, two text editors stacked vertically.
  - **2 Images + 1 Text**: two drop zones stacked vertically, single text editor.
  - **2 Images + 2 Texts**: two drop zones, two text editors, all stacked vertically.
- Each option shows a visual icon (◻ / ◻◻ / T / T|T) and a checkmark for the active selection.
- Switching configs preserves data in slots that remain (e.g., 2→1 images keeps the first image).
- Layout rebuilding is handled by `rebuildLayout()` → `buildDropZones()` / `buildTextEditors()`.
- Active element tracking: clicking a drop zone makes it active (red border); focusing a text editor makes it active. Toolbar controls apply to the active element.
- Active indicators ("Image 1", "Text 2") shown in the toolbar when 2+ elements exist.

### 1.5 Inline Preview
- The rendered canvas output (image left + text right) is always visible in a **Preview section** at the bottom of the left panel.
- Every modification (image, text, formatting, aspect ratio) automatically re-renders and updates the preview via `canvas.toDataURL('image/png')`.
- When **IMG-Height** is active, the preview section height dynamically matches the loaded image's displayed height, with a one-frame stabilization pass to account for layout reflow.

---

## 2. Left Panel — Image Input

### 2.1 Image Sources (per drop zone)
- **Drag & Drop**: Drag an image file onto any drop zone.
- **File picker**: "Attach" button inside each drop zone opens native file dialog.
- **Paste**: Ctrl/Cmd+V pastes an image from the clipboard into the **active** drop zone.
- When multiple drop zones exist (configs 2i1t, 2i2t), each zone manages its own image independently.

### 2.2 Image Tools (applied to active image)
- **Zoom slider**: Scale the active image within its container (10%–300%). "Set 100%" button resets zoom to 100%.
- **Rotation**: ±90° buttons (clockwise / counter-clockwise) applied to the active image.
- **Fit / Fill switch**: Toggle between `object-fit: contain` (fit) and `object-fit: cover` (fill) for the active image.
- **Active image indicator**: When 2 drop zones exist, the toolbar shows "Image 1" / "Image 2" label. Clicking a drop zone makes it active (red border highlight).
- Each image stores its own zoom, rotation, and fit/fill state independently in `state.images[]`.

### 2.3 Drop Zone UX
- Drop zones use classes (`.drop-zone`, `.drop-zone-wrapper`) instead of IDs, allowing multiple instances.
- Each drop zone has its own `<img class="image-preview">` and `.attach-btn` for file selection.
- Visual highlight when dragging over.
- Accepts: `image/*`.
- Shows the loaded image preview once dropped/pasted/selected.
- Active drop zone is highlighted with a red inset border (`.active-zone`).

### 2.4 Preview Section
- Below the drop zone, a `#textPreviewSection` container shows the rendered canvas output as an `<img>` element.
- **Header**: "Preview" label.
- **Dynamic sizing**: `updatePreviewImage()` captures the image's displayed height via `getBoundingClientRect()`, sets explicit width/height on the preview img, and adjusts the section height (img height + header height). A `requestAnimationFrame` stabilization pass re-adjusts after layout settles.
- **No gaps**: explicit dimensions match the canvas aspect ratio, `object-fit: contain` fills without letterboxing.

---

## 3. Right Panel — Text Editor

### 3.1 Text Area
- One or two `<div class="text-editor" contenteditable>` editors, created dynamically in `#textEditorsContainer` by `buildTextEditors()`.
- Each editor manages its own content, font, size, alignment, color, and letter spacing independently in `state.texts[]`.
- Text is vertically centered via JS (`adjustTextVerticalCenter`) which adjusts padding based on `scrollHeight` vs `clientHeight`. Runs on every text input, font size change, and state restore — avoids flexbox contenteditable bugs.
- Lines wrap naturally.
- Active text editor is highlighted with a red top border (`.active-editor`).

### 3.2 Formatting Toolbar (applied to active text editor)
- **Bold**, *Italic*, <u>Underline</u> — toggle buttons that apply inline styles to the active editor.
- **Alignment**: Left, Center, Right — button group.
- **Text color**: `<input type="color">` that applies `color` to the selection or entire text in the active editor.
- **Background color**: `<input type="color">` that sets the right panel's background (shared across all editors).
- **Active text indicator**: When 2 text editors exist, the toolbar shows "Text 1" / "Text 2" label. Focusing an editor makes it active.
- **Google Fonts selector**: A `<select>` populated with ~20 popular Google Fonts. The selected font applies to all text via CSS (`font-family` on the editor div), not via `execCommand('fontName')`.
- **Font size**: Range slider (12px – 120px).
- **Letter spacing**: Range slider (0px – 20px).

### 3.3 Google Fonts
- Dynamically load the selected font via a `<link>` tag injected on change. Font applied via CSS on the editor div, not via `execCommand('fontName')`, to prevent `<font>` tags from interfering with font-size inheritance.
- Default font: `Inter`.

---

## 4. Output / Export

### 4.1 Canvas Rendering
- Hidden `<canvas>` element, rendered at `baseSize × canvasScale` resolution.
- Rendering pipeline (custom engine, no external dependencies):
  1. Fill canvas with theme background color.
  2. Fill the image side — if 2 images, split the image area vertically and render each image independently with its own zoom/rotation/fit-fill.
  3. Fill the text side — if 2 texts, split the text area vertically and render each text block independently with its own font/size/alignment/color.
  4. Parse each contenteditable's innerHTML into styled text runs (bold, italic, underline, color, font family, font size, letter spacing).
  5. Draw text runs on canvas using `CanvasRenderingContext2D.fillText()` with word-wrap and alignment.
  6. Panel split ratio matches the divider position from the editing view.
  7. Panel order (normal/inverted) is preserved in the output.
  8. Each text block is vertically centered independently via two-pass: dry-run measures height, then renders at calculated Y offset.
- Google Fonts loaded via injected `<link>` tags; `document.fonts.ready` ensures fonts are available before each render.
- Stale render protection via incrementing render counter.
- **IMG-Height mode**: canvas height = first loaded image's natural height; width recalculated to maintain aspect ratio.

### 4.2 Export Buttons
- **Download WebP**: `canvas.toBlob(blob => ... , 'image/webP', 0.95)`
- **Download JPG**: `canvas.toBlob(blob => ... , 'image/jpeg', 0.95)`
- **Download PNG**: `canvas.toBlob(blob => ... , 'image/png')`
- **Copy to clipboard**: `canvas.toBlob(blob => navigator.clipboard.write([new ClipboardItem({[blob.type]: blob})]))`

---

## 5. Undo / Redo

### 5.1 State Snapshots
- On every meaningful edit (image loaded, zoom changed, rotation changed, text edited, font changed, colors changed, alignment changed, ratio changed, panels inverted, IMG-Height toggled, config changed), push the current configuration to an undo stack.
- Each snapshot is a plain object: `{ config, images[], texts[], activeImageIdx, activeTextIdx, bgColor, aspectRatio, customWidth, customHeight, panelOrder, leftPanelPct, canvasScale, imgHeightMode }`.
- `images` is an array of `{ dataUrl, zoom, rotation, fitFill, naturalW, naturalH }` (1 or 2 items depending on config).
- `texts` is an array of `{ html, fontFamily, fontSize, letterSpacing, textColor, textAlign }` (1 or 2 items depending on config).

### 5.2 Undo / Redo Stacks
- Maximum 50 entries per stack.
- `Ctrl+Z` triggers undo; `Ctrl+Shift+Z` or `Ctrl+Y` triggers redo.
- Top-bar buttons for Undo (↩) and Redo (↪).

---

## 6. Theming

### 6.1 Dark Mode (default)
- Background: `#1a1a2e`, panels: `#16213e`, text: `#e0e0e0`, inputs: `#0f3460`, accent: `#e94560`.
- All UI controls styled for dark background.

### 6.2 Light Mode
- Background: `#f5f5f5`, panels: `#ffffff`, text: `#222222`, inputs: `#e0e0e0`, accent retains color.
- Toggle button in top-bar switches between modes.
- Uses CSS custom properties (`--bg`, `--panel-bg`, `--text`, `--input-bg`, `--accent`).

### 6.3 Persistence
- Mode preference saved to `localStorage('theme')`.
- Restored on page load.

---

## 7. Technical Constraints

| Constraint | Value |
|---|---|
| File | Single `index.html` |
| CSS | Embedded `<style>` |
| JS | Embedded `<script>` (no build step) |
| Dependencies | None — vanilla JS only (no html2canvas, no jQuery) |
| Canvas text | Custom implementation using `CanvasRenderingContext2D` with Google Fonts loaded via `document.fonts.ready` |
| Browser | Modern Chromium / Firefox / Safari |

---

## 8. Post-Spec Evaluation — Additional Details

### 8.1 Canvas Text Engine
- `html2canvas` is NOT used to keep the app dependency-free.
- Instead, read the `contenteditable` div's inner HTML, parse it into runs of styled text, and draw each run on the canvas using `ctx.fillText()`.
- Parse bold, italic, underline, color spans, alignment, font family, font size, and letter spacing from the inner HTML.
- Google Fonts are loaded via injected `<link>` tags; `document.fonts.ready` ensures they are available before canvas rendering.

### 8.2 Canvas Resolution
- Base canvas dimensions: 1600px wide for preset ratios (1:1 → 1600×1600, 16:9 → 1600×900, 4:5 → 1600×2000); custom ratio uses user-provided dimensions.
- **Canvas Scale** is fixed at 2× (the slider was removed). Multiplies base dimensions for high-resolution output.
- All canvas rendering (image, text) is scaled proportionally to the final resolution.
- When **IMG-Height** is active, canvas height is set to the first loaded image's natural height; width is recalculated to maintain the selected aspect ratio.

### 8.3 Image Rendering on Canvas
- If 2 images in config, the image panel area is split vertically into equal halves via `getImageSubArea()`. Each image renders in its own sub-rectangle.
- Empty drop zones render a "No image loaded" placeholder.
- Apply rotation via `ctx.rotate()` around each sub-area center.
- `fit` vs `fill` maps to `contain` vs `cover` logic when computing destination rect for each image independently.

### 8.4 Inline Preview (multi-image aware)
- The old preview toggle mode (button + keyboard shortcut) was removed.
- Instead, a live preview section (`#textPreviewSection`) is permanently shown at the bottom of the left panel.
- The preview displays the full canvas output (all images + all text blocks) via `canvas.toDataURL('image/png')`.
- `updatePreviewImage()` uses the first loaded image's natural height for IMG-Height mode sizing.
- When multiple images exist, the preview renders all of them as they appear on the canvas.
- No gaps or deformation: explicit dimensions match canvas aspect ratio, `object-fit: contain` fills without letterboxing.
- Every modification automatically re-renders and updates the preview.

### 8.5 Clipboard Copy Fallback
- `navigator.clipboard.write()` with `ClipboardItem` is the primary path.
- Fallback: create a temporary `<img>` from the canvas data URL and prompt the user to right-click → copy.

### 8.6 Accessibility
- All toolbar buttons have `aria-label`.
- Focus indicators visible in both themes.
- Sliders have associated labels.

### 8.7 Performance
- Undo snapshots store each image as a compressed data URL (not raw pixel data) to cap memory usage. Multiple images are stored in the `images[]` array.
- Canvas renders are debounced via `requestAnimationFrame`; rapid changes produce only one render per frame.
- Preview image updates after every render via `updatePreviewImage()`.
- Layout rebuilding (`rebuildLayout()`) replaces DOM content fully — acceptable since config changes are infrequent user actions.

### 8.8 Contenteditable Compatibility (multiple editors)
- Multiple `contenteditable` editors are created dynamically with class `.text-editor` inside `#textEditorsContainer`.
- Each editor gets its own `data-index` attribute and is independently styled (font, size, alignment, color).
- `display: flex` on `contenteditable` causes a known Chromium bug where `font-size` CSS inheritance breaks. Each editor avoids flexbox entirely and uses JS-based vertical centering (`adjustTextVerticalCenter`) that dynamically adjusts padding based on `scrollHeight` vs `clientHeight`.
- Font family is applied via CSS only, not `execCommand('fontName')`, to prevent `<font face="...">` tags from interfering with font-size inheritance and cursor behavior.
- Formatting toolbar uses `execOnActive()` which applies execCommand to the currently focused text editor.

### 8.9 IMG-Height Mode
- Button in the bottom bar, positioned between the "Ratio" label and the "1:1" aspect ratio button.
- **Default active** on page load.
- When active: canvas height = loaded image's natural height; width recalculated from the selected aspect ratio.
- Forces fallback to normal aspect-ratio sizing when no image is loaded.
- **Mutual exclusion**: clicking any Ratio button (1:1, 16:9, 4:5, Custom) deactivates IMG-Height. Clicking IMG-Height deactivates any highlighted Ratio button.
- When IMG-Height is active, no aspect-ratio button shows as highlighted in the UI.
- Aspect ratio buttons use `[data-ar]` selector to exclude IMG-Height from their event handlers.
- Image loading (`loadImageToIndex`) stores natural dimensions in the image state (`naturalW`, `naturalH`) and waits for the DOM image to load before rendering.
