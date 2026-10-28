# AGENTS.md

Vessel: single-file static landing site concept (Three.js r128 + GSAP 3.12.5/ScrollTrigger from cdnjs, Google Fonts). Live on Vercel.

## Commands
- No install, build, lint or test setup. No package.json.
- Preview: `npx serve .` or `python -m http.server` from the repo root, then open the printed URL.
- Deploy: push to `main`; Vercel serves the repo root statically (no build step).

## Where things live
- `index.html` — everything: CSS in `<style>`, scripts at the bottom.
  - Background grid: `#bgGrid` / `#bgGridCanvas`, its own Three.js scene, bend in the vertex shader, grid pattern in the fragment shader.
  - Hero physics: icosahedra scene (`const COUNT = 58`), drag-to-throw, wall splashes.
  - Vision: scroll-scrubbed frame sequence on `#visionCanvas` (`FRAME_COUNT = 96`).
  - Cursor, magnetic links, work-row previews, service-card tilt, footer reveal.
- `assets/vision-frames/frame_00.png` … `frame_95.png` — frame sequence (names must stay zero-padded to 2 digits).
- `vessel-studio-notes_2.md` — design notes and change log; read the latest sections first, early ones are superseded.

## Conventions
- Keep it one self-contained HTML file with no build step; load libraries from CDN with pinned versions.
- The background grid scene is deliberately decoupled from the hero scene; don't merge their cameras or render loops.
- Respect `prefers-reduced-motion` and `(pointer:coarse)` checks when adding motion.
- Content (studio name, work items, links) is placeholder.

## Do not touch / commit
- Keep the file named `index.html` at the root; Vercel 404s without it (this happened before).
- Don't commit IDE files (`.idea/` is currently committed by mistake), `.vercel/`, or large source videos.
- Adding or removing frames requires updating `FRAME_COUNT`.

## Gotchas
- Prefer a local server over `file://` so asset loading matches production.
- Three.js is pinned to r128 (global `THREE` build); newer releases dropped that build, so don't bump casually.

Humans: see README.md
