# PCR & RT-qPCR — Interactive Teaching Module

## Overview

A single-file HTML5 interactive teaching module (`pcr_teaching_module v2.html`) covering the full PCR workflow — from the discovery of Taq polymerase through qPCR quantification. All content, styling, SVG animations, and logic live in one file with zero external dependencies beyond Google Fonts.

Designed for tablet/display use (`html { zoom: 1.4 }`), dark theme, landscape orientation.

---

## File Structure

```
pcr_teaching_module v2.html   ← the entire module
download.jpg                  ← Yellowstone hot springs photo (S1)
Longitudinal-view-of-...webp  ← Thermus aquaticus TEM image (S1)
chien-et-al-1976_Page_1.png   ← Chien et al. 1976 paper image (S1)
README.md                     ← this file
```

All images must be in the same directory as the HTML file.

---

## Navigation Architecture

### Section Order (`SCS` array — the sole routing source of truth)

The HTML contains 9 `<div class="sc">` sections with IDs s1–s9. Their **order in the DOM does not determine navigation order** — the `SCS` array does:

```javascript
const SCS = ['s1','s2','s4','s5','s6','s3','s7','s8','s9'];
```

| Nav position | HTML ID | Title |
|---|---|---|
| 01/09 | s1 | The Discovery |
| 02/09 | s2 | DNA Structure |
| 03/09 | s4 | Denaturation |
| 04/09 | s5 | Primer Annealing |
| 05/09 | s6 | Extension |
| 06/09 | s3 | Thermocycling |
| 07/09 | s7 | Amplification |
| 08/09 | s8 | RT-PCR |
| 09/09 | s9 | qPCR |

Note: s3 (Thermocycler) appears 6th in navigation even though it is the 3rd `<div>` in the HTML. Never rely on DOM order for section sequence — always edit `SCS`.

### Navigation controls

- PREV / NEXT buttons in `<nav>`
- Clickable dot indicators (`dotsEl`)
- Keyboard: `←` / `→` or `PageUp` / `PageDown`
- Direct dot click jumps to any section

### `done[]` object

Tracks first-entry vs revisit per section ID. `onEnter()` fires only on first visit. Revisits re-run init functions (e.g. `initDenSlider()`, `initAnnealSlider()`, `initExtSlider()`) so the section always resets to its starting state. Exception: s7 (Amplification) has a manual Reset button that sets `done['s7'] = false`.

---

## Interactive Sections

### Shared Slider Infrastructure

A single `#tempSlider` widget (fixed, right edge, vertically centered) is reused across s4, s5, and s6. It is hidden on all other sections.

```
#tempSlider
  #tsv         — current temperature value (e.g. "72°C")
  #tsstate     — state label (e.g. "EXTENDING", "ANNEALED")
  #tstrack
    .tstk       — static reference tick marks (94°, 72°, 60°, 37°)
    #tsrange    — <input type="range"> vertical via writing-mode:vertical-lr
  #tshint      — "DRAG TO HEAT" / "DRAG TO COOL"
```

The slider track fill uses two CSS custom properties set via JavaScript:
- `--tc` — current colour (interpolated via `tempToColor()`)
- `--fp` — fill percentage from bottom

**Critical:** `--fp` is calculated relative to each section's own temperature range, not a fixed [20–100] scale:
- s4, s5: `((temp - 20) / 80 * 100)%`
- s6: `((temp - 60) / 32 * 100)%`

The slider's `min`/`max`/`value` are set by each section's init function. Because all three sections share one `<input>`, each init function **must** explicitly restore `min` and `max` — failure to do so would constrain range when navigating back from s6 (which uses min=60) to s4/s5 (which need min=20).

---

### S4 — Denaturation (interactive slider)

**Range:** 20–100°C, starts at 37°C  
**SVG:** `#dna4` — two strand `<g>` groups plus H-bond dots

| Element | Behaviour |
|---|---|
| `#sl4l` | Left strand group — `translateX(−shift)` as temp rises |
| `#sl4r` | Right strand group — `translateX(+shift)` as temp rises |
| `#sl4h` | H-bond dots — opacity fades 1→0 |
| `#heatl` | Heat wave lines — opacity rises 0→1 |

Denaturation factor starts at 75°C, completes at 94°C, eased with quadratic bezier.  
CSS: `#sl4l, #sl4r, #sl4h, #heatl` all have `transition: 0.12s ease` for real-time slider response.

---

### S5 — Primer Annealing (interactive slider)

**Range:** 20–100°C, starts at 92°C (fully detached)  
**SVG:** inline in s5 HTML — two template strands (faded), FWD primer (`#fp`/`#fpl`), REV primer (`#rp`/`#rpl`)

Annealing factor is inverted: 0 = hot/detached, 1 = cool/annealed.  
Transition zone: 72°C (detached) → 58°C (annealed).

| Element | Behaviour |
|---|---|
| `#fp`, `#fpl` | FWD primer — `translateX(+shift)` when detached, opacity fades |
| `#rp`, `#rpl` | REV primer — `translateX(−shift)` when detached, opacity fades |

CSS: `#fp, #rp, #fpl, #rpl { transition: transform .35s ease, opacity .35s ease }`

---

### S6 — Extension (interactive slider, two-phase)

**Range:** 60–92°C, starts at 60°C  
**SVG:** `viewBox="-28 0 420 420"` — two `<g>` groups for left and right duplexes

#### SVG Group Structure

```
<svg>
  <g id="s6gl">          ← Left duplex (template + REV primer + new left strand + tL)
    rect (left backbone)
    rect × 8 (left rungs, faded)
    rect (REV primer, y=279)
    rect id="n0a"        ← new left strand backbone (grows UP from y=279)
    rect.nse × 8 id="n0b–n7b"  ← new left strand rungs
    g.taq-g id="tL"      ← Taq circle, starts cx=134 cy=279
  </g>
  <g id="s6gr">          ← Right duplex (template + FWD primer + new right strand + tR)
    rect (right backbone)
    rect × 8 (right rungs, faded)
    rect (FWD primer, y=80)
    rect id="n0c"        ← new right strand backbone (grows DOWN from y=137)
    rect.nse × 8 id="n0d–n7d"  ← new right strand rungs
    g.taq-g id="tR"      ← Taq circle, starts cx=218 cy=137
  </g>
  text labels (outside groups)
</svg>
```

**Key coordinates:**
- REV primer: `x=90 y=279 height=57` (bottom of left template amplicon region)
- FWD primer: `x=232 y=80 height=57` (top of right template amplicon region)
- tL start: `cx=134 cy=279` (on new strand at REV primer top)
- tR start: `cx=218 cy=137` (on new strand at FWD primer bottom)
- n0a (left backbone) grows upward from y=279 toward top
- n0c (right backbone) grows downward from y=137 toward bottom

#### Two-Phase Animation

**Phase 1 — Extension (60→72°C):**
- `extF` goes 0→1 with ease-in-out
- `tL` translates `Y(extF × −235)px` (moves up 235px from cy=279 to cy=44)
- `tR` translates `Y(extF × 236)px` (moves down 236px from cy=137 to cy=373)
- `n0a` height grows: `279 − (extF×235)` → top, height = `extF×235`
- `n0c` height grows: height = `extF×236`
- New strand rungs (n0b–n7b, n0d–n7d): opacity `extF × 0.7`
- State: READY → EXTENDING → COMPLETE

**Phase 2 — Denaturation (72→92°C):**
- `denF` goes 0→1 with ease-in-out, independent of extF
- `s6gl` translates `X(−denF×70)px`
- `s6gr` translates `X(+denF×70)px`
- State: DENATURING → DENATURED

#### CSS Specificity for Taq transitions

```css
.taq-g { transition: transform 1.9s cubic-bezier(.2,.8,.2,1) }  /* auto-animation legacy */
#tL, #tR { transition: transform .12s ease }                     /* overrides above — slider-responsive */
#s6gl, #s6gr { transition: transform .35s ease }                 /* smooth group separation */
```

ID selectors win over class selectors — `#tL`/`#tR` at 0.12s overrides the 1.9s class transition. This is required so Taq tracks the slider in real time rather than lagging by ~2 seconds.

#### `.nse` class conflict

The `.nse` CSS class applies `transform: translateX(12px)` to new strand rungs (a legacy slide-in effect from the old auto-animation). When slider-driven, this would visually offset rungs to the right. Fix: `initExtSlider()` sets `el.style.transform = 'none'` and `el.style.transition = 'none'` via inline styles, which override the class.

---

## Static Sections

### S1 — Discovery

Timeline of events (Brock 1966, Mullis 1983, Nobel 1993) with staggered CSS animation (`.tli.show` class added with `setTimeout` stagger).

Three clickable photos (`.lb-img`) trigger the lightbox. Click anywhere outside the image to dismiss.

**Lightbox:** `#lb` — fixed overlay, `opacity:0 pointer-events:none` by default, `.open` class shows it. Implemented via `document.querySelectorAll('.lb-img')` click listeners in the `<script>` block.

### S2 — DNA Structure

Static animated SVG (`#dna2`) with a CSS `@keyframes pulse` scale animation. Shows double helix with A·T and G·C base pair labels.

### S3 — Thermocycling

Canvas-drawn temperature profile (`#tc`, 500×165px). Three cycle segments drawn as coloured polylines. Drawn once on entry via `drawTherm()`.

### S7 — Amplification

Two elements:
1. **Copy counter** — `#cnum` shows current copy count; `#cyc` shows cycle number. "Run Cycle" button calls `runCycle()`.
2. **Canvas chart** (`#ampc`, 460×300px) — drawn via `drawAmpCurve(cycle)`. Shows real curve (cubic Hermite: exponential 0→12, blending to plateau at ~28) and ideal exponential (dashed).

Ct = 12 hardcoded. Plateau = 2^19 = 524,288 copies.

### S8 — RT-PCR

Static SVG diagram showing: mRNA box → reverse transcriptase arrow → cDNA box → PCR arrow → Amplicon box. No animation.

### S9 — qPCR

Canvas-drawn (`#qc`, 460×280px) sigmoidal amplification curves. Two curves: high template (low Ct, green) and low template (high Ct, red). Drawn once on entry via `drawQPCR()`.

---

## CSS Design System

All colours use CSS custom properties defined on `:root`:

```css
--dg: #1c5c3a   /* dark green */
--mg: #3a8c5e   /* mid green */
--lg: #5eba82   /* light green (accent) */
--cr: #8c1c1c   /* dark crimson */
--cl: #c43030   /* light crimson */
--bg: #080f0a   /* near-black background */
--tx: #dceee4   /* body text */
--mu: #7aab8a   /* muted text */
--hot: #c43030  /* high-temp colour */
--warm: #e07030 /* mid-temp colour */
--cool: #3a6ec4 /* low-temp colour */
```

Fonts: Cormorant Garamond (headings/titles), JetBrains Mono (labels/data/UI), Nunito Sans (body text).

---

## Key JavaScript Functions

| Function | Purpose |
|---|---|
| `goTo(n)` | Navigate to section index n. Handles slide transition, header badge, dot state, temperature gauge visibility, slider visibility, done[] tracking |
| `onEnter(id)` | First-entry callbacks per section |
| `onSliderInput(v)` | Routes slider value to section-specific handler |
| `setDenTemp(temp)` | s4 — drives denaturation SVG from slider value |
| `setAnnealTemp(temp)` | s5 — drives primer annealing SVG from slider value |
| `setExtTemp(temp)` | s6 — drives extension two-phase SVG from slider value |
| `initDenSlider()` | s4 — sets slider min=20, max=100, value=37, resets SVG |
| `initAnnealSlider()` | s5 — sets slider min=20, max=100, value=92, resets SVG |
| `initExtSlider()` | s6 — sets slider min=60, max=92, value=60, resets all SVG elements |
| `tempToColor(temp)` | Returns hex colour for a temperature (blue→green→orange→red) |
| `lerpColor(a, b, t)` | Linear interpolation between two hex colours |
| `animTL()` | s1 — staggers timeline items with CSS class adds |
| `drawTherm()` | s3 — draws canvas temperature profile |
| `initAmp()` | s7 — resets amplification counter and chart |
| `runCycle()` | s7 — increments cycle and redraws chart |
| `drawAmpCurve(mx)` | s7 — canvas chart of real vs ideal amplification |
| `drawQPCR()` | s9 — canvas chart of qPCR fluorescence curves |
| `animExt()` | s6 — **dead code** (old auto-animation, replaced by initExtSlider) |
| `growStrand()` | s6 — **dead code** (used by animExt, no longer called) |

---

## Common Editing Tasks

### Add a new section

1. Add `<div class="sc" id="sN">` anywhere in the HTML body (order doesn't matter).
2. Insert `'sN'` at the desired position in the `SCS` array.
3. Update the `LABELS` array with the matching header string.
4. Add an `if (id==='sN')` branch in `onEnter()` if the section needs an entry animation.

### Move a section

Edit only the `SCS` array. Do not move HTML divs unless there is a reason to — the DOM order is irrelevant to navigation.

### Add slider to a new section

1. Add the section's `'sN'` to the slider visibility condition in `goTo()`:
   ```javascript
   if (SCS[cur] === 's4' || SCS[cur] === 's5' || SCS[cur] === 's6' || SCS[cur] === 'sN') {
   ```
2. Add the revisit re-init branch in `goTo()`:
   ```javascript
   else if (SCS[cur] === 'sN') { initNewSlider(); }
   ```
3. Add to `onEnter()`:
   ```javascript
   if (id==='sN') initNewSlider();
   ```
4. Add to `onSliderInput()`:
   ```javascript
   else if (SCS[cur] === 'sN') setNewTemp(v);
   ```
5. Write `initNewSlider()` — **always** set `slider.min`, `slider.max`, `slider.value` explicitly.
6. Write `setNewTemp(temp)` — update `#tsv`, `#tsstate`, `--tc`, `--fp`, and drive your SVG.
7. Calculate `--fp` relative to your section's own range: `((temp - min) / (max - min) * 100)%`.

### Adjust SVG positions

The s6 SVG uses `viewBox="-28 0 420 420"` (28px left bleed for the left strand). Coordinates are in SVG user units. The `<g id="s6gl">` and `<g id="s6gr">` groups receive `translateX` transforms during denaturation — all child element coordinates are in the group's local space and shift together.

Taq Taq (`tL`, `tR`) receive `translateY` transforms on top of their group's `translateX` — CSS transforms on SVG elements compose through the rendering tree.

---

## Known Dead Code

`animExt()` and `growStrand()` remain in the file but are no longer called. They were the original auto-animation for s6 before the slider was added. They are harmless and may be removed safely.

The `TEMPS` object contains `s6:{v:72, p:72, c:'#e07030'}` — this entry is never read because the `goTo()` slider visibility check intercepts s6 before the static gauge logic runs.
