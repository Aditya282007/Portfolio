# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project overview

This repository is a small, static 3D-style scroll animation demo built with plain HTML, CSS, and JavaScript. It uses:
- A `<canvas>` element to render a frame-by-frame image sequence (a 3D character animation)
- Locomotive Scroll for smooth scrolling and virtual scrolling behavior
- GSAP + ScrollTrigger for scroll-driven animation and pinning of sections

There is no build system, bundler, or test framework configured; everything runs directly in the browser.

## How to run and develop locally

### Quick preview

The simplest way to preview the project is to open `index.html` in a browser:
- On most systems you can double-click `index.html`, or open it via your browser's "Open File" dialog.

Because the project uses external CDN scripts and smooth-scrolling behavior, it is often more reliable to serve the files over HTTP:

- With Python installed:
  - From the repo root:
    - `python -m http.server 8000`
  - Then open `http://localhost:8000/index.html` in your browser.

There are no configured build, lint, or test commands in this repository (no `package.json` or tooling configuration). Development is done by directly editing `index.html`, `style.css`, and `script.js` and refreshing the browser.

## External libraries and runtime expectations

- **Locomotive Scroll** is loaded via CDN in `index.html` and is used to provide smooth scrolling on the `#main` container.
- **GSAP** and **GSAP ScrollTrigger** are also loaded via CDN and drive all scroll-based animations and pinning.
- **Image sequence frames**:
  - `script.js` expects a sequence of PNG images named `male0001.png` through `male0300.png` in the project root (relative to `index.html`).
  - If you add, remove, or rename frames, update the `files()` function and/or `frameCount` accordingly.

## High-level architecture and structure

### Layout (`index.html`)

- The main scrolling container is the `#main` `<div>`, which Locomotive Scroll takes over as its scroller.
- Inside `#main` there are four full-screen sections:
  - `#page`: Intro section containing the marquee text loop and the `<canvas>` used for the 3D image-sequence animation.
  - `#page1`, `#page2`, `#page3`: Subsequent full-screen text sections that are pinned on scroll to create a stepped narrative effect.
- A fixed `#nav` bar sits above `#main` with the title and a date button.
- All JavaScript is loaded at the bottom of `body`:
  - Locomotive Scroll CDN script
  - GSAP CDN script
  - ScrollTrigger CDN script
  - Local `script.js`

### Styling (`style.css`)

- Global styles set `html, body` to full viewport size and reset margins/padding.
- `#main` is relatively positioned and overflow-hidden, which works with Locomotive Scroll's virtual scrolling.
- Each `#page*` (`#page`, `#page1`, `#page2`, `#page3`) is a `100vh x 100vw` section with a shared background color, forming vertical scroll "slides".
- The `<canvas>` is sized to never exceed the viewport (`max-width: 100vw`, `max-height: 100vh`) and is layered above the background (`z-index: 9`).
- The `#loop` element in `#page` is an absolutely positioned horizontal marquee:
  - Uses `white-space: nowrap` and a keyframed `anim` animation on its `<h1>` children to translate content horizontally across the screen.
  - `<span>` elements inside `#loop h1` use `-webkit-text-stroke` and transparent fill for outlined text.
- Subsequent sections (`#page1`, `#page2`, `#page3`) use absolutely positioned text blocks (`#right-text`, `#left-text`, `#text1`, `#text2`, `#text3`) for layout, with typography tuned for large headings and muted subtext.

### Scroll and animation logic (`script.js`)

1. **Locomotive Scroll + ScrollTrigger wiring**
   - `locomotive()`:
     - Registers the ScrollTrigger plugin with GSAP.
     - Creates a `new LocomotiveScroll({ el: document.querySelector("#main"), smooth: true })` instance.
     - Forwards Locomotive's internal scroll events to ScrollTrigger via `locoScroll.on("scroll", ScrollTrigger.update)`.
     - Configures `ScrollTrigger.scrollerProxy("#main", { ... })` so that ScrollTrigger uses Locomotive's virtual scroll position as its source of truth.
       - `scrollTop(value)` delegates scroll reads/writes to `locoScroll`.
       - `getBoundingClientRect()` always returns a viewport-sized rect.
       - `pinType` is chosen based on whether `#main` uses transforms, switching between `transform` and `fixed` pinning to avoid jitter.
     - Hooks `ScrollTrigger.addEventListener("refresh", () => locoScroll.update())` and calls `ScrollTrigger.refresh()` to synchronize on load.
   - `locomotive()` is executed immediately so the virtual scroller is available before any ScrollTriggers are created.

2. **Canvas and image sequence setup**
   - The global `canvas` and 2D `context` references are obtained from the single `<canvas>` in `#page`.
   - Initial canvas size is set to `window.innerWidth` and `window.innerHeight`.
   - A `resize` event listener resizes the canvas to the current viewport and calls `render()` so the current frame is redrawn to fit.
   - `files(index)` returns a path string for each frame from a hard-coded multiline template literal of file names (`./male0001.png` … `./male0300.png`).
   - `frameCount` is set to `300` and must stay consistent with the number of entries in `files()`.
   - An `images` array is populated with `Image` instances whose `src` values come from `files(i)`.
   - `imageSeq` is a simple GSAP-driven state object `{ frame: 1 }` used to track the current frame index.

3. **Scroll-driven frame animation**
   - A GSAP tween animates `imageSeq.frame` from `0` to `frameCount - 1`:
     - `snap: "frame"` forces integer frame indices.
     - `scrollTrigger` is configured with:
       - `trigger: '#page>canvas'`
       - `start: 'top top'`
       - `end: '600% top'` — the animation progresses over six viewport heights of scroll.
       - `scrub: 0.15` to smoothly link scroll position and frame index.
       - `scroller: '#main'` so it uses the Locomotive-controlled container.
     - `onUpdate: render` redraws the current frame whenever the scroll position changes.
   - `images[1].onload = render;` ensures the sequence draws once a frame has loaded, so the canvas is not blank initially.

4. **Rendering and scaling logic**
   - `render()` simply calls `scaleImage(images[imageSeq.frame], context)`.
   - `scaleImage(img, ctx)`:
     - Computes horizontal and vertical scale ratios to fit the image into the canvas.
     - Uses `Math.max(hRatio, vRatio)` to cover the entire canvas while preserving aspect ratio (no letterboxing).
     - Centers the scaled image via `centerShift_x` / `centerShift_y` and draws it with `drawImage()`.

5. **Section pinning**
   - A ScrollTrigger is created to pin the canvas section itself:
     - `trigger: '#page>canvas'`, `pin: true`, `start: 'top top'`, `end: '600% top'`, `scroller: '#main'`.
     - This keeps the canvas fixed while the scroll-driven frame animation advances.
   - Additional `gsap.to()` calls with `scrollTrigger` configs pin the text sections:
     - `#page1`, `#page2`, `#page3` each have a ScrollTrigger with `pin: true`, `scroller: '#main'`, and `start: 'top top'`, `end: 'bottom top'`.
     - This creates a sequence of full-screen pinned panels that the user scrolls through after the canvas animation.

## When extending or modifying the project

- If you add more sections that participate in Locomotive Scroll + ScrollTrigger behavior, make sure you:
  - Place them inside `#main`.
  - Set `scroller: '#main'` on any new ScrollTriggers so they stay synchronized with Locomotive.
- When changing the image sequence (different character, resolution, or frame count):
  - Keep `frameCount` and the number of entries in `files()` in sync.
  - Ensure images are sized such that scaling to full-screen via `scaleImage` still looks acceptable.
- If you swap from CDN scripts to local copies or a bundler, ensure GSAP, ScrollTrigger, and Locomotive Scroll are still loaded before `script.js` runs, and that `locomotive()` is called before any ScrollTriggers are defined.

### Helper: collect image URLs from a HAR file

You mentioned this snippet for extracting image URLs from a browser HAR file (e.g., from DevTools). It iterates over HAR entries, filters responses whose MIME type starts with `image/`, and logs all image request URLs joined by newlines:

```js
var imageUrls = [];
har.log.entries.forEach(function (entry) {
  if (entry.response.content.mimeType.indexOf("image/") !== 0) return;
  imageUrls.push(entry.request.url);
});
console.log(imageUrls.join('\n'));
```

This is useful if you capture a HAR of the original site and need to quickly list all image sequence URLs (for example, before downloading or renaming them into `male0001.png`, `male0002.png`, etc.).
