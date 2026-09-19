# Architecture — Cloud / Web Delivery & Systems Design

## Overview

This document describes how BREACH achieves hyper-realistic tactical body-camera CQB in both current web build (Three.js) and intended full-quality cloud build (Unreal Engine 5 + Pixel Streaming).

---

## 1. Current Web Build (Three.js) — What You Can Play Now

### Rendering

```
Three.js 0.160
  WebGLRenderer (antialias, ACESFilmic, PCFSoft shadows)
  EffectComposer
    RenderPass
    UnrealBloomPass (0.18 strength, 0.35 radius, 0.85 threshold)
    ShaderPass (BodyCamShader)
```

**BodyCamShader** implements:
- Barrel distortion: `uv = (uv-0.5)*(1 + r²*k + r⁴*k*0.5)+0.5` where k=0.11
- Chromatic aberration: R/B offset by `chromatic * (1+dist*2)` — stronger at edges, like real lens
- Vignette: `1 - dot(uv-0.5)*2.2` pow
- Exposure: uniform multiplied, target lerped based on indoors/dark/flashlight/night
- Sensor noise: `rand(uv+time)*grain*(1-lum*0.5)` — more in darks
- Scanline: `sin(y*800+time*2)*0.02`
- Motion blur: cheap horizontal sample offset by `motion*0.02`
- Flashbang: mix to white by `flash` uniform

### Material System

Procedural CanvasTexture generation to avoid external assets, but with PBR philosophy:

- `wallPaint(color)`: base color + subtle dark streaks (200 rects, 0.03 alpha) + imperfectionNoise (0.02 density) + roughnessMap + normalMap with random bumps
- `concrete()`: base #9a9a94 + 1200 speckles + 4 bezier cracks
- `wood()`: base #b89a6a or dark #5a3d2b + brushed lines with sin wave + 2 knots radial gradients
- `metal()`: brushed lines + rust speckles if rust>0

All return `MeshStandardMaterial` with appropriate roughness/metalness.

### Physics & Collision

No full physics engine for performance. Simple AABB colliders:

```js
colliders = [{x,z,w,d,rot,type}]
checkCollisions(newPos): rotate delta by -rot, check |rx|<hw+0.4 && |rz|<hd+0.4
```

For casings: simple Euler integration, gravity 9.8, bounce damping 0.3.

For ragdoll: `fallVel`, `fallTime`, rotation accumulation.

Future: cannon-es or ammo.js for full ragdoll, but current is performant for 100 enemies.

### Player & BodyCamera

**Player:** position, velocity, yaw, pitch, crouch, stamina 0-100, breathing 0.2-1.0, health 100, indoors/dark/night flags, flashlightOn.

**BodyCamera:**
- TargetPos = player.pos + bob + breath + inertia*0.5 + shake + lean
- Bob: sin(phase)*amp, amp 0.025 walk, 0.07 sprint, phase += dt*6 or 9.5
- Breath: sin(breathPhase)*amp, amp 0.008 idle, 0.03 sprint
- Inertia: mouseDelta*0.0008 accumulates, decays `pow(0.001, dt*60)`, clamp 0.15
- Spring: `velocity += delta*stiffness*dt; velocity *= 1-damping*dt*0.15; currentPos += velocity*dt`
- Rotation: YXZ order, yaw + inertia*0.6 + shake*0.2, pitch + inertia*0.6 + breath*0.5 + shake*0.3 + reload tilt -0.35
- Lean: targetRot.z = -lean*0.22, pos x+=lean*0.35
- Exposure lerp 0.8, flash lerp 2.5, motionBlur lerp 5

### WeaponSystem

- Rifle group: receiver, handguard, barrel, stock, grip, mag, sight — all primitive geometries
- Flashlight: SpotLight 22m, PI/6.5, 0.35 penumbra, 1.2 decay, shadow 1024, target at -3m
- MuzzleLight: PointLight 0-3m, intensity 4.5 on fire, decay lerp 20
- Sway: target = -mouseDelta*0.0005 + sin(time*0.6)*0.005, lerp 8
- Recoil: 0.35 per shot, lerp 8
- Reload: progress 0-1 at 0.85/sec (~1.18s), mag y lerp -0.35 when 0.2-0.45, back to -0.18 after 0.5, weapon y -0.05+sin(prog*PI)*0.08, rot x 0.4+sin*0.2
- Fire: check ammo, fireMode interval 95ms auto / 220 semi, tryFire returns bool, ejects casing, plays 3D sound, mic overload 0.35s

### AudioManager

- AudioContext, master 0.7, env 0.15, bodyMic 1.0, breathing dynamic
- Ambience: 2sec buffer white noise *0.02, lowpass 120Hz, loop
- play3D(pos,type,volume): Oscillator + Gain + Panner HRTF inverse 2m ref 40m max 1.2 rolloff
  - gunshot: 180Hz square 0-0.25s + 3000Hz crack 0-0.08s
  - footstep: 80-120Hz sine 0.12s
  - casing: 2000-3000Hz sine 0.2s
  - door: 120Hz triangle 0.4s
- micOverload: gain 0.3 → 1.0 over duration

### BuildingGenerator

- Clear: traverse scene, remove userData.isBuilding
- Floor: Plane size*size, concrete material repeat size/4
- Floorplan: L1-2 handcrafted templates (believable), L3+ procedural adjacent placement with centering
  - Types: living, kitchen, bedroom, bedroom2, bath, hall, storage, office
  - Placement: random side 0-3 from previous, snap round, avoid duplicate key
  - Center: avgX/Z subtraction
- buildRoom: 4 walls per room, thickness 0.12, height 2.8 or 3.2 if 2 floors
  - 65% interior door, exterior check size*0.4
  - Door: split wall into left/right/top, door mesh 0.9x2.05x0.04 woodDark, userData isDoor, open, angle, basePos/Rot
  - Window: exterior living/bedroom 50% chance, frame 1.2x1.1, glass MeshPhysicalMaterial transmission 0.85 opacity 0.18
  - Collider per wall
- populateFurniture: count by type, random pos inside room 0.8 margin, mesh BoxGeometry with appropriate material, castShadow, collider
  - Lamps: Cylinder 0.15, emissive if night, PointLight 1.2 intensity 6 range if night
- setupLighting:
  - Day: Directional 1.2 + Hemisphere 0.6
  - Night: Directional moon 0.35 + Hemisphere 0.25 + 4 streetlights Point 2.0 14 range + pole cylinder
- buildExterior: ground Plane size*2, color dark green vs light, perimeter walls Box size*2 2.5 0.3 concrete

### Enemy AI

**Enemy:**
- Mesh: torso Box 0.42x0.55x0.22 vest, head Sphere 0.16 skin, helmet Sphere 0.18, legs Cylinder 0.09x0.65, arms Cylinder 0.06x0.45, weapon Box 0.04x0.04x0.6
- State: idle, patrol, suspicious, investigate, search, engage, injured, retreat, dead
- Perception: vision raycast vs building (windows transparent), FOV 0.6 calm 0.85 alert, range 12, lighting check vis 0.25 dark no flashlight, 0.7 crouch
- Hearing: hear(pos,intensity,time) — dist check hearingRange*intensity (18*intensity), error dist*0.25+rand*2, errPos = pos + random*error
- Movement: moveTo(target,dt,world) — dir normalize, ahead raycast 1m for building, if blocked rotate PI/2 random, velocity dir*speed*dt*3.5, check colliders, position update, yaw atan2
- Search: generateSearchPoints — 4 points around lastKnown +-2m
- Damage: health 100, takeDamage 60-100, die → fallVel dir*2.5, isRagdoll true, updateRagdoll adds gravity, rotation
- Breathing anim: sin(time*0.5 or 1.2)*0.02 on torso y

**EnemyManager:**
- spawn(count,rooms): random room, random pos inside 0.6 margin
- update: per enemy update or ragdoll, communication: if alertLevel>=2, nearby <8m others alertLevel=1
- hearAll: per enemy hear
- getAliveCount
- tryEnemyShoot: if engage && canSee && rand>0.96, 40% miss, dist<10 && rand>0.6 → player health -15-35, vignette 0.6→0, play sound, mic overload 0.2

### Level Definitions

20 levels, enemies = id*5, size 14+id*1.2, floors 1 if <4 else 2, isNight true if >=6, objective cycle clear/search/investigate/secure, weather clear or rain/mist if >=14.

### Main Loop

- dt clamped 0.05, lastTime
- If playing:
  - moveSpeed crouch 1.1 sprint 4.2 else 2.0, wishDir from yaw, targetVel lerp 6 or 10
  - stamina: sprint -12/sec, else +18/sec, breathing lerp to 1.0 sprint else 0.2/0.5
  - collision check with slide X/Z
  - indoors check: abs pos < group.children.length*0.5 (simplified)
  - door prompt: raycast from camera center, dist 2.2, isDoor
  - animate doors: lerp angle to targetAngle 4/sec, hinge offset 0.45, pos sin/cos
  - firing: if MouseLeft, tryFire, raycast from center with random 0.04 spread, check building vs enemy (distance fallback angle 0.08), impact creation, hearAll 1.5
  - weapon update, reload update, bodyCamera update, enemyManager update, tryEnemyShoot
  - casings physics, smoke expansion opacity lerp 0.3→0, scale 1+dt*0.12
  - health check → menu DOWN, win → menu CLEAR or ALL CLEAR
  - HUD timestamp FPS, postprocess uniforms
- composer.render()
- mouseDelta reset

### Performance

- Renderer pixelRatio min(devicePixelRatio,1.8)
- Shadow map 1024-2048
- Procedural textures 256-512
- 12 spheres per smoke
- Simple colliders not physics engine
- No LOD yet but Three.js frustum culling
- FogExp2 for depth

---

## 2. Intended Full-Quality Cloud Build (UE5 + Pixel Streaming)

### Architecture Diagram

```
[Player Device]
  Browser
    ├─ Input: Keyboard, Mouse, Touch, Gamepad
    ├─ WebRTC DataChannel (reliable, ordered)
    └─ Video: <video> element H.264/H.265 decode
         ↑
         | WebRTC Video Stream (SRTP)
         |
[Signaling Server]
  Node.js / Python
    ├─ Matchmaking
    ├─ Session management
    ├─ STUN/TURN
    └─ Routes to available GPU server
         |
         ↓
[Cloud GPU Server] (e.g., AWS G5, GCP, Azure NV)
  ├─ Windows/Linux with NVIDIA GPU
  ├─ Unreal Engine 5.4+
  │   ├─ Game: BREACH (C++ & Blueprints)
  │   ├─ Rendering: Nanite, Lumen, Virtual Shadow Maps, Temporal Super Resolution
  │   ├─ Pixel Streaming Plugin (Epic)
  │   │   ├─ Captures backbuffer
  │   │   ├─ Encodes via NVENC (H.264 High, H.265 Main)
  │   │   ├─ Sends via WebRTC
  │   │   └─ Receives input via DataChannel
  │   └─ Systems:
  │       ├─ BodyCameraComponent (spring, inertia, exposure, optics)
  │       ├─ WeaponSystem (IK, procedural reload, ballistics)
  │       ├─ AI (Perception, Behavior Tree, EQS, cover, search)
  │       ├─ Audio (MetaSounds, Steam Audio, convolution reverb per room)
  │       ├─ Physics (Chaos, ragdoll, destruction)
  │       └─ World (20 levels, Megascans, PCG, volumetric fog, water, grass, wind, weather)
  └─ Encoder: 1080p60 8Mbps, 1440p60 12Mbps, 4K60 25Mbps adaptive

[CDN / Website]
  breachtactical.com
    ├─ Landing, briefing, level select
    ├─ Web client (JS) that connects to signaling
    ├─ Auth, progress save (cloud)
    └─ Static assets
```

### Why Cloud?

- UE5 Lumen + Nanite + Virtual Shadows requires RTX 4070+ for 60 FPS at high quality — not available on phone/low-end laptop
- Cloud GPU renders at 60-120 FPS, phone only decodes video (hardware accelerated, low power)
- Phone still needs decent network (15Mbps+, <50ms RTT to edge) and decode capability, but not 3D rendering power
- Advantage: instant access, no download, always latest version, anti-cheat server-side, scalable

### UE5 Systems Mapping (from web build)

| Web Build | UE5 Equivalent |
|-----------|----------------|
| Three.js MeshStandardMaterial + CanvasTexture | Megascans + Material Instances, roughness variation via vertex paint, imperfection via decal |
| BodyCamShader (barrel, CA, vignette, grain) | Post Process Material, Lens Distortion, Chromatic Aberration, Film Grain, Vignette, Auto Exposure (Histogram) |
| BodyCamera spring | Spring Arm Component + Camera Lag, Inertia via Physics, Breathing via Timeline |
| WeaponSystem primitive | Skeletal Mesh + IK (Control Rig), Anim Montage reload variations, Ballistics via Line Trace |
| BuildingGenerator procedural | PCG + Modular building pieces, handcrafted art pass per level, not random boxes |
| Enemy AI simple state machine | Behavior Tree + Blackboard + Perception (Sight/Hearing) + EQS + Cover System + Search with uncertainty (add random to last known) |
| AudioManager Web Audio | MetaSounds + Steam Audio propagation, occlusion, reverb per room, bodycam mic via Audio Mixer + low-pass on overload |
| Smoke 12 spheres | Niagara volumetric smoke, wind via Vector Field, light interaction via Volumetric Fog |
| Flashbang shader flash | Post Process + Audio Mixer ducking + AI Perception stimulus |
| Door rotation lerp | Physics Constraint + Interactable Component, sound via Audio Component, affects NavMesh + Audio propagation |

### Networking

- Input: client → server via DataChannel, 60Hz, includes mouse delta, keys, timestamp. Server applies with lag compensation.
- Video: server → client via WebRTC, NVENC, adaptive bitrate based on bandwidth estimation (Google Congestion Control)
- Signaling: WebSocket to exchange SDP, ICE candidates
- Session: 1 player per GPU instance (single-player), spin up on demand, spin down after idle 5 min

### Scaling

- Kubernetes with GPU nodes, autoscaling based on queue
- Each pod: 1 UE5 instance + Pixel Streaming
- Cost: ~$0.7/hr per G5 instance, 20 min average session = $0.23

### Security

- No game code on client, only video — anti-cheat inherent
- Input validation server-side
- Progress saved server-side (database)

### Fallback

If network <10Mbps or RTT >100ms, downgrade to 720p30, increase encoding QP, or offer local Three.js build (current index.html) as fallback.

---

## 3. Systems Cooperation for Presence

The core insight: realism is not graphics alone. It is cooperation:

- **Environment** provides believable architecture, materials, imperfections
- **Lighting** makes night matter, flashlight matter, exposure matter
- **Camera** removes floating feeling, adds inertia, bob, breath, shake
- **Movement** has weight, acceleration, stamina, breathing
- **Weapon** is physical, not floating, with mechanical sounds, smoke, casings
- **AI** exists independently, patrols, hears with uncertainty, searches incorrectly, takes cover, communicates vaguely
- **Physics** makes doors matter for sight/sound, objects have weight
- **Sound** tells where things are, propagates through rooms, mic overloads
- **Particles** react to environment, not generic
- **UI** minimal — no minimap, no markers, information via environment

All reinforce each other. Remove one, presence breaks.

---

## 4. Quality Test (from prompt)

At milestones, capture screenshots/clips and ask:

- Does this look like obvious videogame? → Fix materials, lighting, imperfections
- Camera physically attached? → Fix spring, inertia, bob
- Building believable? → Fix architecture relationships
- Materials correct? → Fix roughness/metalness
- Lighting makes sense? → Fix exposure, shadows
- Imperfections believable? → Fix contextual placement
- AI knows things it shouldn't? → Fix perception, add uncertainty
- Player vulnerable? → Fix health, enemy accuracy, cover
- Sound tells where? → Fix propagation, HRTF, occlusion
- Weapon connected? → Fix sway, lag, IK
- Movement weight? → Fix acceleration, stamina
- Gunfire affects camera naturally? → Fix recoil, shake
- Environment reacts? → Fix particles, physics
- Uncertainty created? → Fix AI search, sound, lighting

If unsatisfactory, identify specific cause and correct. Do not add more effects — more ≠ realism.

---

## 5. Future

- Full UE5 migration with pixel streaming
- 20 handcrafted levels, not procedural
- Multiplayer co-op? No, single-player focus per prompt
- Modding support? Maybe
- VR bodycam? Possible but motion sickness from bob — need comfort options

---

Built on Arena.ai, 2026-09-19.
