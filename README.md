# Orbe

A weekend WebXR experiment for Meta Quest passthrough AR: a plasma orb that floats
in front of you, follows your hand, gets occluded by it realistically, and — when
you close your fist around it — shatters and scans your room, then reforms as a
tiny, resizable model of it that lives inside the orb from then on.

**Live demo:** https://edgarant.github.io/orbe/ — open it in the Meta Quest Browser.

## Why this exists

The goal was to see something of my own floating in my room, iterated entirely
through code with an AI assistant instead of a game-engine editor. Unity was
deliberately ruled out: too much of the workflow there is clicking through a
visual editor. WebXR gives the same result — 3D content composited over the
real world through the headset's passthrough cameras — as a single web page,
so every change is just editing a file, committing, and refreshing the browser
on the headset. No APK, no native SDK, no developer mode, no cable.

## How it's built

- **One file.** The entire app is `index.html` — markup, styles, and a single
  `<script type="module">` with all the logic. No build step.
- **three.js from a CDN**, pinned to a fixed version (`0.169.0`), loaded via an
  [import map](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/script/type/importmap)
  in the page itself — no `npm install`, no bundler, no local dependencies.
  Opening the file (once pushed) is instant.
- **Hosted on GitHub Pages**, which gives it the HTTPS URL that WebXR requires,
  without running any server.
- **No addons.** WebXR session setup is done by hand
  (`navigator.xr.requestSession('immersive-ar', ...)` +
  `renderer.xr.setSession(...)`) instead of three.js's `ARButton` helper, and
  hand tracking reads `renderer.xr.getHand(n).joints[...]` directly instead of
  going through `XRHandModelFactory`. Nothing is loaded from an external asset
  CDN besides three.js itself.
- **No lights, no post-processing.** Everything on the orb uses
  `MeshBasicMaterial`/`PointsMaterial` with additive blending, so it reads as a
  glowing, self-lit object regardless of the real room's lighting.
- **All the tunable numbers live together** at the top of the script, in one
  `CONFIG` object, grouped by feature and commented — colors, sizes, gesture
  thresholds, timings, all in one place instead of scattered through the code.
- **No per-frame allocation.** The render loop reuses pre-allocated `Vector3`s
  and typed arrays every frame instead of creating new objects, since this
  runs on a mobile chip in stereo at 72–90 fps.

## What's in it

### The orb

A small solid core, an additive-blended glow halo around it, and an outer
"plasma" shell of ~200 individually-animated circular points distributed
evenly on a sphere (a
[Fibonacci sphere](https://en.wikipedia.org/wiki/Fibonacci_lattice)), each
wobbling in and out around its own resting radius with its own phase and
speed. No shaders — the wobble and the point-sprite circle texture are both
plain per-point math and a small canvas-generated texture, respectively.

### Hand tracking and following

The orb hovers ~10cm above a knuckle joint of your right hand while held
(`middle-finger-phalanx-proximal` — closer to the palm's center than the
metacarpal joint, which sits almost at the wrist), smoothed with
frame-rate-independent exponential lerp so it doesn't feel rigid at any
headset refresh rate. The *right* hand specifically is identified via the
WebXR `"connected"` event's `handedness`, not by array index — three.js
doesn't guarantee which of `getHand(0)`/`getHand(1)` is which side.

### Real occlusion

WebXR has no built-in way to hide virtual content behind your real hand — by
default, rendered content always draws on top of the passthrough video. To
fake real depth, every hand joint gets an invisible sphere
(`colorWrite: false`, so it writes to the depth buffer but draws no pixels)
sized roughly to that joint's real bone thickness. When your real hand passes
in front of the orb, its invisible proxy wins the depth test and the orb's
pixels behind it get discarded, revealing the real passthrough hand
underneath. The thumb and index fingertips — the pinch point — are excluded
from this, or the orb would vanish from view exactly when you're trying to
aim a pinch at it.

### Gestures (right hand)

| Gesture | Effect |
|---|---|
| Pinch (thumb + index) near the floating orb | Grab it — it now follows your hand |
| Pinch again while holding it | Let go — it resumes floating right where you released it |
| Close your fist while holding it | Shatter it and scan the room (see below) |
| Pinch with **both hands** near a floating orb that already contains a room model | Resize that model — the ratio between your hands' current distance and their distance when you started the pinch drives the scale |

Pinch and fist are both simple geometric heuristics on joint positions
(thumb-to-index distance; average fingertip-to-wrist distance), not machine
learning — good enough for a deliberate gesture, cheap to compute every
frame.

### Shatter → scan → miniature

Closing your fist around the held orb triggers a short sequence:

1. **Shatter** — the orb's own plasma pixels fly outward with some randomness
   and gravity while the core and halo scale down to nothing.
2. **Scan** — a laser-colored ring sweeps from floor to ceiling. It's built
   from whatever the Quest's own room understanding already knows, read once
   via the WebXR `plane-detection` and `mesh-detection` optional features
   (`frame.detectedPlanes` / `frame.detectedMeshes`) — this is *not* a live
   scan; the headset already captured the room during its own one-time Space
   Setup, and the page is just reading that data and revealing it with a
   sweep animation for effect. Detected surfaces fade in, teal and
   translucent, as the sweep passes their height.
3. **Collapse** — instead of fading away, the whole revealed room shrinks
   toward the point where the orb broke, position and scale both
   interpolated around that single pivot so the layout stays proportional —
   a real "zoom to miniature," not everything converging to a point.
4. **Reform** — the shrunk surfaces are reparented onto the orb with
   `Object3D.attach()` (which preserves their world transform across the
   parent change), so the tiny room model now travels permanently with the
   orb — carrying it, floating, hand occlusion all apply to the model for
   free through the same parent/child transform.

A later fist-shatter clears out whatever model is currently inside before the
next scan builds a fresh one.

## Known limitations

- **Room scanning needs Quest's Space Setup done first.** The page cannot
  scan your room itself — WebXR's plane/mesh detection only exposes what the
  headset's OS already captured. If Space Setup hasn't been run, the laser
  sweep still plays but nothing lights up.
- **Furniture and door detection isn't guaranteed.** Walls/floor/ceiling
  detect reliably; smaller objects only show up if Space Setup recorded them
  as separate surfaces and the browser exposes that to WebXR.
- Reforming the orb after a scan is instant, not animated.
- The fist and pinch thresholds (`FIST_TIP_TO_WRIST_THRESHOLD`,
  `PINCH_DISTANCE`) are tuned by feel, not calibrated to hand size.

## Workflow

Everything is edited directly in `index.html` and pushed to `main`; GitHub
Pages redeploys automatically within seconds. There's no local dev server —
the loop is edit → commit → push → refresh the page in the headset.
