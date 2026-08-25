# Implementation Plan: Cute Cat Pet App

## Overview

Implement a self-contained, single-file (`index.html`) browser app where users pet an animated cat. All logic lives in one `<script>` block using Vanilla ES2020 and Tailwind CSS CDN. Implementation proceeds bottom-up: Tailwind config → HTML shell → state/audio → reaction logic → counter UI → wiring.

---

## Tasks

- [x] 1. Set up HTML shell, Tailwind config, and custom keyframe animations
  - [x] 1.1 Create `index.html` with full-viewport gradient background, Google Fonts link, and base Tailwind CDN `<script>` tag
    - Apply pastel pink-to-cyan linear gradient to `<body>` using Tailwind utilities
    - Load "Quicksand" or "Varela Round" from Google Fonts with `sans-serif` fallback
    - _Requirements: 1.1, 1.2, 11.1_

  - [x] 1.2 Declare `window.tailwind.config` before the CDN script tag with `wiggle` and `zzz` keyframes and animation utilities
    - `wiggle` keyframe: `0%,100% translateX(0)` / `25% translateX(-10px)` / `75% translateX(10px)`; `0.5s ease-in-out infinite`
    - `zzz` keyframe: fade in, float up 30px, fade out; `2s ease-in-out infinite`
    - _Requirements: 10.1, 10.2, 10.3, 10.4_

  - [ ]* 1.3 Write unit tests for Tailwind config shape
    - Verify `window.tailwind.config.theme.extend.keyframes` contains `wiggle` and `zzz` keys
    - Verify `window.tailwind.config.theme.extend.animation` exposes `animate-wiggle` and `animate-zzz`
    - _Requirements: 10.1, 10.2, 10.3_

- [ ] 2. Build the HTML structure and cat illustration
  - [x] 2.1 Add `#app` flex-column centered layout and `#cat-container` frosted-glass bubble
    - `#cat-container`: `backdrop-blur`, semi-transparent white background, fully rounded corners, semi-transparent white border
    - Ensure square aspect ratio across 320px–2560px viewports
    - _Requirements: 1.3, 1.4, 1.5, 2.3_

  - [x] 2.2 Embed the SVG cat illustration inside `#cat-container`
    - SVG with exactly two ears, two eyes, and a body; ≤ 10 distinct shapes; fits 80×80 to 200×200 bounding box
    - Add `drop-shadow` / `box-shadow` with blur 4–16px, opacity 0.15–0.40
    - Add `id="cat-svg"`, `role="button"`, `tabindex="0"`, `cursor: pointer`
    - _Requirements: 2.1, 2.2, 2.3_

  - [-] 2.3 Add `#overlay-text` element (absolutely positioned above cat) and `#counter-pet` / `#counter-aura` frosted-glass bubbles below the container
    - Counter elements: frosted-glass style matching `#cat-container`
    - Initialize display text to `"Total Pets: 0 ✨"` and `"Aura: 0 ✨"`
    - _Requirements: 9.1, 9.2, 9.5_

- [ ] 3. Implement state object, audioEngine, and counterUI
  - [x] 3.1 Define `state` object with `petCount`, `auraCount`, and `activeReaction`
    - `{ petCount: 0, auraCount: 0, activeReaction: null }`
    - _Requirements: 3.1, 4.3, 9.1, 9.2_

  - [x] 3.2 Implement `audioEngine` with `play(key)`, `stop(key)`, and `stopAll()`
    - Pre-instantiate `Audio` objects for `meow.mp3`, `purr.mp3`, `zoom.mp3` on `DOMContentLoaded`; set `.volume = 0.5`
    - `play(key)`: reset `currentTime = 0`, call `.play()`, catch and suppress all rejections
    - _Requirements: 8.1, 8.2, 8.3, 8.4_

  - [ ]* 3.3 Write property test for audio reset before every play (Property 5)
    - **Property 5: Audio reset before every play**
    - **Validates: Requirements 8.2**

  - [-] 3.4 Implement `formatCount(n)` and `updateCounterUI()` that read `state` and re-render both counter DOM nodes within one animation frame
    - Cap display value at `999,999,999`; text format `"Total Pets: {n} ✨"` and `"Aura: {a} ✨"`
    - _Requirements: 9.1, 9.2, 9.3, 9.4_

  - [ ]* 3.5 Write property test for counter display format invariant (Property 6)
    - **Property 6: Counter display format invariant**
    - **Validates: Requirements 9.1, 9.2**

  - [ ]* 3.6 Write unit tests for `formatCount`
    - `formatCount(999_999_999)` → `"999999999"`; `formatCount(1_000_000_000)` → `"999999999"`
    - _Requirements: 9.1, 9.2_

- [~] 4. Checkpoint — Ensure state, audio, and counter pass all tests
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 5. Implement `selectReaction` and `overlayUI`
  - [~] 5.1 Implement `selectReaction(random = Math.random())` with weighted dispatch
    - `< 0.40` → `'happy'`; `< 0.70` → `'purring'`; `< 0.90` → `'sleeping'`; else `'zoomies'`
    - _Requirements: 3.2_

  - [ ]* 5.2 Write property test for reaction dispatch covering full probability space (Property 7)
    - **Property 7: Reaction weighted dispatch covers full probability space**
    - **Validates: Requirements 3.2**

  - [ ]* 5.3 Write property test for reaction selection probability distribution (Property 2)
    - **Property 2: Reaction selection probability distribution**
    - **Validates: Requirements 3.2**

  - [ ]* 5.4 Write unit tests for `selectReaction` boundary values
    - `selectReaction(0.0)` → `'happy'`; `selectReaction(0.39)` → `'happy'`; `selectReaction(0.40)` → `'purring'`; `selectReaction(0.69)` → `'purring'`; `selectReaction(0.70)` → `'sleeping'`; `selectReaction(0.89)` → `'sleeping'`; `selectReaction(0.90)` → `'zoomies'`; `selectReaction(0.99)` → `'zoomies'`
    - _Requirements: 3.2_

  - [~] 5.5 Implement `overlayUI.show(text)` and `overlayUI.hide()` on `#overlay-text`
    - `show(text)`: set `textContent`, apply `opacity-100` class
    - `hide()`: apply `transition-opacity duration-300 opacity-0`; after 300ms clear `textContent` and reset classes
    - _Requirements: 4.2, 4.4, 5.3, 5.5, 6.3, 6.4, 7.4, 7.6_

- [ ] 6. Implement the four reaction functions
  - [~] 6.1 Implement `happy(catEl, overlayEl, audio)` returning an idempotent `cancel()` closure
    - Apply `animate-bounce` for 800ms; show overlay `"YAY! (+5 Aura ✨)"`; add 5 to `state.auraCount`
    - After 800ms: remove `animate-bounce`, call `overlayUI.hide()`; update counter UI
    - `cancel()`: `clearTimeout`, remove class, hide overlay — safe to call multiple times
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5_

  - [~] 6.2 Implement `purring(catEl, overlayEl, audio)` returning an idempotent `cancel()` closure
    - Apply `animate-pulse` for 2000ms; play `purr` audio; show overlay `"Purrr... 💜"`
    - After 2000ms: remove `animate-pulse`, stop purr audio, call `overlayUI.hide()`
    - Suppress audio errors per Req 5.6; `cancel()` is idempotent
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7_

  - [~] 6.3 Implement `sleeping(catEl, overlayEl, audio)` returning an idempotent `cancel()` closure
    - Apply inline `transform: scale(0.9) rotate(6deg)` for 2000ms; inject three `💤` emoji elements with `animate-zzz` and 200ms stagger
    - Show overlay `"Zzz... 😴"`
    - After 2000ms: remove transforms, remove `💤` elements from DOM, call `overlayUI.hide()`
    - _Requirements: 6.1, 6.2, 6.3, 6.4_

  - [~] 6.4 Implement `zoomies(catEl, overlayEl, audio)` returning an idempotent `cancel()` closure
    - Apply `animate-wiggle` for 600ms (≥ 6 oscillations at 0.5s period); play `zoom` audio
    - Inject three `💨` Speed_Lines elements that animate outward and fade to transparent over 600ms
    - Show overlay `"ZOOMIES!! 💨"`
    - After 600ms: remove `animate-wiggle`, remove Speed_Lines from DOM, call `overlayUI.hide()`
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5, 7.6_

  - [ ]* 6.5 Write property test for reaction cancellation is total and idempotent (Property 3)
    - **Property 3: Reaction cancellation is total and idempotent**
    - **Validates: Requirements 3.3, 4.5, 5.7, 6.4, 7.5, 7.6**

  - [ ]* 6.6 Write property test for overlay text round-trip (Property 8)
    - **Property 8: Overlay text round-trip**
    - **Validates: Requirements 4.2, 5.3, 6.3, 7.4**

  - [ ]* 6.7 Write unit tests for aura accumulation
    - After 3 happy reactions → `auraCount === 15`
    - `auraCount` never decreases across any reaction sequence
    - _Requirements: 4.3, 9.2_

- [~] 7. Checkpoint — Ensure all reaction functions and property tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 8. Implement `handlePet` and wire everything together
  - [~] 8.1 Implement `handlePet()` click handler with synchronous cancellation of active reaction
    - Cancel `state.activeReaction` synchronously (within same event loop tick), null it out
    - Increment `state.petCount`; call `updateCounterUI()`
    - Select reaction via `selectReaction()`; store returned cancel closure in `state.activeReaction`
    - Play `meow` audio immediately after starting the reaction (Req 3.4)
    - _Requirements: 3.1, 3.3, 3.4_

  - [~] 8.2 Attach `handlePet` to `#cat-svg` click event and keyboard Enter/Space on `DOMContentLoaded`
    - `addEventListener('click', handlePet)` on `#cat-svg`
    - `addEventListener('keydown', e => { if (e.key === 'Enter' || e.key === ' ') handlePet(); })` on `#cat-svg`
    - _Requirements: 3.1, 2.3_

  - [ ]* 8.3 Write property test for pet counter increments on every click (Property 1)
    - **Property 1: Pet counter increments on every click**
    - **Validates: Requirements 3.1, 9.1, 9.3**

  - [ ]* 8.4 Write property test for aura counter accumulates correctly (Property 4)
    - **Property 4: Aura counter accumulates correctly**
    - **Validates: Requirements 4.3, 9.2, 9.4**

- [~] 9. Final checkpoint — Full integration verification
  - Ensure all tests pass, ask the user if questions arise.

---

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP delivery
- The design's `Correctness Properties` section maps 1-to-1 to the property test sub-tasks above
- All code lives in a single `index.html`; the test layer (fast-check) is a separate concern from the deliverable
- Audio errors are always suppressed silently — never surface UI feedback for missing files
- The `cancel()` closure returned by every reaction must be idempotent (safe to call multiple times)
- Tailwind config **must** be declared before the CDN `<script>` tag is evaluated (Req 10.4)

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1", "1.2"] },
    { "id": 1, "tasks": ["1.3", "2.1", "3.1"] },
    { "id": 2, "tasks": ["2.2", "3.2"] },
    { "id": 3, "tasks": ["2.3", "3.3", "3.4"] },
    { "id": 4, "tasks": ["3.5", "3.6", "5.1", "5.5"] },
    { "id": 5, "tasks": ["5.2", "5.3", "5.4", "6.1"] },
    { "id": 6, "tasks": ["6.2", "6.3", "6.4"] },
    { "id": 7, "tasks": ["6.5", "6.6", "6.7", "8.1"] },
    { "id": 8, "tasks": ["8.2"] },
    { "id": 9, "tasks": ["8.3", "8.4"] }
  ]
}
```
