# Claude Code Prompt — "Break the Wall" Entry Sequence (3D)

## Project
Build a 3D brick wall that the user explodes by clicking a button. Behind the wall is a campaign website. After the explosion, the camera pushes forward through the cavity and crossfades to the campaign site.

This is the **entry sequence** for a campaign site built in Framer. The final deliverable will be a **Framer Code Component**, but we will build in two stages:

- **Stage 1 (build first, iterate here):** A standalone `index.html` file I can open directly in a browser to iterate on visuals, physics tuning, and timing. All assets and dependencies loaded via CDN. No build step.
- **Stage 2 (only after I confirm Stage 1 looks right):** Convert Stage 1 into a single-file Framer Code Component (TypeScript, default export, property controls, no external CSS). **Do not start Stage 2 until I explicitly say "Stage 1 is approved, proceed to Stage 2."**

## The big picture
The user lands on the campaign site. The first thing they see is a wall of bricks filling the viewport, with a button that says "Break the Wall." When they click it, the wall fractures and explodes outward using real-time physics. As the bricks fly away, the camera dollies forward through the cavity, and the scene crossfades into the campaign site below.

Tone: dramatic, weighty, satisfying. The explosion should feel like a *moment* — not a quick effect. Aim for ~2 seconds from click to "user is now inside the site."

## Tech stack (Stage 1)
- **HTML + ES modules**, no build step, all dependencies via CDN (use `https://esm.sh` or `https://unpkg.com`).
- **Three.js** for 3D rendering.
- **@react-three/rapier** is **not** used in Stage 1 (it's React-only). Use **Rapier directly** via `@dimforge/rapier3d-compat` from esm.sh. This is the same physics engine, just without the React wrapper.
- **React** is **not** used in Stage 1. Plain Three.js + plain JS. (We'll wrap in React for the Framer component in Stage 2.)
- The single asset: `Brick.glb` — a small (12 KB) textured brick mesh. Treat this as **one brick**, not a wall. The code must instance it many times to build the wall.

## What to build

### 1. Scene setup
- Full-viewport `<canvas>`. Three.js renderer, perspective camera, dark near-black background.
- Camera positioned facing the wall head-on, framed so the wall fills the viewport with a small margin.
- Lighting: one directional light (key, warm-ish, casting soft shadows), one ambient light (cool, low intensity), one rim/fill light from behind to give brick edges some separation. Tune for "gritty construction site at dusk," not "cheerful daylight."
- Enable shadows (PCF soft).

### 2. Build the wall from the brick GLB
- Load `Brick.glb` once using `GLTFLoader`.
- Extract the brick mesh and its material from the loaded GLB.
- Construct a wall by **instancing** this mesh in a grid: roughly **20 columns × 12 rows** of bricks, with a running-bond pattern (every other row offset by half a brick width). Make rows/columns property-configurable (see Property Controls below).
- Use `THREE.InstancedMesh` for performance — all bricks share one draw call. (Each brick will need its own physics body though; instanced rendering doesn't preclude per-instance physics.)
- Add a thin dark mortar plane behind the bricks so gaps don't show the background through.
- The wall sits at `z = 0`. Camera starts at roughly `z = 8` (tune to taste so the wall fills the frame).

### 3. Physics setup (Rapier)
- Initialize a Rapier world with gravity `(0, -9.81, 0)`.
- For each brick instance, create a **dynamic rigid body** with a cuboid collider matching the brick's bounding box. Initially, set each body to **kinematic** (or use a constraint) so the wall holds together and doesn't fall under gravity before the explosion.
- Add a static ground plane below the wall so debris has something to land on (visible or not — your call, but it should exist for physics).

### 4. The "Break the Wall" button
- Render a button as an HTML overlay (positioned absolutely over the canvas, centered horizontally, ~70% down the viewport).
- Style: bold, slightly rough/industrial. White text on dark semi-transparent background, or whatever reads as "do this." Subtle pulse animation to draw the eye.
- The button should fade in ~500ms after the wall finishes loading.
- On click: fire the explosion sequence (next section), then hide the button.

### 5. The explosion (this is the moment — make it good)
On button click:

1. **Convert all brick rigid bodies from kinematic to dynamic** so they respond to forces and gravity.
2. **Apply an outward impulse** to each brick. The impulse direction is computed per-brick as the vector from a chosen "explosion origin" (center of the wall, slightly behind it on the z-axis, e.g. `(0, 0, -0.5)`) to the brick's position, normalized, then scaled by an impulse magnitude. Bricks closer to the origin get larger impulses; bricks further away get smaller ones (inverse falloff or just linear — tune for feel).
3. **Add randomized angular impulse** to each brick so they tumble, not just translate.
4. **Add slight randomization to the linear impulse** (±15% magnitude, ±10° direction) so the explosion doesn't look mathematically uniform.
5. **Trigger a screen shake**: animate the camera position with a small random offset (±0.05 units) for ~300ms, easing out.
6. **Trigger a dust burst**: spawn ~150 small semi-transparent particle sprites at the wall's plane, with outward velocities and slow fade-out over ~1.5s. Use `THREE.Points` with a soft circular texture, or instanced small planes — whichever you find cleaner.
7. **Optional but recommended:** play a low-frequency thud/rumble sound. Load from a CDN URL (e.g. a freesound.org direct link or any CC0 explosion sample). Make this controllable via a `soundEnabled` flag, default true.

### 6. The camera push-through
- Starting ~600ms after the explosion fires (giving the bricks time to clear the center), animate the camera forward on the z-axis from its starting position to roughly `z = -2` (i.e., past where the wall used to be) over ~1.2s with an ease-in-out curve.
- During the last ~400ms of the camera push, **crossfade the canvas to a black overlay** (or to white, whichever feels right — black probably).
- When the crossfade completes, fire a callback `onWallBroken()`. In Stage 1, this callback just logs to console and shows a placeholder message ("You are now inside the campaign site"). In Stage 2, this callback will be exposed as a prop so Framer can hand off to the next view.

### 7. Resource cleanup
After `onWallBroken()` fires, dispose of the physics world, the canvas WebGL context, and all GLB resources. The Three.js scene should fully unmount so it doesn't keep eating memory while the user browses the campaign site.

## Property Controls (apply in Stage 2; in Stage 1, expose as constants at the top of the file with comments)
- `brickGlbUrl` (default `./Brick.glb`)
- `wallColumns` (default 20)
- `wallRows` (default 12)
- `explosionImpulse` (default tuned value, e.g. 8)
- `cameraStartZ` (default 8)
- `cameraEndZ` (default -2)
- `dustParticleCount` (default 150)
- `enableShake` (default true)
- `enableSound` (default true)
- `buttonLabel` (default "Break the Wall")
- `triggerMode` ("button" | "wall-click", default "button")

## Stage 1 deliverable
A single `index.html` file that I can place in the same folder as `Brick.glb` and open with a local server (e.g. `python -m http.server`). Document at the top of the file:
- Any tunable constants and what they do
- The exact command to serve it locally
- Browser compatibility expectations (modern Chrome/Safari/Firefox)

Include a loading state ("Loading wall...") that shows while the GLB and Rapier WASM are loading.

## Stage 2 deliverable (do NOT build until I approve Stage 1)
Convert the working Stage 1 into a Framer Code Component:
- Single `.tsx` file, default export named `EntryWall`.
- Wraps the Three.js logic in a React component using `useEffect` for setup/teardown, `useRef` for the canvas.
- Uses `addPropertyControls` to expose the controls listed above.
- Exposes `onWallBroken` as a prop (callback function).
- The `Brick.glb` URL becomes a prop with a Framer `ControlType.File` control so I can swap it in the Framer canvas.
- All inline styles or a single injected `<style>` tag — no external CSS.
- TypeScript, no `any` unless genuinely unavoidable.
- Must compile cleanly when pasted into Framer's code editor.

## What "good" looks like
- The wall feels solid before the click — no jiggling, no settling.
- The explosion is **chunky** — bricks have weight, tumble realistically, and the screen shakes enough to feel it but not enough to make people queasy.
- The dust adds atmosphere; without it, the explosion looks sterile.
- The camera push feels like motion, not a cut.
- Total elapsed time from click to `onWallBroken()` firing: 1.8–2.2 seconds. Not faster (loses drama), not slower (loses energy).

## Out of scope for this build
- The campaign site itself (just fire `onWallBroken` and stop).
- Mobile-specific tuning (build desktop-first; we'll tune mobile after Stage 1).
- Accessibility / reduced-motion handling (note where it would go but don't build it).
- Multiple explosions / replay (one shot, then unmount).

## Final note
Prioritize the **feel of the explosion** over feature completeness. If you have to choose between adding a property control and making the bricks tumble more believably, tune the bricks. This is a campaign launch experience — it has to land.
