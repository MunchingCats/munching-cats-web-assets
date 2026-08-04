# Block Seekers — Audit Report (Diagnostics & Iframe Hardening)

**Repository:** `munching-cats-web-assets`  
**Canonical file:** `arcade/block-seekers/block-seekers.html`  
**Date:** 2026-08-04  
**Build:** Bast's Voxel Escape (low walls, face-forward camera, fixed Alastor `d.z` AI)

---

## 1. Codebase Status

| Item | Status |
|------|--------|
| Entry file | **`block-seekers.html` only** (`index.html` removed from repo) |
| Embed URL | Must point at `.../arcade/block-seekers/block-seekers.html` (directory URL alone will 404 / list without an index) |
| Gameplay build | Village map (path / low wall / water / wheat), 9 lives, Shadow-Bomb, absolute WASD |
| This audit | Diagnostics + iframe hardening applied and documented |

### Squarespace embed note

Use the **explicit** HTML path:

```html
<iframe
  src="https://YOUR_HOST/arcade/block-seekers/block-seekers.html"
  width="100%"
  height="600"
  allow="autoplay; fullscreen"
  loading="eager"
  style="border:0;min-height:480px;background:#6093e7;">
</iframe>
```

---

## 2. Pre-Audit Findings (what was broken / fragile)

| Risk | Pre-audit behavior | Impact in Squarespace iframes |
|------|--------------------|-------------------------------|
| Silent JS failures | No `window.onerror` / rejection handler | Black frame; errors only in parent console |
| Single Three.js CDN | cdnjs only | CSP / blockers → `THREE is not defined` |
| Eager `AudioContext` | `new AudioContext()` at script parse | Autoplay policy can throw and abort boot |
| Window-based sizing | `window.innerWidth/Height` only | Nested iframes can report `0` → 0px canvas / NaN aspect |
| Immediate init | `init3D()` with no THREE wait | Race if CDN fallback still loading |

---

## 3. Hardening Applied

### 3.1 On-screen error overlay

- `#js-error-overlay` (red monospace text)
- `window.onerror` → prints message + source + stack
- `unhandledrejection` → prints promise failures
- Init failures (`bootGame` try/catch) also surface here

### 3.2 Multi-CDN Three.js (r128)

1. **cdnjs** — `.../three.js/r128/three.min.js`  
2. **jsDelivr** — via `<script onerror>` rewrite  
3. **unpkg** — injected if `THREE` still missing  

`bootGame()` retries up to ~4s if THREE arrives late.

### 3.3 Deferred AudioContext

- `AudioCtx` starts as `null`
- Created in `startGame()` via `ensureAudioCtx()` (user gesture path)
- `playSound()` no-ops until context exists

### 3.4 `#viewport`-based sizing

- `getViewportSize()` reads `clientWidth` / `clientHeight` of `#viewport`
- Falls back to window / document, then floors at **640×480** if still &lt; 64px
- Used by `init3D()` and `onWindowResize()`
- CSS: `100dvh`, min-heights, canvas `width/height: 100%`

### 3.5 Safer WebGL boot

- Throws clear errors if `THREE` missing, `#viewport` missing, or WebGL context null
- `setPixelRatio` capped at 2
- `renderLoop` guards null renderer/scene/camera

---

## 4. Remaining Notes

| Topic | Status |
|-------|--------|
| Alastor AI `d.z` | Fixed in current gameplay build |
| Numeric `emissive` on some materials | Still present (cosmetic / Three.js type quirk); not a black-screen cause |
| Water / wheat tiles (`MAP` 2 / 3) | Walkable (only `MAP === 1` blocks) — intentional for village feel |
| Self-hosting Three.js | Optional next step if all three CDNs blocked by CSP |

---

## 5. Verification Checklist

- [x] Canonical single file: `block-seekers.html`
- [x] `window.onerror` + `unhandledrejection` overlay
- [x] CDN chain: cdnjs → jsDelivr → unpkg
- [x] `AudioContext` created in `startGame()` only
- [x] Renderer size from `#viewport` via `getViewportSize()`
- [ ] Live Squarespace iframe retested after deploy (explicit `.html` URL)
- [ ] Hard-refresh / CDN cache purge after push

---

## 6. Verdict

The game was already feature-complete for the village escape loop; the main embed failures were **silent errors**, **CDN dependency**, **autoplay AudioContext**, and **0px iframe sizing**. Those are now mitigated in `block-seekers.html`. Embeds must target **`block-seekers.html` directly** because `index.html` no longer exists.
