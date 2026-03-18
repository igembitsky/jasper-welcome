# Lightsaber Progress Bar Redesign

## Context

The current lightsaber scroll-progress indicator is vertical, positioned on the left side of the viewport. The user wants it reimagined as a horizontal lightsaber along the bottom of the screen with a realistic Star Wars glow, aggressive spark particles, and a more readable percentage display. The goal is a fun, playful animation that makes users want to scroll just to watch it.

## Decisions

- **Blade color:** Green (Yoda/Luke ROTJ)
- **Orientation:** Horizontal, fixed to bottom of viewport
- **Hilt position:** Left side, blade extends right
- **Spark approach:** Canvas overlay for particle system
- **Spark intensity:** Full fireworks — aggressive, high particle count
- **Percentage display:** Glowing holographic readout integrated into the hilt

## File to Modify

- `/Users/igor/projects/jasper-welcome/index.html` (all CSS, HTML, and JS are in this single file)

## Design

### 1. Layout & Structure

**Position:** Fixed to the bottom of the viewport, ~20px from bottom edge, ~20px left/right padding.

**Components (left to right):**

1. **Hilt** (~50px wide, 36px tall)
   - Metallic ridged appearance via `repeating-linear-gradient(90deg, ...)` (adapted from current vertical hilt)
   - Rounded corners: `4px`
   - Inner border highlight and drop shadow for depth
   - Contains the percentage readout (see section 4)
   - `::before` — left-side emitter cap (6px wide metallic gradient)
   - `::after` — accent glow strip (green gradient, centered)

2. **Blade** (variable width, 8px tall)
   - Width = `scrollPercent%` of available container space (container = viewport width - hilt width - padding)
   - Rounded right end only: `border-radius: 0 4px 4px 0`
   - 5-layer glow system (see section 2)
   - `transition: width 0.15s ease-out`

3. **Canvas overlay** (absolutely positioned)
   - Covers blade area + 60px above and 40px below for spark travel space
   - `pointer-events: none`
   - Handles all spark particle rendering

**Container:**
- `position: fixed; bottom: 20px; left: 20px; right: 20px;`
- `z-index: 200; pointer-events: none;`
- `display: flex; align-items: center;`

**Entry animation:**
- Initially: `translateY(80px)` (below viewport)
- On scroll > 3%: add `.visible` class → `translateY(0)` with `0.6s cubic-bezier(0.34, 1.56, 0.64, 1)` (existing ease-out-back)

**Responsive (mobile ≤ 768px):**
- Scale to 65%: `transform-origin: bottom left; scale(0.65)`
- Bottom: 10px, left: 10px, right: 10px

### 2. Blade Glow (5 layers, inside-out)

1. **Core gradient:** `linear-gradient(90deg, #fff 0%, #c8e6c9 10%, #66bb6a 40%, #2e7d32 100%)`
   - White-hot at the hilt connection, fading to deep green at the tip

2. **Inner glow (box-shadow):** `0 0 10px #66bb6a, 0 0 20px rgba(102,187,106,0.7)`

3. **Outer glow (box-shadow):** `0 0 40px rgba(102,187,106,0.4), 0 0 80px rgba(102,187,106,0.2)`

4. **Bloom (::after pseudo-element):**
   - `position: absolute; inset: -4px; border-radius: inherit;`
   - `background: rgba(102,187,106,0.15); filter: blur(8px); z-index: -1;`

5. **Tip flare (::before pseudo-element):**
   - Positioned at right edge of blade
   - `background: radial-gradient(circle, rgba(255,255,255,0.6) 0%, rgba(102,187,106,0.3) 40%, transparent 70%)`
   - 24px circle, CSS `pulse` animation (2s cycle, subtle opacity breathe)
   - Must track blade tip position — use `right: -6px` relative to blade

### 3. Spark Particle System (Canvas)

**Canvas element:**
- Dimensions: same width as saber container, ~100px tall
- Positioned to overlap the blade vertically (blade centered within canvas height)
- Cleared and redrawn each frame via `requestAnimationFrame`

**Emission:**
- Emit from blade tip (right edge of current blade width)
- 3-6 particles per frame **while user is actively scrolling**
- Emission rate proportional to scroll velocity (faster scroll = more sparks)
- When scrolling stops, taper emission over ~0.5s (don't cut abruptly)

**Per-particle properties (randomized on creation):**
- `x, y` — blade tip position
- `vx` — random: 50–200 px/s, biased right and upward
- `vy` — random: -200 to 100 px/s (mostly upward)
- `angle` — spread from -30° to -150° from horizontal
- `gravity` — 30 px/s² downward
- `size` — 1–4px
- `shape` — 70% circle, 30% elongated streak (2:1 aspect ratio aligned to velocity)
- `color` — random from: `#fff`, `#c8e6c9`, `#66bb6a`, `#a5d6a7`; 15% chance of bright `#fff` flash
- `lifetime` — 0.3–0.8s
- `alpha` — starts at 0.8–1.0, fades to 0 over last 30% of lifetime

**Trail effect:**
- Each particle stores last 2-3 positions
- Ghost positions rendered at 50% and 25% of current alpha
- Creates motion streak behind each spark

**Performance:**
- Max 50 active particles at any time (recycle oldest if exceeded)
- Use object pool to avoid GC pressure
- Sync animation loop with existing scroll handler's `requestAnimationFrame`

### 4. Percentage Display

- Rendered inside the hilt element
- Font: JetBrains Mono, 11px, bold
- Color: `#66bb6a`
- `text-shadow: 0 0 8px #66bb6a, 0 0 16px rgba(102,187,106,0.5)`
- Updates in real-time with scroll (reuses existing scroll handler)
- Format: `XX%` (no decimals, 0-100)

### 5. Code Organization

All code stays in `index.html` (consistent with current architecture):

**CSS changes:**
- Replace all `.lightsaber*` styles with horizontal versions
- Remove the old vertical blade-wrap and blade styles
- Add new horizontal container, hilt, blade, canvas positioning
- Update mobile media query

**HTML changes:**
- Replace current lightsaber markup with:
  ```html
  <div class="lightsaber" id="lightsaber">
    <div class="lightsaber-hilt">
      <span class="lightsaber-pct" id="lightsaber-pct">0%</span>
    </div>
    <div class="lightsaber-blade-wrap">
      <div class="lightsaber-blade" id="lightsaber-blade"></div>
      <canvas class="lightsaber-sparks" id="lightsaber-sparks"></canvas>
    </div>
  </div>
  ```

**JavaScript changes:**
- Update scroll handler to set blade `width` instead of `height`
- Add particle system class/functions:
  - `createParticle(x, y)` — creates a spark with randomized properties
  - `updateParticles(dt)` — physics step (velocity, gravity, alpha decay)
  - `renderParticles(ctx)` — draws all active particles + trails to canvas
  - `emitSparks(bladeWidth, scrollVelocity)` — spawns particles based on scroll speed
- Canvas resize handler (match container dimensions)
- Object pool array for particles (pre-allocate 50 slots)

## Verification

1. Open `index.html` in a browser
2. Scroll down — lightsaber should slide in from below at 3% scroll
3. Blade extends right proportional to scroll progress
4. Hilt shows glowing green percentage
5. Sparks fly from blade tip while scrolling, with arcing trajectories and fading trails
6. Sparks taper off when scrolling stops
7. Scroll to 100% — blade fills full width, sparks are at max intensity
8. Test on mobile viewport (≤768px) — saber scales down, still functional
9. Verify no performance issues (check for dropped frames in DevTools)
