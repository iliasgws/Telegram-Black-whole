# Black Hole Delete Android Port

## Goal
Port the visual interaction from `riveauxmia/telegram-black-hole-delete` to Telegram Android as a native, production-minded feature experiment.

Reference implementation:
- https://github.com/riveauxmia/telegram-black-hole-delete

Do not redesign the effect. Preserve the original visual language and timing as closely as Android allows.

## Scope
Integrate with Telegram Android's existing **Clear History** flow. Do not replace or reimplement Telegram's deletion backend.

The effect should:
1. Capture the visible chat wallpaper and visible message cells before history is hidden/cleared.
2. Open a black hole from the device's top display cutout / camera-hole region, or from top-center just above the screen when no suitable cutout exists.
3. Bend and darken the wallpaper toward the hole.
4. Pull visible messages into the hole using deformable textured meshes, including acceleration, stretching, fading and blur near the horizon.
5. Render the black-hole body and spinning light ring.
6. Transition to Telegram's existing cleared-history state and existing undo/bulletin UI.
7. Collapse the hole back into the display-cutout origin and remove all temporary rendering views.

## Reference files to study first
From the iOS reference repository:
- `README.md`
- `BlackHoleDelete/Sources/BlackHoleTimeline.swift`
- `BlackHoleDelete/Sources/BlackHoleDeleteAnimator.swift`
- `BlackHoleDelete/Resources/BlackHole.metal`
- `BlackHoleDelete/Resources/BlackHoleDisk.png`
- `Integration/ChatControllerBlackHoleClear.swift`

## Android architecture
Keep the feature isolated, preferably under a dedicated component/package such as:

`TMessagesProj/src/main/java/org/telegram/ui/Components/BlackHoleDelete/`

Suggested classes:
- `BlackHoleConfig.java`
- `BlackHoleTimeline.java`
- `BlackHoleDeleteAnimator.java`
- `BlackHoleRenderer.java`
- `BlackHoleGLView.java`

Shader/resources should live in suitable Android resource/assets locations.

## Rendering design
Preserve the original three-pass structure:

### 1. Backdrop pass
Render the captured wallpaper with:
- radial lensing toward the hole
- swirl/twist around the horizon
- deep local darkening
- warm bloom around the light ring
- subtle dithering if useful against banding

### 2. Message mesh pass
For each visible message:
- snapshot the message cell to a `Bitmap`
- upload it as an OpenGL texture
- subdivide its rect into an approximately 6dp mesh grid
- deform vertices toward the black-hole center
- use the same TidalFlow / travel-time equations from the iOS implementation
- preserve near-horizon acceleration and "spaghettification"
- fade and soften the texture near the event horizon

Do not implement the effect as a set of ordinary View translation/scale animations.

### 3. Overlay pass
Render:
- the black body
- cutout/capsule-to-circle morph
- animated light ring using the reference texture / visual behavior

## Renderer
Prefer OpenGL ES 3.0 with EGL or the closest rendering path that integrates cleanly with Telegram Android.

Use `Choreographer.FrameCallback` for timing.

Avoid unnecessary allocations inside the frame loop.

Precompile shaders and prewarm rendering resources before the animation starts when practical, ideally while the confirmation UI is visible.

## Display cutout behavior
Do not fake an iPhone Dynamic Island.

Use Android window/display-cutout information.

Desired origin behavior:
- centered punch-hole -> originate there
- pill-shaped top cutout -> originate from its bounds
- corner punch-hole -> originate there
- no usable cutout -> use a small capsule just above top-center

## Timeline and physics
Port the original config/timeline math as directly as possible.

Initial tuned values from the reference implementation:
- holeRadius: 90dp-equivalent
- emergeDuration: 0.8s
- collapseDuration: 0.3s
- collapseBounce: 0.35
- settleDuration: 0.34s
- swallowDuration: 1.15s
- swallowLead: 0.1
- pullStrength: 3
- pullReach: 0.8 * radius
- flowRamp: 0.35s
- flowAccel: 8
- arcShape: 1
- fadeOuter: 1.6 * radius
- fadeInner: 1.03 * radius
- fadeBlur: 14dp-equivalent
- glow: 0.82
- lensStrength: 1.5
- spaceDarkness: 0.55
- spaceSize: 2 * radius
- backgroundSwirl: 0.9 rad
- backgroundSpin: 0.8 rad/s

Use a capped simulation delta similar to the reference implementation:
`dt = min(realDt, 1/30s)`
so dropped frames do not cause visible jumps.

## Telegram integration
Find Telegram Android's current Clear History confirmation and execution path.

Search around likely areas including:
- `ChatActivity`
- history deletion / clear-history helpers
- `MessagesController`
- bulletin / undo UI

Capture visual state **before** the message list is hidden, reset or emptied.

Do not change deletion semantics. The effect is visual glue around the existing flow.

## Accessibility and fallback
When system animations are disabled or animator duration scale is zero, skip the effect and use Telegram's existing Clear History behavior.

The normal flow must remain functional even if OpenGL initialization, snapshots or shaders fail.

## Performance targets
- 60fps baseline
- support 90/120Hz where feasible
- reuse buffers/textures when possible
- no per-frame Bitmap creation
- no large object churn in frame callbacks
- ensure overlay teardown releases GL resources

## Validation
Before considering the task complete:
1. Build Telegram Android successfully.
2. Verify ordinary chat use is unaffected.
3. Verify Clear History still works with the effect disabled/failing.
4. Verify the effect runs on a device/emulator with no cutout.
5. Verify behavior with at least one cutout layout if available.
6. Verify light and dark themes.
7. Verify large/small message cells and media-heavy chats.
8. Verify navigation/status-bar transitions do not visibly jump.
9. Document known fidelity differences from the iOS reference.

## Working style for Codex
- Inspect existing Telegram components before adding new abstractions.
- Reuse Telegram helpers and rendering infrastructure where appropriate.
- Keep changes focused and reviewable.
- Build frequently.
- Fix compile errors before continuing to feature polish.
- Do not silently replace working Telegram behavior with stubs.
- Keep a short implementation note describing modified Telegram files and integration points.
