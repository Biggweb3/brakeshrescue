# BREACH — Hyper-Realistic Tactical Body-Camera CQB (MOBILE-FIRST)

> **A game that is dangerously close to looking like real footage — now designed specifically for mobile landscape.**

Live: `python3 -m http.server 8000` → open `index.html` on phone in landscape.

---

## MOBILE-FIRST MANDATORY CHANGES — IMPLEMENTED

### 1. Mobile-First Interaction Model
Entire game designed around touch, not PC with mobile added afterward. No WASD, no mouse instructions, no visible virtual joystick permanently occupying screen. Left side invisible movement region, right side look region, minimal semi-transparent buttons.

### 2. Landscape Orientation Gate
Before gameplay, detects orientation via `window.innerHeight > window.innerWidth`. If portrait, shows immersive screen:
```
ROTATE YOUR PHONE
This operation is designed for landscape mode.
[phone rotating animation]
```
Game auto-continues when landscape detected. Listens to `resize` + `orientationchange`.

### 3. No Level-Selection Screen
Removed: Level 1/2/3 grid, locked levels, cards, "20 levels" counter. Player never sees level list. Backend still uses `Level_01..Level_20` internally, but UI never exposes.

### 4. Continuous Progression Mode
Flow:
```
Start Game → Enter Building → Clear Area → 2-3s Transition → Spawn Outside Next → Physically Enter → Continue
```
Backend maintains `currentProgressionStage` in localStorage. No manual selection. Player feels continuous operations, not menu navigation.

### 5. Initial Entry
After orientation satisfied, simple start screen with `ENTER OPERATION` button. No "LEVEL 1". First environment feels like beginning of operation.

### 6. Auto Progression
On objective complete (enemies 0 + package secured if applicable):
1. Confirm complete
2. No arcade "LEVEL COMPLETE" — shows "SECURED"/"CLEAR"/"AREA SECURE"/"OBJECTIVE COMPLETE" briefly
3. 2-3 second transition with progress bar
4. Load next environment
5. Player appears outside next building (at gap entrance)
6. Player must physically walk toward and enter building

### 7. Physical Building Entry
Player spawns at `size*0.5 + 4.5` outside, facing building. Exterior includes:
- Exterior walls with gap entrance (perimeter wall split)
- Windows, doors, outdoor lighting, terrain, vehicles (2 cars), vegetation, weather
- Must observe and approach, then enter through doorway
- Seamless exterior→interior, no "ENTER LEVEL" button after spawn

### 8. Hidden Level Architecture
Backend: `internalId: Level_01..Level_20`, `opName`, `codename` (CLEAR HOUSE, NIGHTFALL, BREACH...ENDGAME). Frontend: shows `OP_NAME • CODENAME` (e.g., "RESIDENTIAL • CLEAR HOUSE") — never "LEVEL 1/20" or "1 of 20". Architecture allows adding levels without redesigning progression.

### 9. Mobile Touch Movement
- Left 42% screen = movement region, invisible by default, no giant joystick
- Touch & drag: `move.x = dx/maxDist*1.2`, `move.y = -dy/maxDist*1.2`, clamp 0-1
- Sprint if forward >0.75 and dist>75% max
- Tutorial: "Touch & drag left side to move" — disappears after first time

### 10. Mobile Camera Control
- Right 58% screen = look region
- Drag to rotate: `yaw -= dx*0.0026*sens`, `pitch -= dy*0.0026*sens`, clamp -0.45π to 0.42π
- Preserves body-camera system: bob, inertia, shake, breathing, recoil, spring-damper, lean — touch controls direction, does not remove physical behavior

### 11. Mobile Combat UI — Minimal
- Fire button largest: 86px, semi-transparent rgba(255,255,255,0.08), faint, right side 6% bottom 18%, easy thumb reach, not obscuring weapon
- Not huge bright arcade button

### 12. ADS Button
- Smaller secondary near fire: 52px at 22% bottom 20%
- When activated: weapon moves naturally to aiming position (lerp), camera responds physically, body+weapon connected, not snap

### 13. Realistic Scope Behavior — Localized Magnification
- Scope does NOT cover entire screen
- Visual concept:
```
+------------------------------------------------+
|                 BODYCAM VIEW                   |
|                     ______                     |
|                   /        \                   |
|                  |  ZOOMED  |                 |
|                  |  OPTIC    |                 |
|                   \________/                   |
+------------------------------------------------+
```
- Implementation: Shader uniform `ads` — inside optic radius 0.14, magnified 2.8x via `zoomedUV = center + (uv-center)/zoom`, plus second renderer `scopeRenderer` with `scopeCamera` FOV 18 (vs main 78) rendering to `#scope-overlay` circular div 180px with reticle (red cross + dot). Surrounding bodycam view remains visible.

### 14. Peek Left/Right
- Two small controls: ◂ PEEK (68% bottom 88%) and PEEK ▸ (88% bottom 88%), 48px
- Tap/hold: body moves 0.35m lateral, camera moves with body, weapon follows, physically believable, not camera-only

### 15. Reload Button
- Smaller, less prominent than fire: 50px at 20% bottom 38%, faint semi-transparent
- Uses existing realistic reload: camera tilts -0.35 rad, weapon tilts 0.4 rad, mag out/in visible, hands to equipment, manipulates weapon, returns ready

### 16. Flashlight/Torch
- Small subtle: 48px at 6% bottom 38%
- Weapon-mounted SpotLight 20m, PI/6.5 cone, casts shadows (if gfx enabled), interacts with smoke/dust/glass, reveals beam in fog, useful because darkness intentional

### 17. Recommended Layout
```
LEFT SIDE: invisible movement region (42%)
RIGHT SIDE: fire (largest), ads, peek left/right, reload, flashlight
Spacing allows one control without accidental touch, sparse, environment dominates
```

### 18. UI Customization — Settings → Customize UI
- Move/reposition buttons via drag in customize mode (dashed border)
- Opacity slider 0.1-1.0
- Fire size 0.7-1.5x, secondary size 0.7-1.4x
- Reset to default
- Save custom layout to localStorage `breach_ui_layout` with normalized coords (x=left/width, y=top/height) — works across aspect ratios
- Persists between sessions

### 19. Settings Menu
- Gear icon extreme top-right 36px, subtle rgba(0,0,0,0.35), blur
- Panel with sections:
  - Gameplay: sensitivity (look 0.2-3, ADS 0.2-2, move 0.5-2)
  - Controls: customize toggle, opacity, size, reset
  - Audio: master, effects, env, breathing (gains)
  - Display: graphics quality low/mid/high, bodycam effects toggle, shadows toggle, current sector debug, reset progression
- Touch-friendly, not desktop app

### 20. UI Opacity Low-Profile
- Default 0.55, semi-transparent, minimal, faint, no huge solid circles, no neon, no thick borders, no permanent tutorial text, no floating damage indicators — gameplay first, interface second

### 21. First-Time Tutorial
- Brief contextual bubbles, not giant window covering game:
  - "Touch & drag left side to move" (left bottom)
  - "Drag right side to look around" (right bottom)
  - "Tap to fire — largest button" (right 22%)
  - "Aim through optic — localized zoom" (right 28%)
  - "Peek around corners — body moves" (right 12%)
- Each 2.2s, disappears, flag `firstTime=false` saved, not repeated every level

### 22. No Permanent Labels
- After tutorial, no MOVE/SHOOT/RELOAD labels — minimalist icons (FIRE, ADS, ◂ PEEK, PEEK ▸, ↻, ☼)

### 23. Responsive UI
- Normalized positioning (percent), safe-area-inset env(), viewport-fit=cover, touch-action:none, overscroll-behavior:none, handles notches/rounded corners, landscape changes via resize listener

### 24. Mobile Performance
- LOD: reduced texture 256 vs 512, shadow map 512 vs 1024/2048 on low, pixelRatio min(devicePixelRatio, 1.2) vs 1.8
- Occlusion: Three.js frustum culling
- Asset streaming: procedural, no external
- Efficient lighting: 3 streetlights vs 4, PointLight 1.0 vs 1.6
- Efficient particles: smoke 6 spheres low vs 10 mid/high, 5/6 segments vs 12
- Selective physics: simple AABB colliders, not full engine
- AI throttling: vision 11m vs 12m, hearing 16 vs 18, search points 3 vs 4, state times shorter
- Instancing: furniture still but less count (5 vs 6)
- Shadows toggleable
- Bodycam effects toggleable
- Distant simulation reduced (future: lower tick rate)

### 25. Mobile Input Preserves Bodycam
- Touch invisible/subtle, does not destroy realism
- Camera still behaves physical on run/turn/stop/aim/fire/reload/peek/stairs/injury — touch is control method, not floating viewpoint

### 26. Level Progression Testable
- Internal `currentProgressionStage` 0-19, `Level_01..Level_20`, `currentProgressionStage +=1` on complete, no level-select UI, backend vs frontend separation

### 27. Failure & Restart
- On death: transition overlay "DOWN" + "Tap to retry operation" — tap retries current stage, does not skip, does not go to level-select

### 28. Save Progress
- localStorage: `breach_progression` (current stage), `breach_settings` (sensitivity, audio, gfx, firstTime), `breach_ui_layout` (normalized positions), completed implicit via current stage
- Reopening after Stage 4 returns to Stage 4, not beginning, unless reset

### 29. No Full Campaign Reveal
- Never shows "20 LEVELS", "LEVEL 1/20", "LEVEL 2 UNLOCKED", "19 MORE LEVELS" — feels like progression journey, discovers more by playing

### 30. Final Experience
- Rotate phone → bodycam activates → enter operation → hear something → check doorway → aim (localized optic) → fire → reload → clear → 2-3s transition → outside next structure → approach → enter → next encounter — no arcade level menu

### 31. Architecture Separation
```
GAME LOGIC (BuildingGenerator, EnemyManager, WeaponSystem)
  ↓
PLAYER SYSTEM (Player, BodyCamera, health, stamina)
  ↓
INPUT ABSTRACTION (input.move, input.lookDelta, lean, sprint, aiming)
  ↓
MOBILE TOUCH INPUT (moveRegion, lookRegion, fire/ads/peek/reload/flash buttons, touch identifiers, normalized coords)
  ↓
UI (HUD, scope-overlay, settings, tutorial, transition, orientation gate)
```
Game logic independent from control scheme.

### 32. Acceptance Test — Verified

**Orientation:**
- Portrait displays orientation instruction with rotating phone animation
- Landscape allows gameplay, auto-continues

**Progression:**
- No level-selection screen exists
- Player starts at beginning automatically (ENTER OPERATION)
- Cannot select future stages
- Completing stage auto-advances after 2-3s with transition bar
- Next stage begins outside building at gap entrance
- Player physically enters building

**Controls:**
- No visible permanent joystick — left region invisible
- Left side movement, right side camera
- Fire largest (86px vs 48-52px)
- ADS exists (52px)
- Peek left/right exists (48px)
- Reload exists (50px)
- Flashlight exists (48px)
- Buttons subtle semi-transparent 0.55 default

**Customization:**
- Settings gear top-right exists
- UI customization works (ENABLE/DISABLE, drag buttons, dashed border)
- Buttons repositionable (normalized x,y saved)
- Opacity adjustable (slider)
- Size adjustable (fire + secondary)
- Reset works
- Settings persist via localStorage

**Scope:**
- ADS does not cover entire screen — circular 180px overlay
- Optic localized magnification 2.8x inside radius 0.14
- Surrounding bodycam view remains visible
- Second renderer with FOV 18 vs 78 for true magnification, plus shader magnification

**Realism:**
- Mobile controls do not destroy bodycam — bob, inertia, shake, breathing, recoil preserved
- Weapon physically connected — sway, lag, recoil, reload tilt
- Camera physical — spring-damper, inertia, motion blur, exposure
- Reload immersive — mag visible, camera tilt, body connected
- Flashlight interacts — SpotLight shadows, smoke/dust/glass

**Performance:**
- Responsive on mobile — pixelRatio 1.2, texture 256, shadow 512, smoke 6 spheres, efficient colliders
- UI does not cause overhead — pointer-events none except buttons, backdrop-filter blur only on controls
- Viewport handled — viewport-fit=cover, safe-area-inset, touch-action:none, overscroll-behavior:none, resize listener

---

## Original Systems Preserved

All hyper-realistic systems from previous build preserved: PBR materials with imperfections, physically believable lighting, body-camera physics, heavy responsive movement, breathing, weapon physical attachment, enemy AI with vision/hearing/uncertainty/states/tactics/communication/searching/injury/ragdoll, dust/impacts per surface, volumetric smoke, flashbang, frag, flashlight, day/night, wind, audio propagation, casing sounds per material, bodycam mic overload, doors/windows, minimal HUD.

See `ARCHITECTURE.md` and `CONTROLS.md` for details.

---

## How to Test on Phone

1. Open URL in mobile browser (Chrome/Safari)
2. Hold vertically — see "ROTATE YOUR PHONE" gate
3. Rotate to landscape — gate hides, start screen appears
4. Tap ENTER OPERATION — spawns outside building at exterior gap, must walk toward and enter
5. Left side drag to move, right side drag to look, FIRE largest, ADS shows localized circular optic with magnified view inside, surrounding view visible
6. Peek left/right moves body+camera, reload shows mag, flashlight toggles SpotLight
7. Clear hostiles + package if present → transition 2.6s → outside next building → physically enter
8. Settings gear top-right → customize UI → drag buttons → opacity/size sliders → persists
9. Die → tap to retry same stage, not level select
10. Close browser → reopen → continues from current stage

---

## Files

- `index.html` — Mobile-first single file, ~110KB, Three.js 0.160, EffectComposer, custom BodyCamShader with ADS localized magnification, scopeRenderer FOV 18, touch input abstraction, progression hidden, orientation gate, UI customization
- `ARCHITECTURE.md` — Cloud delivery + systems
- `CONTROLS.md` — Controls details
- `package.json`

Built on Arena.ai Agent Mode, 2026-09-19.
