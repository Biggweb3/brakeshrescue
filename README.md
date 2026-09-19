# BREACH — Hyper-Realistic Tactical Body-Camera CQB

> **A game that is dangerously close to looking like real footage.**

BREACH is an original single-player tactical first-person CQB game built around one central objective: **make the player feel physically present inside a real environment, viewing the world through a body-worn camera.**

Visual and experiential reference: BODYCAM-level realism. Original assets, original levels, original code. No copied assets, maps, characters, animations, sounds, branding, or source.

Live preview: `python3 -m http.server 8000` → open `index.html`

---

## Core Philosophy

- **Vulnerability is the mechanic.** Both player and AI are cautious because they can die quickly.
- **Uncertainty is the primary system.** “Did I hear something? Was that movement? Is this room clear?”
- **The building is the level design.** Rooms have believable relationships, doors make sense, windows correspond to exterior.
- **Every system cooperates to create presence:** environment, lighting, materials, camera, movement, animation, AI, physics, sound, particles, weapon behavior.

### What it is NOT

- Not an arcade shooting gallery
- Not shiny videogame surfaces
- Not a fake camera filter pasted over normal FPS
- Not enemies spawning in front of player because “it’s time for combat”

---

## Visual Realism — Web Implementation of UE5 Philosophy

Target: highest-quality real-time graphics possible in browser, approximating UE5 capabilities:

- **PBR Materials:** roughness/metallic/normal maps procedurally generated with contextual imperfections. Walls have subtle paint variation, concrete has cracks, wood has grain and knots, metal has brushed lines and rust where logical.
- **Lighting:** Physically believable lights — interior point lights, exterior streetlights, moonlight, flashlight with shadows. Night is actually dark. Flashlights, street lights, windows, moonlight matter. Exposure adaptation.
- **Geometry Detail:** No perfect surfaces. Furniture shows wear. Tiles not identical. But imperfection tells a story — new building looks new, abandoned shows deterioration.
- **Environmental Detail:** 40 believable objects > 300 random objects. Outlets, switches, lamps, furniture, kitchen objects, curtains, pipes, etc. Placed where they belong.

### Rendering Stack

- Three.js 0.160, WebGL2, ACESFilmic tone mapping, PCFSoft shadows
- EffectComposer post-processing:
  - UnrealBloomPass (subtle, 0.18 strength — avoid excessive bloom)
  - **Custom BodyCam Shader:** barrel distortion (0.11), chromatic aberration edge-dependent, vignette, sensor noise luminance-dependent, scanline, motion blur, flashbang whiteout, desaturation, black crush

---

## The Body-Camera — Most Important System

Not a floating FPS camera. Physically attached to chest (1.42m high, not eye level).

**Movement response:**
- Walk: 0.025m bob at 6Hz, Run: 0.07m at 9.5Hz
- Inertia: mouse delta accumulates into inertia vector with exponential decay (0.001^(dt*60))
- Spring-damper: targetPos → velocity → currentPos with stiffness 18, damping 10
- Stopping: continuation of body movement, not instant freeze
- Stairs: vertical bob handled via player stepping
- Lean: Q/E moves body + camera, exposes actual body portion, not magical camera extension
- Weapon recoil shakes camera, but player can still see

**Optics:**
- Wide FOV 78° (65° ADS), mild barrel distortion, edge distortion, subtle chromatic aberration (0.0025), vignette 0.45, realistic depth, sensor noise more in darks, motion blur from speed + mouse.

**Exposure:**
- Adaptive exposure lerps to target: indoors+flashlight 1.1, dark 1.8 (noisy), night 1.4, day 0.9
- Flashlight affects exposure naturally, bright light causes response.

---

## Movement — Heavy, Physical, Responsive

- Acceleration/deceleration exists, no glide
- Turning creates slight body movement
- Sprint increases camera movement + breathing, stamina system
- Crouch: 1.05m height, slower, harder to see
- Lean: dedicated left/right, body moves
- Breathing system: quiet when idle, heavier after sprint/combat, mic captures close breathing, audible after firefight against quiet environment

---

## Weapons — Physically Attached

- Rifle built from primitives with PBR metal/polymer materials
- Movement responds to walking, running, breathing, turning, aiming, recoil, reload, injury
- Muzzle flash: PointLight + color variation
- Casing ejection: physics objects with per-surface sound, bounce, friction
- Recoil: 0.35 per shot, camera kick, weapon kick, not absurdly strong
- Flashlight: SpotLight mounted, 22m range, 26° cone, casts shadows, interacts with dust/smoke/glass/metal, affects exposure
- Reload: highly detailed — camera tilts down -0.35 rad, weapon tilts 0.4 rad, magazine visibly leaves, hand moves to equipment, new mag inserted. Procedural animation, variations possible (calm vs stressed). Uses lerp for mag out/in.
- Sway: mouse delta + breathing + movement, weapon lags behind camera

Controls: LMB fire, RMB aim (FOV 65), R reload, F flashlight, G smoke, H flash, Q/E lean, E interact.

---

## Enemy AI — Most Important Gameplay System

**No cheating:** No wallhack, no teleport, no spawn in front, no coordinates, no endless rush.

**Perception:**
- Vision: distance (12m), line of sight (raycast vs building, windows transparent), lighting (harder in dark without flashlight), obstacles, doors, player movement, exposure, crouch. FOV 0.6 rad calm, 0.85 rad alert.
- Hearing: gunshots (intensity 1.5, range 18m*intensity), footsteps (future), doors (0.6), objects, explosions. Sound propagates through environment.

**Uncertainty — Essential:**
When enemy hears gunshot, it gets approximate area + direction with error = dist*0.25 + random*2. It thinks “sound came from over there”, not exact coordinates. Can investigate wrong room, change direction on new sound, become confident as evidence accumulates.

**States:** calm, patrolling, idle, suspicious, investigating, alert, searching, engaging, taking cover, repositioning, flanking, retreating, injured, recovering, dead. Transitions natural.

Example: Calm patrol → hears shot → suspicious (pause, look) → investigate (move cautiously) → search (check last known, nearby rooms, doorways, behind cover, listening) → if sees player → alert/engage → if loses sight → search, not return to normal.

**Tactics:** Take cover, remain behind walls, peek, reposition, retreat, search, wait, approach from another direction. Depends on situation, not suicidal.

**Communication:** If one detects player, nearby (<8m) become more alert, but not exact location. “Someone is here” → “Something happened in that direction”.

**Searching:** Most important behavior. Checks last known, looks around, nearby rooms, behind cover, doorways, listening, repositioning, waiting. Sometimes searches incorrectly — perfect AI doesn’t feel realistic.

**Injury:** Simplified damage, not medical simulation. Arm injury → impaired + cover, leg → reduced mobility, torso/head → incapacitation. Visible reaction, posture, movement.

**Ragdoll:** Animation + physics blend. Falls according to impact direction, orientation, nearby objects, stairs, walls, furniture, floor. Dust cloud on dusty concrete, different debris per surface (drywall vs concrete vs wood vs metal sparks).

---

## Particles & Effects

- **Dust/Impacts:** Different per surface, not one universal effect
- **Smoke:** Volumetric-like using 12 spheres per grenade, expands, moves, opacity fades, light interacts, wind affects
- **Flash:** Bright flash, temporary whiteout (shader flash uniform), audio distortion (mic overload), AI confusion, recovery
- **Frag:** Light, sound, smoke, dust, debris, camera movement, environmental reaction. Not cartoonishly large. Different inside small room vs outside.
- **Water:** Reflects sky, lights, buildings. Small waves, ripples (planned for outdoor levels)
- **Grass/Vegetation:** Dense thin blades instanced (planned), wind variation, no repeated patterns
- **Wind:** Global system influencing grass, trees, smoke, dust, fog, curtains, loose objects. Weight-dependent.
- **Weather:** Clear night, cloudy, light/heavy rain, mist, clear day. Rain interacts with windows, roads, puddles, wet surfaces reflect.
- **Clouds/Sky:** Moving slowly, density varies, affects ambient lighting, moonlight interaction at night.

---

## Audio — As Important As Graphics

- **Realistic:** footsteps, breathing, weapon handling, reload, mag sounds, casing sounds per surface (concrete/wood/tile/metal/carpet/soil), gunfire, distant gunfire, doors, windows, wind, rain, electricity, AC, pipes, building creaks, vehicles, insects
- **Silence exists:** Silence creates tension, not filled every moment
- **Sound Propagation:** Same room = immediate, behind wall = muffled, down hallway = reverberation, outside = different profile, multiple rooms = filtered. AI hearing uses same logic.
- **Casing Sounds:** Subtle, per material, not attention-grabbing
- **Environmental Audio:** House = electrical hum, fridge, AC, pipes, wind, distant traffic, dogs, insects. Abandoned different. Industrial different. Outdoor different.
- **Bodycam Mic:** Physical part of camera — cloth/handling noise, breathing audible, sudden gunfire overload (gain 0.3 → 1.0 over 0.5s), explosions distort briefly, wind affects mic.

Implementation: Web Audio API, HRTF panning, inverse distance, procedural synthesis to avoid asset loading. Master 0.7, env 0.15, bodyMic 1.0, breathing dynamic.

---

## Physics & Interaction

- Objects have appropriate responses, physics selectively where immersion contributes
- **Doors:** closed/open/partially open/locked/unlocked/damaged, influence visibility, sound, lighting, AI navigation, combat. Closed changes sound travel, open exposes sightline, partially open creates uncertainty.
- **Windows:** reflection, transparency, lighting, outside visibility, breakable (glass material transmission 0.85, opacity 0.18)
- **Materials:** PBR throughout, roughness/metallic/normal variation, concrete ≠ plastic, wood ≠ metal, etc.
- **Imperfections:** scratches, dust, wear, stains, fading, roughness variation, small damage — but contextual, not random dirt everywhere.

---

## Level Progression — 20 Levels

| Level | Enemies | Size | Floors | Night | Objective |
|-------|---------|------|--------|-------|-----------|
| 1 | 5 | 15.2m | 1 | No | Clear residential — learn movement, camera, weapon, flashlight, basic AI |
| 2 | 10 | 16.4m | 1 | No | Larger building, more rooms |
| 3 | 15 | 17.6m | 1 | No | Complex navigation |
| 4 | 20 | 18.8m | 2 | No | Multiple floors |
| 5 | 25 | 20m | 2 | No | Larger two-story |
| 6-20 | 30-100 | 21.2-38m | 2 | Yes | Larger structures, multiple buildings, outdoor areas, dark, complicated AI, sound encounters, multiple entrances, staircases, basements, upper floors, longer sightlines, complex objectives |

**Not just enemy count:** Difficulty via better AI, complex buildings, more floors/rooms, darker environments, more possible positions, complicated sound propagation, difficult navigation, varied behavior, complex objectives, environmental uncertainty.

**Missions:** clear building, secure location, investigate disturbance, recover object, reach location, rescue NPC, secure room, search structure, survive.

**Randomization:** Patrol positions vary, idle varies, search routes vary, some rooms different enemies, environmental events vary — but controlled, still deliberately designed.

### Current Implementation

- Procedural believable architecture: handcrafted templates for L1-2, BSP-like adjacent placement for L3+ with centering
- Rooms have purpose: living, kitchen, bedroom, bedroom2, bath, hall, storage, office
- Furniture: sofa, table, bed, kitchen counters, etc. with collision
- Lighting: sun/moon + hemisphere + interior point lights + streetlights at night
- Exterior: ground plane, perimeter walls
- Enemy spawning: distributed throughout rooms, not simultaneously in same spot
- Efficient AI: LOD via reduced tick for distant? Currently all tick but with early outs

---

## Controls

```
WASD Move • Mouse Look • Shift Sprint • Ctrl Crouch
Q/E Lean (exposes body) • RMB Aim • LMB Fire
R Reload (detailed, camera tilts) • F Flashlight
G Smoke (volumetric) • H Flashbang
E Interact (doors) • ESC Pause
```

---

## Performance Target

60 FPS+, 90-120 possible on powerful hardware. Techniques:

- LOD, occlusion (Three.js frustum), asset streaming (procedural), efficient AI updates, instancing (grass planned), optimized particles (12 spheres per smoke), texture resolution 256-512, efficient physics (simple AABB colliders, not full physics engine per object)
- Distant AI reduced simulation (planned: lower tick rate)
- Distant objects LOD (planned)

---

## Cloud / Web Delivery Architecture

Preferred for full UE5 quality:

```
Browser (keyboard/touch/controller) 
  → WebRTC / WebSocket input
  → Cloud GPU Server (Unreal Engine 5, real-time simulation)
  → Video encoding (H.264/H.265)
  → Pixel streaming
  → Browser client decodes & displays
```

Website is access point, cloud does heavy rendering. Phone doesn't need to render UE5 locally, but needs decode + network quality.

**Current web build** is a faithful approximation running locally in browser using Three.js, demonstrating all systems cooperating. For production UE5 pixel streaming, replace Three.js renderer with UE5 Pixel Streaming plugin, keep same game logic concepts.

See `ARCHITECTURE.md` for detailed cloud design.

---

## Development Strategy — Vertical Slice

Built vertically:

1. Single highly detailed room → small building
2. Player movement → body-camera
3. One weapon → firing → reload
4. One enemy → AI perception → searching → combat → injury → ragdoll
5. Sound propagation → lighting → smoke/particles
6. Combine into Level 1 → playtest → fix camera if wrong, movement if floaty, AI if cheats, building if artificial, lighting if gamey, audio if generic, reload if pasted
7. Only when Level 1 feels convincing, expand to 20

Quality test: Screenshots/clips — Does it look like obvious videogame? Does camera feel attached? Building believable? Materials correct? Lighting makes sense? Imperfections believable? AI knows things it shouldn't? Player vulnerable? Sound tells where? Weapon connected? Movement weight? Gunfire affects camera naturally? Environment reacts? Uncertainty created? If unsatisfactory, fix specific cause, not add more effects.

---

## Project Structure

```
index.html          # Main game — single file for easy preview, ~1900 lines
README.md           # This file
ARCHITECTURE.md     # Cloud delivery & systems design
src/
  core/             # (future modular split)
  world/
  ai/
  levels/
public/             # Assets (procedural, no external)
```

---

## How to Run

```bash
python3 -m http.server 8000
# open http://localhost:8000
# Click to lock mouse, WASD, etc.
```

No build step, no npm install, uses importmap with CDN Three.js.

---

## Future Work (UE5 Migration)

- Replace Three.js with UE5 + Pixel Streaming
- Megascans materials, Nanite geometry, Lumen lighting
- MetaHuman-based enemies with IK, procedural reload variations (calm/stressed/moving)
- Full navmesh with cover points, flanking, breaching
- Volumetric fog, water with FFT waves, dense grass with WPO wind
- Audio: convolution reverb per room, Steam Audio propagation
- 20 handcrafted levels with art pass, not procedural
- Multi-floor with staircases as choke points, basements, attics
- Objective types: hostage rescue with NPC AI, bomb defusal, evidence collection

---

## Design Pillars

1. **Presence over graphics** — graphics serve presence, not vice versa
2. **Uncertainty over clarity** — absence of information intentional
3. **Vulnerability over power fantasy** — player cautious, AI cautious
4. **Systems over scripts** — enemies exist independently, not waiting for player
5. **Believability over quantity** — 40 correct objects > 300 random

---

## License

Original creation. No BODYCAM assets, maps, characters, animations, sounds, branding, source code, UI, or exact level designs copied. BODYCAM is reference for quality and feeling only.

Built on Arena.ai Agent Mode.

