# Keychron Launcher Prototype - Session Notes

## Last Updated: Backlight Page Major Overhaul (2026-02-14)

---

## Current Progress

### 5-Phase Fine-Tuning Plan

| Phase | Status | Description |
|-------|--------|-------------|
| **Phase 1** | ✅ Done | i18n completion — settings, feedback, he-mode |
| **Phase 2** | ✅ Done | i18n completion — backlight effect names + advance page |
| **Phase 3** | ✅ Done | Inline styles → CSS classes — settings + feedback |
| **Phase 4** | ⬜ Pending | Inline styles → CSS classes — backlight + keymap |
| **Phase 5** | ⬜ Pending | Move connect `<style>` block + advance inline cleanup |

### Additional Completed Work (Outside Plan)

| Task | Status | Description |
|------|--------|-------------|
| **Assist i18n** | ✅ Done | Quick Start page setup guide + status text i18n |
| **Macro Button** | ✅ Done | Submit button width:100% → min-width:160px, right-aligned |
| **Macro List Editor** | ✅ Done | Replaced chip timeline → list-based editor (referencing real Keychron Launcher) |
| **Nav Gradient Effect** | ✅ Done | Animated gradient hover/active effect on sidebar nav items (referencing real launcher) |
| **Backlight Overhaul** | ✅ Done | 23 effects, dynamic keyboard animations, HSV picker, Per-key/Mix/Indicator tabs |
| **CSS Cache Busting** | ✅ Done | Added cache-busting script to index.html for development |
| **Toggle Markup Fix** | ✅ Done | Fixed toggle-slider → toggle-track + toggle-thumb in backlight.html |

---

## Completed Work History

### Backlight Page Major Overhaul (2026-02-14)

**Background:**
- Analyzed real Keychron Launcher (`launcher.keychron.com/#/light`) in detail
- Real launcher has 23 effects (0-22), HSV color picker, Per-key RGB, Mix RGB, Indicator Light tabs
- Our prototype only had 13 effects, no color picker, empty Mix RGB / Indicator tabs
- User directive: functional parity + improved UX from user behavior perspective

**Key Changes:**

1. **`pages/backlight.html`** — Complete rewrite (~995 lines)
   - 23 effect chips (up from 13) with `data-anim` attributes for animation dispatch
   - Dynamic `requestAnimationFrame`-based keyboard preview animations:
     - `blStartBreathingAnim` — opacity pulse cycle (3s)
     - `blStartSpectrumAnim` — full hue rotation (5s)
     - `blStartRainbowWaveAnim` — directional wave (leftright/updown/outin)
     - `blStartTwinkleAnim` — random per-key phase+speed sparkle
     - `blStartPinwheelAnim` — angle-based color wheel rotation
     - `blStartSpiralAnim` — distance+angle spiral pattern
     - `blStartReactiveAnim` — random key press simulation with fade
     - `blStartRainAnim` — column-based raindrop trails
     - `blStartSplashAnim` — radial ripple expansion
     - `blStartBeaconAnim` — center-out wave pulse
   - HSV color picker: satval gradient box + hue strip + hex/RGB inputs + drag interaction
   - Per-key RGB tab: toggle, 4 preset palettes (Gaming/Ocean/Sunset/Forest), 9 color swatches + custom
   - Mix RGB tab: toggle, 2 zones (F1-F6, F7-F12) with effect selectors + speed sliders + mini keyboard
   - Indicator Light tab: CapsLock toggle, key preview, HSV color picker
   - Fixed toggle markup: `toggle-slider` → `toggle-track` + `toggle-thumb` (matching components.css)

2. **`i18n/ko.json`** + **`i18n/en.json`** — Added translations
   - 10 new effect names (13-22): rainbowBeacon, jellybean, pixelRain, typingHeatmap, digitalRain, reactiveSimple, reactiveMultiwide, reactiveMultinexus, splash, solidSplash
   - New keys: color, hexColor, mixRgbDesc, mixRgbZone, mixRgbEffect, indicatorKey, indicatorColor

3. **`css/pages.css`** — Added ~300 lines of new CSS classes
   - HSV picker: `.bl-hsv-picker`, `.bl-hsv-satval`, `.bl-hsv-hue-strip`, cursors
   - Color inputs: `.bl-color-inputs`, `.bl-color-preview`, `.bl-color-hex`, `.bl-color-rgb`
   - Per-key: `.bl-perkey-controls`, `.bl-perkey-header`, `.bl-palette-btn`, `.bl-swatch`
   - Mix RGB: `.bl-mix-panel`, `.bl-mix-zones`, `.bl-mix-zone`, `.bl-mix-select`
   - Indicator: `.bl-indicator-layout`, `.bl-indicator-keys`, `.bl-indicator-capslock-key`
   - Settings header: `.bl-settings-header-left`, `.bl-settings-dot`, `.bl-slider-icon`

4. **`index.html`** — Added CSS cache-busting script for development

**Note:** Real launcher doesn't animate keyboard preview in demo mode — our prototype goes beyond by providing dynamic `requestAnimationFrame` animations for each effect type.

### Navigation Gradient Effect (Commit: cd4e234)

**Background:**
- Previous sidebar nav-item used flat solid background for hover/active states — looked plain
- Analysis of real Keychron Launcher showed `::after` pseudo-element with `linear-gradient` + `@keyframes` animation
- Real launcher uses teal+purple+blue+pink multi-color gradient; we differentiated with Keychron orange-based tones

**Key Changes:**
- `css/layout.css` — Added animated gradient effect to nav-item
  - `@keyframes nav-gradient-shift` — background-position 0%→100%→0% (8s infinite loop)
  - `::after` pseudo-element: 6-stop orange-family gradient, `background-size: 500%`
  - Inactive: `opacity: 0`, Hover: `opacity: 0.55`, Active: `opacity: 1`
  - `transition: opacity 0.5s ease` — smooth fade in/out
  - `.nav-item > *` gets `z-index: 1` to keep text/icons above gradient
  - `.nav-item.active::before` (left bar) gets `z-index: 2`
  - Separate lighter gradient for light theme (reduced opacity values)


### Macro List Editor Replacement (Commit: 912a421)

**Background:**
- Previous macro editor used a chip-based timeline design
- Analysis of the real Keychron Launcher (`launcher.keychron.com`) showed list-based editor is far more intuitive
- V2 prototype developed on a separate page (`macro-v2.html`) then merged into the original

**Key Changes:**
- `pages/macro.html` — Complete replacement from chip timeline to list-based event editor
  - 2-column layout: Record controls (140px) | Event list (max-width 560px)
  - Card-style event rows with colored left borders (green=press, red=release, gray=delay, blue=text)
  - Custom record button: neutral gray pill + red dot (red bg + white dot blink when recording)
  - Always-visible row action buttons (up/down/more)
  - Context menu: Insert above / Insert below / Delete
  - QMK code sync in Code Input tab
- `css/pages.css` — Added new `ml-*` classes + removed old chip CSS (`mv2-toolbar`, `mv2-evt-*`, `mv2-timeline-*`)
- Deleted `macro-v2.html`, removed macro-v2 route from `app.js`

**New CSS Classes Added:**
- `ml-editor-split`, `ml-record-col`, `ml-rec-btn`, `ml-rec-dot`
- `ml-list-col`, `ml-list-header`, `ml-list-title`, `ml-list-actions`
- `ml-event-list`, `ml-row`, `ml-row--press/release/delay/text`
- `ml-row-check`, `ml-row-num`, `ml-row-type`, `ml-row-value`
- `ml-key-input`, `ml-delay-input`, `ml-delay-unit`, `ml-text-input`
- `ml-row-actions`, `ml-row-btn`, `ml-add-btn`
- `ml-context-menu`, `ml-ctx-item`, `ml-ctx-danger`, `ml-ctx-divider`
- `ml-code-textarea`, `ml-empty-state`, `ml-empty-hint`

### Phase 1: i18n — settings, feedback, he-mode (Commit: c6afd26)

**Modified files:**
- `i18n/ko.json` — ~25 new translation keys added
- `i18n/en.json` — same English keys
- `pages/settings.html` — data-i18n attributes (theme, device info, log buttons, etc.)
- `pages/feedback.html` — data-i18n attributes (email, attachment, dropzone, toast, etc.)
- `pages/he-mode.html` — JS "No Data", "keys" text → I18n.t() calls
- `js/i18n.js` — cache-busting on JSON fetch (`{ cache: 'no-store' }`)
- `js/app.js` — cache-busting on page template fetch

### Phase 2: i18n — backlight + advance (Commit: e522859)

**Modified files:**
- `i18n/ko.json` — 13 effect names + 4 palettes + 18 advance keys added
- `i18n/en.json` — same
- `pages/backlight.html` — data-i18n on 13 effect chips + 4 palette names
- `pages/advance.html` — data-i18n on DKS/Toggle/TapHold/Combo panel text

### Phase 3: Inline Styles → CSS — settings + feedback (Commit: b6524e1)

**Modified files:**
- `css/pages.css` — ~40 new CSS classes for settings/feedback + `.toast` flex fix
- `pages/settings.html` — all inline styles removed, replaced with CSS classes
- `pages/feedback.html` — ~15 inline styles removed, replaced with CSS classes

**Key patterns applied:**
- HTML `style="..."` → CSS classes
- JS `el.style.xxx = '...'` → `el.classList.toggle('className')`
- JS `onfocus`/`onblur`/`mouseenter`/`mouseleave` → CSS `:focus`/`:hover`

### Assist Page i18n (Included in commit: b6524e1)

- `pages/assist.html` — 14 hardcoded texts got `data-i18n` attributes, 3 JS strings → `I18n.t()` calls

### Macro Submit Button Fix (Commit: c498416)

- `css/pages.css` — `.macro-submit-btn` width:100% → min-width:160px
- `css/pages.css` — `.macro-submit-bar` justify-content: center → flex-end

### Pre-Plan Work (Previous Sessions)

- **Macro page redesign**: 2-sub-column editor layout (matching Keychron Launcher)
- **Backlight 2-column layout**: effect chips + settings panel side by side
- **Connect page**: 3-step cards + demo mode button
- **All 12 pages prototyped**

---

## Next Up (Phase 4)

### Phase 4: Inline Styles → CSS — backlight + keymap

**`pages/backlight.html`:**
- Inline styles → CSS classes (same pattern as Phase 3)

**`pages/keymap.html`:**
- Inline styles → CSS classes

---

## Technical Notes

### Cache Issues
- Python http.server caches JSON/HTML files → fixed with `{ cache: 'no-store' }` + `?v=${Date.now()}`
- Applied to both i18n.js fetch and app.js page template fetch

### SPA Router
- `app.js` fetches `pages/{pageId}.html` → injects into innerHTML → executes scripts separately
- `I18n.apply()` is called after page load in app.js line ~104

### ThemeManager
- Use `ThemeManager.getCurrent()` method (no `.current` property)
- `ThemeManager.set()`, `ThemeManager._updateSwitcherUI()` available
- localStorage-based

### Local Development
```bash
cd prototype
python3 -m http.server 8080
# View at http://localhost:8080
```

### Deployment
- `git push origin main` → GitHub Actions `deploy.yml` → SCP to EC2 → auto deploy (port 8083)
- Takes ~20-30 seconds

---

## Project Structure

```
prototype/
├── index.html          # SPA entry point (sidebar)
├── css/
│   ├── tokens.css      # Design tokens (Light/Dark)
│   ├── layout.css      # Sidebar + detail pane
│   ├── components.css  # Reusable components
│   └── pages.css       # Page-specific styles
├── js/
│   ├── theme.js        # Theme switching (System/Light/Dark)
│   ├── i18n.js         # Translation engine (KO/EN)
│   ├── app.js          # SPA router
│   └── components.js   # Keyboard renderer etc.
├── pages/              # 12 page templates
│   ├── connect.html, he-mode.html, keymap.html
│   ├── backlight.html, macro.html, assist.html
│   ├── advance.html, firmware.html, wireless-firmware.html
│   ├── test.html, feedback.html, settings.html
├── i18n/
│   ├── ko.json         # Korean translations
│   └── en.json         # English translations
└── assets/
    └── keyboard-k6he.svg
```
