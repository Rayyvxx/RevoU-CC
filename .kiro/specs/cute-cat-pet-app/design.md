# Design Document: Cute Cat Pet App

## Overview

The Cute Cat Pet App is a self-contained, single-file (`index.html`) browser application. There are no build steps, no bundler, no npm packages, and no server. All logic lives in a single `<script>` block; all styles are a combination of Tailwind utility classes (loaded via CDN) and a small inline `<style>` block for custom keyframes not expressible as Tailwind config.

### Key Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Delivery format | Single `index.html` | Req 11 — must open via `file://` with zero tooling |
| CSS framework | Tailwind CSS CDN | Req 10 — custom `wiggle` / `zzz` utilities via `window.tailwind.config` |
| JavaScript | Vanilla ES2020 | No framework dependency; DOM is tiny |
| Animations | CSS keyframes + JS class-toggling | Gives precise timing control required by Reqs 4–7 |
| Audio | HTML5 `Audio` objects pre-instantiated | Req 8 — explicit `currentTime` reset before every play call |
| State | Plain JS object (in-memory) | Pet_Counter and Aura_Counter never need to persist across reloads |

---

## Architecture

The app is structured as a **single rendering pass** followed by an **event-driven reaction loop**.

```
┌─────────────────────────────────────────────────────────────────┐
│  index.html                                                     │
│                                                                 │
│  ┌──────────────┐   DOMContentLoaded   ┌────────────────────┐  │
│  │  HTML Shell  │ ──────────────────▶  │  init()            │  │
│  │  (structure) │                      │  - audioEngine     │  │
│  └──────────────┘                      │  - counterState    │  │
│                                        │  - attachListeners │  │
│  ┌──────────────┐                      └────────┬───────────┘  │
│  │  Tailwind    │                               │               │
│  │  CDN + config│                        click  │               │
│  └──────────────┘                               ▼               │
│                                        ┌────────────────────┐  │
│  ┌──────────────┐                      │  handlePet()       │  │
│  │  Audio files │                      │  - incrementPet    │  │
│  │  (optional)  │                      │  - selectReaction  │  │
│  └──────────────┘                      │  - cancelActive    │  │
│                                        │  - startReaction   │  │
│                                        └────────┬───────────┘  │
│                                                 │               │
│                           ┌─────────────────────┼──────────┐   │
│                           ▼         ▼           ▼          ▼   │
│                        happy()  purring()  sleeping()  zoomies()│
│                           │         │           │          │    │
│                           └─────────┴───────────┴──────────┘   │
│                                          │                      │
│                                          ▼                      │
│                                  updateCounterUI()              │
└─────────────────────────────────────────────────────────────────┘
```

### Module Breakdown (all within one `<script>` block)

| Module | Responsibility |
|---|---|
| `audioEngine` | Pre-instantiates `Audio` objects; exposes `play(key)` with error suppression |
| `state` | Holds `petCount`, `auraCount`, `activeReaction` (null or `{ cancel() }`) |
| `reactionDispatch` | Weighted random selection → calls the correct reaction function |
| `reactions` | Four pure functions: `happy`, `purring`, `sleeping`, `zoomies` — each returns a `cancel()` closure |
| `counterUI` | Reads `state` and updates the two counter DOM nodes |
| `overlayUI` | Manages Overlay_Text lifecycle (show / fade-out) |
| `speedLinesUI` | Manages Speed_Lines (💨) and Zzz (💤) emoji DOM elements |

---

## Components and Interfaces

### HTML Structure

```html
<body>                                    <!-- gradient background (Tailwind) -->
  <div id="app">                          <!-- flex column, centered viewport -->

    <div id="cat-container">             <!-- frosted-glass bubble -->
      <svg id="cat-svg">…</svg>          <!-- Cat_Element — the clickable target -->
      <div id="overlay-text"></div>      <!-- Overlay_Text — absolutely positioned -->
      <!-- speed lines / zzz emojis injected here dynamically -->
    </div>

    <div id="counter-pet">Total Pets: 0 ✨</div>
    <div id="counter-aura">Aura: 0 ✨</div>

  </div>
</body>
```

### audioEngine

```js
// Interface
audioEngine.play(key)        // key: 'meow' | 'purr' | 'zoom'
audioEngine.stop(key)        // resets currentTime, pauses
audioEngine.stopAll()        // stops every registered audio object
```

- On `DOMContentLoaded`: instantiates `new Audio('meow.mp3')`, `new Audio('purr.mp3')`, `new Audio('zoom.mp3')`; sets `.volume = 0.5` on each.
- `play(key)`: sets `.currentTime = 0`, calls `.play()`, catches and suppresses any rejection.
- `stop(key)`: calls `.pause()`, sets `.currentTime = 0`.

### reactionDispatch

```js
// Weighted random — returns one of 'happy' | 'purring' | 'sleeping' | 'zoomies'
function selectReaction(random = Math.random()) {
  if (random < 0.40) return 'happy';
  if (random < 0.70) return 'purring';
  if (random < 0.90) return 'sleeping';
  return 'zoomies';
}
```

The `random` parameter is injectable for testability.

### Reaction Contract

Every reaction function returns a cancel closure that must be idempotent:

```js
// Each reaction: (catEl, overlayEl, audioEngine) => cancel()
function happy(catEl, overlayEl, audio) {
  // ... setup ...
  return function cancel() { /* teardown — safe to call multiple times */ };
}
```

### State Object

```js
const state = {
  petCount: 0,
  auraCount: 0,
  activeReaction: null  // null | { cancel: Function }
};
```

### handlePet (main click handler)

```js
function handlePet() {
  // 1. Cancel active reaction synchronously (within same event loop tick)
  if (state.activeReaction) {
    state.activeReaction.cancel();
    state.activeReaction = null;
  }
  // 2. Increment counter
  state.petCount += 1;
  updateCounterUI();
  // 3. Select and start new reaction
  const name = selectReaction();
  const reactionFn = reactions[name];
  state.activeReaction = { cancel: reactionFn(catEl, overlayEl, audioEngine) };
  // 4. Play meow.mp3 immediately (Req 3.4)
  audioEngine.play('meow');
}
```

---

## Data Models

### AppState

```js
{
  petCount:  number,   // integer ≥ 0; display capped at 999,999,999
  auraCount: number,   // integer ≥ 0; display capped at 999,999,999
  activeReaction: {    // null when idle
    cancel: () => void // idempotent teardown
  } | null
}
```

### ReactionDescriptor (internal, per-reaction)

```js
{
  name:          string,   // 'happy' | 'purring' | 'sleeping' | 'zoomies'
  overlayText:   string,   // e.g. 'YAY! (+5 Aura ✨)'
  duration:      number,   // ms — total animation window
  auraPoints:    number,   // 0 unless happy (5)
  audioKey:      string,   // 'meow' | 'purr' | 'zoom'
  animationClass: string,  // Tailwind / custom CSS class applied to cat
}
```

Static reaction descriptors:

| Reaction | overlayText | duration | auraPoints | audioKey | animationClass |
|---|---|---|---|---|---|
| happy | `"YAY! (+5 Aura ✨)"` | 800ms | 5 | `zoom`* | `animate-bounce` |
| purring | `"Purrr... 💜"` | 2000ms | 0 | `purr` | `animate-pulse` |
| sleeping | `"Zzz... 😴"` | 2000ms | 0 | — | (inline transform) |
| zoomies | `"ZOOMIES!! 💨"` | 600ms | 0 | `zoom` | `animate-wiggle` |

*Meow always plays first (Req 3.4); the reaction-specific audio is layered immediately after.

### AudioEntry

```js
{
  key:    string,       // 'meow' | 'purr' | 'zoom'
  src:    string,       // e.g. 'meow.mp3'
  object: HTMLAudioElement
}
```

### OverlayLifecycle

```
HIDDEN → VISIBLE → FADING → HIDDEN
```
- VISIBLE: class `opacity-100` applied
- FADING: class `transition-opacity duration-300 opacity-0` applied after animation duration
- HIDDEN: element `textContent` cleared, classes reset

### Tailwind Config Shape

```js
window.tailwind = {
  config: {
    theme: {
      extend: {
        keyframes: {
          wiggle: {
            '0%, 100%': { transform: 'translateX(0)' },
            '25%':      { transform: 'translateX(-10px)' },
            '75%':      { transform: 'translateX(10px)' }
          },
          zzz: {
            '0%':   { opacity: '0', transform: 'translateY(0)' },
            '50%':  { opacity: '1' },
            '100%': { opacity: '0', transform: 'translateY(-30px)' }
          }
        },
        animation: {
          wiggle: 'wiggle 0.5s ease-in-out infinite',
          zzz:    'zzz 2s ease-in-out infinite'
        }
      }
    }
  }
};
```

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Pet counter increments on every click

*For any* sequence of N clicks on the Cat_Element, the final `state.petCount` SHALL equal N, regardless of which reactions were triggered or whether previous reactions were still active.

**Validates: Requirements 3.1, 9.1, 9.3**

---

### Property 2: Reaction selection probability distribution

*For any* sufficiently large set of random numbers uniformly distributed in [0, 1), the proportion of calls to `selectReaction` that return `'happy'` SHALL converge to 40%, `'purring'` to 30%, `'sleeping'` to 20%, and `'zoomies'` to 10% — with at most one reaction returned per call.

**Validates: Requirements 3.2**

---

### Property 3: Reaction cancellation is total and idempotent

*For any* active reaction, calling its `cancel()` function SHALL remove all DOM elements it added (Overlay_Text, Speed_Lines, Zzz emojis), remove all CSS animation classes it applied, and stop its associated audio — and calling `cancel()` a second time SHALL produce no error and no additional DOM changes.

**Validates: Requirements 3.3, 4.5, 5.7, 6.4, 7.5, 7.6**

---

### Property 4: Aura counter accumulates correctly

*For any* sequence of reactions applied, the final `state.auraCount` SHALL equal 5 × (number of Happy_Reactions triggered), and `state.auraCount` SHALL never decrease.

**Validates: Requirements 4.3, 9.2, 9.4**

---

### Property 5: Audio reset before every play

*For any* call to `audioEngine.play(key)`, the `.currentTime` of the corresponding `Audio` object SHALL be set to `0` before `.play()` is invoked, regardless of the object's prior playback state.

**Validates: Requirements 8.2**

---

### Property 6: Counter display format invariant

*For any* non-negative integer value of `petCount` or `auraCount`, the rendered text of the Pet_Counter element SHALL match the pattern `"Total Pets: {n} ✨"` and the Aura_Counter element SHALL match `"Aura: {a} ✨"`, where the displayed number is `min(value, 999_999_999)`.

**Validates: Requirements 9.1, 9.2**

---

### Property 7: Reaction weighted dispatch covers full probability space

*For any* value `r` in [0, 1), `selectReaction(r)` SHALL return exactly one reaction name from `{ 'happy', 'purring', 'sleeping', 'zoomies' }` and SHALL never return `undefined` or throw.

**Validates: Requirements 3.2**

---

### Property 8: Overlay text round-trip

*For any* reaction triggered, the Overlay_Text shown during that reaction SHALL exactly match the string specified for that reaction in the requirements (`"YAY! (+5 Aura ✨)"`, `"Purrr... 💜"`, `"Zzz... 😴"`, `"ZOOMIES!! 💨"`), and after the reaction completes the Overlay_Text SHALL be absent from the visible DOM.

**Validates: Requirements 4.2, 5.3, 6.3, 7.4**

---

## Error Handling

| Failure Scenario | Handling Strategy |
|---|---|
| `meow.mp3` / `purr.mp3` / `zoom.mp3` missing | `Audio.play()` rejects; caught silently; reaction continues (Req 8.3, 11.3) |
| Browser autoplay policy blocks audio | Same `.play()` rejection path — swallowed silently |
| Tailwind CDN fails to load | Visual styling degrades; all interactive JS logic continues unaffected (Req 11.4) |
| Google Fonts CDN fails | Browser falls back to `sans-serif` system font (Req 1.2) |
| Rapid successive clicks | `handlePet` synchronously cancels the active reaction before starting the new one — guaranteed within the same event loop tick (Req 3.3) |
| `cancel()` called on already-cancelled reaction | All teardown code guards with `if (timer) clearTimeout(timer)` style checks; idempotent by design |
| Counter exceeds 999,999,999 | Internal `state.petCount` / `state.auraCount` continue to grow accurately; only the displayed string is clamped |

---

## Testing Strategy

### Overview

Because this is a single-file Vanilla JS / HTML app with no build step, the test layer is a separate concern from the deliverable. Tests target the **pure logic functions** extracted from the script block.

### Unit Tests (example-based)

Target the following specific scenarios with concrete examples:

- `selectReaction(0.0)` → `'happy'`
- `selectReaction(0.39)` → `'happy'`
- `selectReaction(0.40)` → `'purring'`
- `selectReaction(0.69)` → `'purring'`
- `selectReaction(0.70)` → `'sleeping'`
- `selectReaction(0.89)` → `'sleeping'`
- `selectReaction(0.90)` → `'zoomies'`
- `selectReaction(0.99)` → `'zoomies'`
- Counter cap: `formatCount(999_999_999)` → `"999999999"`; `formatCount(1_000_000_000)` → `"999999999"`
- Aura accumulation: after 3 happy reactions → `auraCount === 15`

### Property-Based Tests

Use a library such as [fast-check](https://fast-check.dev) (JavaScript) for browser-runnable property tests. Each test runs a minimum of **100 iterations**.

Tag format: `// Feature: cute-cat-pet-app, Property {N}: {property_text}`

#### Property 1 — Pet counter increments on every click
```
// Feature: cute-cat-pet-app, Property 1: Pet counter increments on every click
fc.assert(fc.property(fc.integer({ min: 1, max: 500 }), (n) => {
  const state = { petCount: 0 };
  for (let i = 0; i < n; i++) state.petCount += 1;
  return state.petCount === n;
}));
```

#### Property 2 — Reaction selection probability distribution
```
// Feature: cute-cat-pet-app, Property 2: Reaction selection probability distribution
fc.assert(fc.property(fc.float({ min: 0, max: 0.9999 }), (r) => {
  const result = selectReaction(r);
  return ['happy', 'purring', 'sleeping', 'zoomies'].includes(result);
}));
```

#### Property 3 — Reaction cancellation is total and idempotent
```
// Feature: cute-cat-pet-app, Property 3: Reaction cancellation is total and idempotent
// DOM-based test: for any reaction, cancel() twice leaves DOM in clean state
fc.assert(fc.property(fc.constantFrom('happy', 'purring', 'sleeping', 'zoomies'), (name) => {
  const { catEl, overlayEl } = buildTestDOM();
  const cancel = reactions[name](catEl, overlayEl, mockAudio);
  cancel();
  cancel(); // second call must not throw or mutate DOM further
  return overlayEl.textContent === '' && catEl.classList.length === baseClassCount;
}));
```

#### Property 4 — Aura counter accumulates correctly
```
// Feature: cute-cat-pet-app, Property 4: Aura counter accumulates correctly
fc.assert(fc.property(fc.integer({ min: 0, max: 200 }), (happyCount) => {
  let aura = 0;
  for (let i = 0; i < happyCount; i++) aura += 5;
  return aura === happyCount * 5 && aura >= 0;
}));
```

#### Property 5 — Audio reset before every play
```
// Feature: cute-cat-pet-app, Property 5: Audio reset before every play
fc.assert(fc.property(fc.constantFrom('meow', 'purr', 'zoom'), fc.float({ min: 0, max: 100 }), (key, priorTime) => {
  const mockAudioObj = { currentTime: priorTime, play: () => Promise.resolve() };
  audioEngine._objects[key] = mockAudioObj;
  audioEngine.play(key);
  return mockAudioObj.currentTime === 0;
}));
```

#### Property 6 — Counter display format invariant
```
// Feature: cute-cat-pet-app, Property 6: Counter display format invariant
fc.assert(fc.property(fc.integer({ min: 0 }), (n) => {
  const display = formatCount(n);
  return Number(display) === Math.min(n, 999_999_999);
}));
```

#### Property 7 — Reaction dispatch covers full probability space
```
// Feature: cute-cat-pet-app, Property 7: Reaction weighted dispatch covers full probability space
fc.assert(fc.property(fc.float({ min: 0, max: 0.9999 }), (r) => {
  const valid = ['happy', 'purring', 'sleeping', 'zoomies'];
  const result = selectReaction(r);
  return valid.includes(result);
}));
```

#### Property 8 — Overlay text round-trip
```
// Feature: cute-cat-pet-app, Property 8: Overlay text round-trip
fc.assert(fc.property(fc.constantFrom('happy', 'purring', 'sleeping', 'zoomies'), (name) => {
  const expected = overlayTextMap[name];
  const { overlayEl } = buildTestDOM();
  const cancel = reactions[name](catEl, overlayEl, mockAudio);
  const textDuringReaction = overlayEl.textContent;
  cancel();
  return textDuringReaction === expected && overlayEl.textContent === '';
}));
```

### Integration / Smoke Tests

- Open `index.html` via `file://` in a real browser; verify no console errors on load.
- Click the cat 10 times; verify Pet_Counter increments each time.
- Remove `meow.mp3`; verify the app still loads and animations still run.
- Throttle CPU in DevTools; click rapidly; verify no overlapping animations or duplicate Overlay_Text nodes.

### Accessibility Notes

The `Cat_Element` must carry `role="button"` and `tabindex="0"` to be keyboard-accessible, allowing Enter/Space to trigger the same `handlePet` handler. Colour contrast of Overlay_Text against the frosted-glass background should be verified manually with a contrast checker.
