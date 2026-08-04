# Block Seekers — Diagnostic Analysis Report

**Repository:** `munching-cats-web-assets`  
**Path:** `arcade/block-seekers/`  
**Date:** 2026-08-04  
**Scope:** WebGL black screen / render crashes inside Squarespace iframes

---

## 1. Codebase Status

| File | Role | Status |
|------|------|--------|
| `block-seekers.html` | Canonical game build | **Latest game code present** — 9 lives, maze, Shadow-Bomb / Bomberman mechanics (committed as `7b62ffd`, then enhanced with diagnostics) |
| `index.html` | Default document for directory / iframe URLs | **Was a 709-byte placeholder** (`[ WebGL Build Placeholder — Coming Soon ]`). **Now synced** to match `block-seekers.html` |
| `ANALYSIS.md` | This report | Created |

### Git audit (pre-fix)

- Branch: `main` (up to date with `origin/main`)
- Working tree was **clean** before diagnostic edits
- Latest relevant commits:
  - `7b62ffd` — Update block-seekers.html (9 Lives + Bomberman)
  - `a85b554` — Create block-seekers.html
  - `d585007` — Arcade placeholders including `index.html`

### Sync finding (critical)

**Squarespace (and most hosts) resolve a folder URL to `index.html`, not `block-seekers.html`.**

Until this diagnostic pass, embedding:

```text
.../arcade/block-seekers/
```

served the **placeholder**, which looks like a black / empty / “coming soon” panel — easily mistaken for a WebGL crash. Pointing the iframe at `block-seekers.html` would work; pointing at the directory would not.

**Resolution applied:** `index.html` overwritten with the same content as `block-seekers.html`.

---

## 2. Potential Causes of WebGL Black Screen / Crashes in Squarespace Iframes

### A. Wrong entry file (highest likelihood prior to sync)

- Iframe `src` targets directory → loads placeholder `index.html`
- Symptoms: dark background, no canvas, no start screen from the real game

### B. Zero-size viewport / canvas collapse

- Original code used `window.innerWidth` / `window.innerHeight` only
- Inside nested iframes, those can be `0` during early layout
- `WebGLRenderer.setSize(0, 0)` produces an invisible canvas (black host frame)
- Orthographic aspect `w/h` with `h === 0` → `Infinity` / `NaN` camera frustum → render failure

### C. Three.js CDN blocked

- Single dependency: `cdnjs.cloudflare.com/.../three.js/r128/three.min.js`
- Squarespace Content-Security-Policy, ad blockers, or corporate filters can block third-party scripts
- Symptom: `THREE is not defined` → silent script abort (no visible UI error before diagnostics)

### D. WebGL context denied

- Some browsers / privacy modes / iframe sandbox attributes block WebGL
- Missing `allow-scripts` / overly strict sandbox on the iframe
- Cross-origin iframe without GPU permission → `WebGLRenderer` throws or returns null context

### E. Timing / load order

- Inline game script runs before CDN finishes (race with async network)
- Fallback CDNs needed when primary fails mid-load

### F. AudioContext at parse time (secondary)

- Eager `new AudioContext()` can throw or warn in locked autoplay policies
- Unlikely to cause full black screen alone, but can abort script if uncaught in stricter environments

### G. Known gameplay bugs (not black-screen causes, but noted)

- Bomb blast loop uses `d.y` instead of `d.z` in `explodeBomb()` — horizontal blast arms may be wrong
- Several `MeshStandardMaterial({ emissive: 0.2 })` pass a number where Three.js expects a color — may warn or behave oddly

---

## 3. Three.js Initialization & CDN Verification

### Before diagnostics

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
```

- Version: **r128** (stable, widely cached)
- No `onerror` fallback
- No `typeof THREE` guard
- Immediate `init3D(); renderLoop();` at end of script

### After diagnostics (applied in both HTML files)

| Enhancement | Purpose |
|-------------|---------|
| `window.onerror` + `unhandledrejection` | Surface hidden JS errors in `#js-error-overlay` |
| CDN `onerror` → jsDelivr r128 | Primary CDN failure recovery |
| Secondary unpkg inject if `THREE` still missing | Tertiary CDN path |
| `getViewportSize()` with container measure + floors (`≥64`, defaults 640×480) | Prevent 0×0 canvas in iframes |
| `html/body` `100dvh` + `#viewport` min sizes + canvas `100%` CSS | Layout stability in embeds |
| Lazy `AudioContext` | Avoid autoplay/iframe audio aborts |
| `bootGame()` with THREE wait/retry (up to ~4s) + try/catch | Race-safe init |
| WebGL context null check after renderer create | Explicit GPU failure message |

### CDN chain

1. `https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js`
2. `https://cdn.jsdelivr.net/npm/three@0.128.0/build/three.min.js`
3. `https://unpkg.com/three@0.128.0/build/three.min.js`

---

## 4. Step-by-Step Resolution Plan

### Immediate (done in this pass)

1. ✅ Confirm `block-seekers.html` contains 9 Lives + Bomberman build  
2. ✅ Sync `index.html` ← `block-seekers.html`  
3. ✅ Add on-screen `window.onerror` diagnostics  
4. ✅ Add Three.js CDN fallbacks  
5. ✅ Harden viewport sizing for iframe embeds  

### Deploy / verify on Squarespace

1. **Commit & push** diagnostic + sync changes to `main` (or release branch GitHub Pages / CDN uses)
2. Point the Squarespace embed to either:
   - `.../arcade/block-seekers/` (now serves real game via `index.html`), **or**
   - `.../arcade/block-seekers/index.html` / `block-seekers.html` explicitly
3. Set iframe attributes roughly as:

   ```html
   <iframe
     src="https://YOUR_HOST/arcade/block-seekers/"
     width="100%" height="600"
     allow="autoplay; fullscreen"
     loading="eager"
     style="border:0;min-height:480px;background:#0c0c10;">
   </iframe>
   ```

4. Hard-refresh / purge CDN cache after deploy
5. Open DevTools on the **iframe document** (not only the parent):
   - Console: look for CSP / `THREE` / WebGL errors  
   - If red overlay appears, read the on-screen stack (diagnostics)

### If still black after deploy

| Check | Action |
|-------|--------|
| Overlay shows CDN failure | Host a local `three.min.js` under `arcade/block-seekers/vendor/` and script-src relative path (CSP-proof) |
| Overlay shows WebGL null | Remove sandbox restrictions; try another browser; confirm GPU not blocked |
| Overlay blank, canvas 0×0 | Ensure iframe has explicit `height` (not only `%` of unset parent) |
| Parent CSP blocks cdnjs/jsdelivr/unpkg | Add those hosts to Squarespace custom code CSP allowlist, or self-host Three.js |
| Wrong URL | Confirm Network tab requests the HTML that contains `Bast's Voxel Hide & Seek` |

### Optional follow-ups

1. Fix `explodeBomb` direction typo (`d.y` → `d.z`)
2. Fix invalid `emissive` numeric materials → proper `THREE.Color`
3. Self-host Three.js r128 to eliminate third-party CDN risk entirely
4. Add a tiny non-WebGL “loading / failed” banner independent of Three.js

---

## 5. Verification Checklist

- [x] Latest Bomberman / 9-lives code in `block-seekers.html`
- [x] `index.html` byte-synced with game file
- [x] `window.onerror` at top of game `<script>`
- [x] Three.js multi-CDN fallbacks
- [x] Viewport size guards against 0px iframe collapse
- [ ] Pushed to remote / live host
- [ ] Squarespace iframe URL retested after cache purge

---

## 6. Summary Verdict

The most likely cause of a “black screen” on Squarespace was **`index.html` still being the arcade placeholder**, while the real WebGL game lived only in `block-seekers.html`. Secondary risks (0px iframe sizing, single CDN, silent JS errors) are now mitigated with diagnostics and safer init. After deploy, any remaining failure should appear as a **red on-screen error overlay** instead of a silent black frame.
