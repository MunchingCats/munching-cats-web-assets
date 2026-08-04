# Block Seekers — Analysis Report (v2)

**Repository:** `munching-cats-web-assets`  
**Path:** `arcade/block-seekers/`  
**Build:** **v2 — Bast's Voxel Escape**  
**Date:** 2026-08-04  
**Entry files:** `block-seekers.html` + `index.html` (byte-identical)

---

## 1. Codebase Status (v2)

| File | Status |
|------|--------|
| `block-seekers.html` | Active v2 game (~30 KB) |
| `index.html` | **Synced** — same content as `block-seekers.html` |
| `ANALYSIS.md` | This document (v2 rewrite) |

### What v2 is

Title on screen: **Bast's Voxel Escape**. Pac-Man–style grid hide-and-seek with Bomberman counterplay, restyled as a **Minecraft village diorama**.

| Feature | v2 detail |
|---------|-----------|
| Grid | `GRID = 11`, village `MAP` (cobble paths + house walls) |
| Look | Sky `#78a7ff`, procedural grass/cobble canvas textures, wood walls + grass roofs |
| Camera | Orthographic, zoom `d = 5.2` (tighter framing than v1) |
| Bast | Black cat mesh: ears, green eyes, pink mouth, gold triple crown + collar |
| Alastor | Navy uniform, canvas **FA** badge, mushroom stalk/cap/gills |
| Lives | 9 paw icons |
| Blocks | Max 3 voxel shields (`SPACE` / BLOCK) |
| Win | Charge Shadow-Bomb (3 wall deflections) → `B` / BOMB → blast Alastor |
| Controls | Absolute WASD/arrows (`moveBastDirect`); on-screen controls legend |
| Score bridge | `postMessage` `MUNCHING_CATS_MINIGAME_SCORE` / `game: 'block_seekers'` |

### Sync status

Directory and explicit-file embeds both serve the real game (the v1 placeholder/`index.html` mismatch is resolved for this build).

---

## 2. v1 → v2 Delta (short)

| Area | v1 / interim | v2 |
|------|----------------|-----|
| Theme | Dark void maze | Minecraft sky + village cobble |
| Title UX | Hide & Seek | Voxel Escape + controls legend |
| Characters | Simple primitives | Bast crown/collar; Alastor FA + gills |
| Camera | Wider (`d ≈ 9.5`) | Closer (`d = 5.2`) |
| Movement API | `moveBast` | `moveBastDirect` (screen-absolute WASD) |
| Bomb blast dirs | Previously used `d.y` bug in one build | Fixed to `d.z` in `explodeBomb()` |
| Diagnostics | Briefly added (`onerror`, CDN fallbacks, iframe size guards) | **Removed** in this overwrite — single cdnjs Three.js only |

---

## 3. WebGL / Squarespace iframe risk (v2)

### Still healthy

- Dual entry files synced → folder URL loads the game
- Self-contained HTML + CDN Three.js r128 — no local asset pipeline
- Start overlay is opaque and readable even if scene looks odd

### Residual black-screen / crash risks

1. **0×0 canvas in iframes**  
   `init3D` / resize still use only `window.innerWidth` / `innerHeight`. Nested Squarespace iframes can report `0` early → invisible canvas or NaN orthographic aspect.

2. **Single CDN, no fallback**  
   ```html
   <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
   ```  
   CSP / blockers → `THREE is not defined` → silent failure (no on-screen error overlay in v2).

3. **Eager `AudioContext`**  
   Created at script parse. Can throw or fail under autoplay locks inside cross-origin iframes; may abort the rest of the script if uncaught.

4. **No `window.onerror` / WebGL null check**  
   Failures stay in the parent/iframe console only.

5. **Iframe height**  
   Parent must give a real height (e.g. `min-height: 480px`). `%` height of an unset parent still collapses layout.

### Recommended iframe snippet

```html
<iframe
  src="https://YOUR_HOST/arcade/block-seekers/"
  width="100%"
  height="600"
  allow="autoplay; fullscreen"
  loading="eager"
  style="border:0;min-height:480px;background:#78a7ff;">
</iframe>
```

---

## 4. Three.js init verification

| Check | Result |
|-------|--------|
| CDN | cdnjs Three.js **r128** only |
| Fallbacks | None in v2 |
| Renderer | `THREE.WebGLRenderer({ antialias: true })` |
| Size source | `window.innerWidth/Height` (no `#viewport` measure) |
| Scene bg | `0x78a7ff` (matches CSS body) |
| Boot | Immediate `init3D(); renderLoop();` at end of script |
| Resize | Updates orthographic frustum with `d = 5.2` |

---

## 5. Known gameplay / code issues (v2)

| Issue | Location | Effect |
|-------|----------|--------|
| Alastor AI uses `d.y` instead of `d.z` | `updateAlastor` validDirs filter | Vertical dirs never validate correctly → Alastor mostly only moves on ±X |
| Invalid `emissive: <number>` on several materials | Bast, wood, gills, shields, etc. | Three.js expects a color; may warn / ignore / look wrong |
| Title meta vs UI | `<title>` still “Hide & Seek”; H1 is “Voxel Escape” | Cosmetic inconsistency |
| Lives UI not drawn until `startGame` | `lives-box` empty on boot | Fine once PLAY is pressed |

Bomb cross blast (`explodeBomb`) correctly uses `d.z` in v2.

---

## 6. Step-by-step resolution plan

### Deploy

1. Commit/push `arcade/block-seekers/{index.html,block-seekers.html}` (and this `ANALYSIS.md` if desired)
2. Point Squarespace embed at `.../arcade/block-seekers/` or `index.html`
3. Purge CDN/cache; hard-refresh
4. Confirm Network tab HTML contains **Bast's Voxel Escape** and `const GRID = 11`

### If black / blank after deploy

1. Open DevTools **inside the iframe** document  
2. Console: CSP, `THREE is not defined`, WebGL errors  
3. Ensure iframe has explicit pixel height  
4. If CDN blocked → self-host `three.min.js` under `arcade/block-seekers/vendor/`  
5. Optionally re-add v1 diagnostics: `window.onerror` overlay, CDN fallbacks, `getViewportSize()` from `#viewport`

### Code fixes (next patch)

1. Fix Alastor filter: `alastorPos.z + d.z` (not `d.y`)  
2. Replace numeric `emissive` with colors (e.g. `emissive: 0x222222`) or drop the property  
3. Lazy-init `AudioContext` on first user gesture  
4. Align `<title>` with “Bast's Voxel Escape”  
5. Reintroduce iframe-safe sizing + error overlay for production embeds

---

## 7. Verification checklist

- [x] v2 Minecraft village build in both HTML entry files  
- [x] `index.html` ↔ `block-seekers.html` hashes match  
- [x] 9 lives, shields, Shadow-Bomb, absolute WASD, controls legend present  
- [ ] Alastor `d.y` bug fixed  
- [ ] Emissive material cleanup  
- [ ] Pushed / live host verified in Squarespace iframe  
- [ ] Optional: diagnostics + viewport guards restored for production

---

## 8. Verdict

**v2 is a visual and UX upgrade** (village diorama, character detail, controls legend, tighter camera) with **entry files correctly synced** for directory embeds. The main remaining embed risks are **iframe zero-size**, **single Three.js CDN**, and **silent JS failures** (diagnostics were stripped in this overwrite). The main gameplay bug to fix next is **Alastor AI using `d.y` instead of `d.z`**, which cripples vertical patrol.
