# Controls — BREACH

## Movement (Heavy, Physical, Responsive)

- **WASD** — Move forward/back/strafe. Acceleration/deceleration, no glide.
- **Mouse** — Look. Inertia, slight rotational lag.
- **Shift** — Sprint. Increases camera bob (0.07m), breathing, stamina drain, louder footsteps.
- **Ctrl** — Crouch. Height 1.05m vs 1.42m standing, slower, harder for AI to see (vis *0.7).
- **Q / E** — Lean left/right. Moves body + camera together, exposes actual body portion, not magical camera extension. Lean 0.35m lateral.
- **Space** — (Disabled) No jump — CQB is about clearing, not platforming. Stairs handled via collision.

## Combat

- **Left Mouse** — Fire. 95ms auto interval, recoil 0.35, camera kick, muzzle flash, casing ejection.
- **Right Mouse** — Aim Down Sights. FOV 78 → 65, weapon position lerp to center, sway reduced.
- **R** — Reload. Detailed: camera tilts down -0.35 rad, weapon tilts 0.4 rad, mag visibly leaves, hand to equipment, new mag inserted. 1.18 sec. Variations possible calm/stressed.
- **F** — Flashlight. Toggle mounted SpotLight 22m, PI/6.5 cone, casts shadows, interacts with dust/smoke/glass/metal, affects exposure.
- **G** — Smoke grenade. Volumetric-like 12 spheres, expands, wind affects, light interacts.
- **H** — Flashbang. Bright flash, whiteout shader, audio distortion (mic overload 1.2s), AI blind <6m.

## Interaction

- **E** — Interact. Doors (open/close, affects sound propagation, visibility, lighting, AI nav) and objective package (secure).
- **ESC** — Pause. Shows menu, breathing, building still exists.

## HUD (Minimal)

- No minimap, no enemy markers, no health bars, no outlines, no hit markers.
- Top: REC indicator (blink), bodycam model AXON BODY 4, level name, enemy count, objective
- Bottom: Ammo 30/90, fire mode, light status, objective text, controls hint (faint), breathing indicator (pulses with breathing)
- Center: No crosshair unless aiming (hidden). Interaction prompt when near door.
- Vignette: Damage flash red radial, opacity 0.6→0 in 300ms.
- Flash overlay: White flashbang 0.95→0 in 250ms.

## Tactics

- Think before entering room
- Check corners — architecture creates tension
- Listen — sound propagation same for player and AI
- Doors matter — closed changes sound, open exposes sightline, partially open creates uncertainty
- Flashlight reveals but exposes — AI can see light?
- Lean to peek without exposing full body
- Breathing affects aim after sprint
- Silence is intentional — not every moment filled with ambient noise

## Audio Cues

- Footsteps: volume 0.9 sprint, 0.5 walk, 0.25 crouch — AI hears with uncertainty
- Gunshots: intensity 1.5, range 27m, AI gets approximate area
- Doors: intensity 0.6, range 10.8m
- Casings: per material sound (concrete/wood/tile/metal/carpet/soil) — subtle
- Breathing: audible after firefight against quieter environment
- Mic overload: sudden gunfire creates temporary gain drop 0.3→1.0

## AI Behavior You Will See

- Calm patrol → hears sound → suspicious (pause, look) → investigate → search (checks last known, nearby rooms, behind cover, doorways, listening) → engage → if loses sight → search, not return to normal
- Sometimes searches wrong room — perfect AI doesn't feel realistic
- Takes cover, peeks, repositions, retreats, waits, flanks
- Communicates vaguely: nearby <8m become alert but not exact location
- Injured: seeks cover, reduced mobility

## Level Progression

- L1: 5 enemies, small residential, day, learn systems
- L2: 10 enemies, larger, more rooms
- L3: 15 enemies, complex navigation
- L4: 20 enemies, multiple floors
- L5: 25 enemies, larger two-story
- L6+: 30-100 enemies, night, larger, darker, more complex, outdoor, multiple buildings, basements, longer sightlines

Difficulty not just enemy count — better AI, complex buildings, darker, more positions, complicated sound, navigation, varied behavior, complex objectives, environmental uncertainty.

## Quality Test

Ask: Does camera feel attached? Building believable? Materials correct? Lighting makes sense? Imperfections believable? AI knows things it shouldn't? Player vulnerable? Sound tells where? Weapon connected? Movement weight? Gunfire affects camera naturally? Environment reacts? Uncertainty created?

If any answer unsatisfactory, fix specific cause, not add more effects.

## Performance

Target 60 FPS+, 90-120 possible. LOD, occlusion, asset streaming, efficient AI, instancing, optimized particles, texture 256-512, efficient physics.

## Cloud Delivery

Full UE5 version: cloud GPU renders, encodes H.264/H.265, pixel streams to browser, input back via WebRTC. Phone doesn't render UE5, only decodes video. Needs 15Mbps+, <50ms RTT.

Current web build is faithful approximation running locally.

