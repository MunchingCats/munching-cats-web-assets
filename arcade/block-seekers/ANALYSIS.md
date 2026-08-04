# Block Seekers — Game Architecture & Quality Analysis

**Repository:** `munching-cats-web-assets`  
**Canonical file:** `arcade/block-seekers/block-seekers.html`  
**Date:** 2026-08-04  
**Build:** Bast's Voxel Escape (Three.js voxel-village rewrite)

---

## 1. Codebase Status

| Item | Status |
|------|--------|
| Entry file | **`block-seekers.html` only**; the directory has no `index.html` |
| Runtime architecture | Self-contained HTML, CSS, UI, game loop, procedural models, textures, audio, and effects |
| External dependency | Three.js r128, loaded sequentially from cdnjs → jsDelivr → unpkg |
| Gameplay build | 11×11 village, 9 lives, three shields, three-charge Shadow-Bomb, BFS enemy AI |
| Input support | Keyboard, arrow keys, and coarse-pointer touch controls |
| Parent integration | Posts `MUNCHING_CATS_MINIGAME_SCORE` on victory or defeat |
| Current verification | JavaScript syntax, Git diff integrity, and headless Chrome WebGL initialization passed |

### Squarespace embed note

Use the **explicit HTML path** because the game directory does not contain an index document:

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

## 2. Game Architecture

| System | Implementation | Assessment |
|--------|----------------|------------|
| Render loop | `requestAnimationFrame` with delta time capped at 40 ms | Frame-rate independent and protected from large resume spikes |
| World model | String-based 11×11 level plus `destroyed` wall set | Compact, readable, and supports runtime destruction |
| Movement | Grid-authoritative positions with exponential mesh interpolation | Responsive collision logic with visually smooth locomotion |
| Enemy navigation | Breadth-first search toward Bast every movement cycle | Deterministic shortest-path pursuit; substantially stronger than random stepping |
| Line of sight | Cardinal row/column scan through walls, water, bombs, and shields | Clear arcade rules that match the visible red cone |
| Projectiles | Continuous floating-point travel at 5.7 tiles/second | Fast shots remain visually smooth and can be intercepted |
| Effects | Fixed pool of 110 debris meshes plus temporary sprite text | Avoids particle allocation during combat; text objects are disposed |
| Audio | Deferred Web Audio oscillators and generated noise buffer | No external audio assets and compliant with browser gesture policies |
| State flow | Start overlay → active run → result overlay → clean reset | All dynamic shots, shields, bombs, particles, and destroyed walls reset |

### Coordinate and terrain model

- Grid coordinates are converted to world coordinates with a fixed `HALF` offset.
- Dirt blocks form the voxel sub-layer; grass tops sit at approximately **Y=0.30**.
- Cobblestone overlays sit at approximately **Y=0.32**.
- Animated water is recessed and treated as non-walkable.
- Wheat is decorative and walkable.
- Village walls top out at **Y=0.60**.
- Exterior terrain receives small height variations to create a terraced diorama edge.
- Trees and clouds remain outside the navigable board, preventing line-of-sight obstruction.

---

## 3. Gameplay & Presentation Analysis

### 3.1 Core gameplay loop

1. Bast moves in absolute screen-space directions.
2. Alastor computes a shortest path and closes distance continuously.
3. Cardinal line of sight allows Alastor to fire a spore.
4. Bast dodges or places a shield so the projectile hits cover.
5. Each impact charges one of three Shadow-Bomb segments.
6. A charged bomb ticks for 1.5 seconds and detonates four tiles in each cardinal direction.
7. The player wins by placing the blast path across Alastor's predicted route.

The loop has a strong **evade → intercept → charge → predict → detonate** structure. Shields are tactical rather than permanent because an impact consumes them, while placing a fourth shield removes the oldest one.

### 3.2 Bast

- Bast is deliberately oversized relative to one grid tile.
- The model includes black body volumes, emissive lime eyes, pink mouth and nose, fangs, a metallic three-spike crown, collar, ears, and tail.
- Movement uses position interpolation, pitch/roll gait, and a separate idle breathing cycle.
- Facing follows the latest input direction, preserving readable directional intent.
- Damage grants 1.15 seconds of invulnerability and a visible blink cycle.

### 3.3 Alastor

- Alastor has a navy uniform, canvas-generated `FA` badge, limbs, neck stalk, mushroom cap, and radial underside gills.
- The cap uses an independent rig for continuous wobble.
- A translucent red wedge communicates current facing and obstruction distance.
- BFS pursuit makes Alastor pressure the player instead of wandering.
- Firing rate gradually accelerates, down to a 0.48-second lower bound.

### 3.4 Combat feedback

| Event | Visual feedback | Audio feedback |
|-------|-----------------|----------------|
| Shield placement | Blue voxel, edge outline, debris burst, `SHIELD!` text | Rising triangle tone |
| Spore firing | Emissive red projectile and muzzle particles | Descending sawtooth shot |
| Deflection | Colored impact debris and `DEFLECTED!` text | Rising square confirmation |
| Full charge | Pulsing HUD bar and `BOMB READY!` text | Deflection confirmation |
| Bomb | Pulsating purple sphere and rotating ring | Arming tone |
| Explosion | Cross-shaped flashes, debris, camera shake | Oscillator impact plus noise burst |
| Player damage | Red debris, blink invulnerability, shake, `-1 LIFE` | Low descending damage tone |

### 3.5 Score model

- Passive survival: **+4 points/second**
- Deflection: **+150**
- Destroyed wall: **+100**
- Damage: **−75**
- Remaining life bonus: **+400 per life**
- Victory bonus: **+2,500**
- Completion adjustment: **−3 points/second**

Because passive score exceeds the completion penalty, the net result still increases by approximately one point per second. This slightly rewards delaying a winning bomb. If speed-running is intended, passive survival scoring should be removed or reduced below the completion penalty.

---

## 4. Risk & Improvement Notes

| Priority | Topic | Analysis | Recommendation |
|----------|-------|----------|----------------|
| High | Mobile rendering cost | The scene uses many individual terrain, wall, crop, gill, and particle meshes with 2048² shadows and pixel ratio up to 2 | Convert repeated terrain and vegetation to `InstancedMesh`; reduce shadow map to 1024 on small screens |
| Medium | Difficulty curve | BFS pursuit and an accelerating firing interval can become oppressive in narrow corridors | Scale AI movement and fire rate from elapsed time in clearer stages |
| Medium | Score incentives | Net time scoring weakly rewards longer runs | Make time a pure penalty or add a visible par-time bonus |
| Medium | CDN-only engine | All fallbacks are remote and may be blocked by a strict iframe CSP | Host a local pinned copy of Three.js as the final fallback |
| Medium | Game pause | Losing focus does not pause simulation; delta capping prevents a jump but gameplay resumes immediately | Pause on `visibilitychange` and provide a pause overlay |
| Low | Accessibility | Touch and keyboard work, but there is no mute, reduced-motion mode, or focus transfer between overlays | Add mute/pause controls and respect `prefers-reduced-motion` |
| Low | WebGL recovery | Initialization errors surface, but context loss is not recovered | Handle `webglcontextlost` and `webglcontextrestored` |
| Low | PostMessage scope | Score uses target origin `"*"` | Keep payload non-sensitive and validate event origin in the parent; use a known origin when deployment is fixed |

### Behavioral edge cases

- Player and AI coordinates become authoritative at movement start, before their meshes finish interpolating. This is good for deterministic grid collision, but very fast reactions can appear to collide slightly ahead of the visible model.
- Alastor does not enter Bast's tile. Instead, an attempted path step into that tile causes damage and retries after a short delay.
- Bomb blasts stop after striking the first destructible interior wall in each direction.
- Border walls are intentionally indestructible, preventing escape from the 11×11 arena.
- Water blocks movement and projectiles; wheat does neither.

---

## 5. Verification Checklist

- [x] Canonical single-file build at `block-seekers.html`
- [x] JavaScript parsed successfully with Node
- [x] `git diff --check` passed
- [x] Headless Chrome created the WebGL canvas without triggering the diagnostic overlay
- [x] Sequential Three.js fallback chain: cdnjs → jsDelivr → unpkg
- [x] `AudioContext` created only after the Play gesture
- [x] Renderer dimensions derived from `#viewport`
- [x] Delta-time movement, AI, projectiles, bomb fuse, and effects
- [x] Dynamic assets removed or reset between runs
- [x] Victory and defeat both post a score payload
- [ ] Manual full-play victory test with keyboard
- [ ] Manual touch-device gameplay test
- [ ] Live Squarespace iframe test after deployment
- [ ] Performance profiling on a low-end mobile GPU

---

## 6. Verdict

The rewrite delivers a coherent and feature-complete arcade loop with strong character readability, deterministic grid mechanics, meaningful defensive choices, and substantially improved visual feedback. Its architecture is appropriate for a self-contained embedded minigame: simulation is time-based, audio is gesture-gated, failures are visible, and external score reporting is isolated to the end state.

The primary production concern is **rendering cost**, not gameplay correctness. The world is procedurally assembled from many separate meshes, which is acceptable on desktop but should be instanced before treating low-end mobile performance as guaranteed. Balance also needs a manual full-play pass, particularly the late-game Alastor firing rate and the score system's small incentive to delay victory.

**Overall assessment:** production-capable desktop embed, functionally complete, with mobile optimization and live iframe validation remaining before final release sign-off.
