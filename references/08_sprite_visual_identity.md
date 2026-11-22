# 08 — SPRITE VISUAL IDENTITY
## Classic RO Directional Sprite System and Animation Framework

> **Purpose:** Define directional sprite characteristics, animation frames, transparency, and visual consistency.  
> **Related:** [../sprites/16_sprite_reconstruction.md](../sprites/16_sprite_reconstruction.md), [../sprites/17_sprite_sheet_architecture.md](../sprites/17_sprite_sheet_architecture.md)

---

## 8.1 SPRITE SYSTEM OVERVIEW

Classic Ragnarok Online characters in the field are represented as **2D sprites on billboard geometry** — not 3D models. Understanding this system is fundamental to ROCHAR Pathway A.

**Sprite system architecture:**
- 8 directional facing variants per animation state
- Multiple animation states per character
- Multiple visual layers composited per frame (body, head, weapon, shadow)
- Billboard rendering in 3D world space
- Foot-anchored positioning

---

## 8.2 8-DIRECTION FACING SYSTEM

Directions are indexed 0–7, convention based on numpad directions:

```
         [N]  = Direction 0 (facing north, back to camera)
     [NW] [NE] = Directions 7, 1
   [W]           [E] = Directions 6, 2
     [SW] [SE] = Directions 5, 3
         [S]  = Direction 4 (facing south, face to camera)

NUMPAD CONVENTION:
  8 = North    (facing away)
  9 = NE
  6 = East     (facing right)
  3 = SE
  2 = South    (facing camera)
  1 = SW
  4 = West     (facing left)
  7 = NW

DIRECTION INDEX (0-based):
  0 = North
  1 = NE
  2 = East
  3 = SE
  4 = South (front-facing — most detailed, most seen)
  5 = SW
  6 = West (mirror of East)
  7 = NW (mirror of NE)
```

**Optimization:** West (6) and East (2) are often horizontal mirrors. Same for NW/NE and SW/SE.  
This reduces unique sprite art needed from 8 to 5 sets.

---

## 8.3 ANIMATION STATE LIBRARY

| State Name | Typical Frame Count | FPS | Loop | Priority |
|-----------|-------------------|-----|------|---------|
| `idle` | 3–5 frames | 4–6 | Yes | Low |
| `walk` | 8 frames | 8 | Yes | Medium |
| `run` | 6–8 frames | 12 | Yes | Medium |
| `attack_1h` | 6–8 frames | 15 | No | High |
| `attack_2h` | 8–10 frames | 12 | No | High |
| `attack_bow` | 10–12 frames | 12 | No | High |
| `cast` | 8–12 frames | 8 | No | High |
| `skill` | 10–16 frames | 12 | No | High |
| `hurt` | 3–4 frames | 20 | No | Very High |
| `dead` | 4–6 frames | 8 | No | Highest |
| `sit` | 3 frames | 2 | Yes | Low |
| `pickup` | 4 frames | 8 | No | Medium |
| `freeze` | 1–2 frames | 1 | Yes | — |
| `magnified` | Idle variant | — | — | — |

**Frame count note:** These are approximations for original game reference.  
Your implementation should define frame count based on animation quality requirements.

---

## 8.4 SPRITE LAYER COMPOSITING ORDER

Each rendered character frame is a **composite of multiple sprite layers**:

```
COMPOSITE ORDER (back to front):
  1. SHADOW  — ground shadow disc below character (always present)
  2. BODY    — the main character body sprite
  3. HEAD    — head sprite overlaid on body
  4. WEAPON  — weapon sprite overlaid on body/hands
  5. SHIELD  — shield sprite (left side)

LAYER SIZES (typical):
  Shadow:   64×32 to 96×48 (wider than tall, elliptical)
  Body:     48×64 to 96×128 depending on class and state
  Head:     32×32 to 48×48
  Weapon:   32×64 to 64×64 (varies by weapon type)
  Shield:   32×48 to 48×64
```

**Layer offset rules:**
- Head layer is positioned relative to the neckline of the body layer
- Weapon offset depends on hand position in the body animation frame
- Shield offset mirrors weapon on left side
- Each layer has its own pivot point for correct alignment

---

## 8.5 TRANSPARENCY METHODS

Classic RO sprites use **magenta transparency** (#FF00FF) as the primary transparency key.

### Method A: Magenta Chroma Key (Original)
```
Transparency indicator: Magenta pixels (R=255, G=0, B=255)
Processing:
  → Any pixel with R>230, G<30, B>230 → discard (fully transparent)
  → All other pixels → opaque

Advantages:
  → Simple, no alpha channel required
  → Compatible with 8-bit palette textures

Disadvantages:
  → Edge aliasing on character outlines
  → Color bleeding from magenta to nearby colors if not preprocessed
```

### Method B: Alpha Channel (Modern)
```
Transparency indicator: Alpha channel value
Processing:
  → Alpha > 128 → opaque (alpha test)
  → Alpha < 128 → discard

Advantages:
  → Clean anti-aliased edges
  → Gradual transparency possible

Recommendation: Convert magenta sprites to alpha PNG during pipeline.
See: ../sprites/16_sprite_reconstruction.md
```

---

## 8.6 SHADOW SPRITE

The character shadow is a **separate elliptical sprite** rendered beneath the character:

```
SHADOW SPRITE CHARACTERISTICS:
  Shape:    Soft-edged ellipse or flattened oval
  Color:    Dark (R:0, G:0, B:0) with partial transparency
  Alpha:    0.4–0.6 opacity (not fully opaque)
  Size:     Width = character_sprite_width × 0.6
            Height = shadow_width × 0.4
  Position: Centered under character foot anchor
  Render:   Before body sprite (back-to-front rendering)
  Rotation: Does NOT rotate with character direction
             Always aligned to world XZ plane

RENDERING NOTE:
  The shadow sprite creates the perception of ground contact.
  Without it, characters appear to float.
  Always include even if simplified to a single semi-transparent circle.
```

---

## 8.7 PIVOT POINT AND ANCHOR SYSTEM

Correct pivot placement ensures the character appears grounded:

```
BODY SPRITE PIVOT:
  X: Center of sprite horizontally (0.5)
  Y: Bottom of sprite at foot level (1.0 — bottom of image)
  
  Result: When positioned in world, the character's feet
          touch the ground at the specified world position.

HEAD SPRITE PIVOT:
  X: Center of head sprite
  Y: Neck connection point (variable — typically 0.7–0.8 from top)

WEAPON SPRITE PIVOT:
  X, Y: Handle/grip point of weapon
  
  Result: When overlaid on body, the weapon grip aligns
          with the character's hand position.
```

---

## 8.8 SPRITE CONSISTENCY REQUIREMENTS

All frames of a character must be visually consistent:

```
CONSISTENCY RULES:

1. COLOR IDENTITY
   → Same skin tone across ALL frames and directions
   → Equipment colors identical across all directions
   → Hair color consistent

2. SCALE CONSISTENCY
   → Character occupies same approximate height across all directions
   → Minor variation accepted (N/S may appear slightly taller/shorter)
   → Maximum variation: ±5% of character height

3. SILHOUETTE CONSISTENCY
   → Equipment must remain recognizable from all 8 directions
   → Hat must be visible from all 8 directions (may be small from E/W)
   → Weapon must be visible from all 8 directions

4. ANIMATION CONSISTENCY
   → Idle animation: very subtle (breathing, minor head tilt)
   → Walk cycle: smooth, non-jerky footfall timing
   → Attack: clear windup → strike → recovery phases

5. PIVOT CONSISTENCY
   → All frames of same animation use identical pivot point
   → Foot position must remain grounded (no floating between frames)
```

---

## 8.9 SCALE AND RESOLUTION STANDARDS

```
STANDARD CHARACTER SPRITE SIZES:

Small character (child/novice):
  Body: 48×80 pixels at 1× scale
  
Standard character:
  Body: 64×96 to 80×112 pixels at 1× scale

Large character / boss:
  Body: 96×128 to 128×160 pixels at 1× scale

RESOLUTION SCALING:
  1× scale: Original pixel resolution
  2× scale: 2× nearest-neighbor upscale for HD displays
  4× scale: 4× nearest-neighbor for modern HiDPI
  
  NEVER use bilinear/bicubic upscaling on pixel art sprites.
  Always use nearest-neighbor (Closest in Blender, NearestFilter in Three.js)

DISPLAY PIXELS PER WORLD UNIT:
  Classic RO: approximately 50–70 pixels per "tile" (1 world unit)
  Character at standard distance: approximately 70–100px height
```

---

## 8.10 DIRECTION-SPECIFIC VISUAL NOTES

```
SOUTH (facing camera — most detailed):
  → Full face visible
  → Front of costume visible
  → Weapon/shield clearly displayed
  → Most time in this direction — highest detail

NORTH (back to camera — least detailed):
  → Back of hair only
  → Back of cape/garment prominent
  → Weapon may be partially hidden
  → Less player time in this direction

EAST/WEST (profile views):
  → Half-face visible (profile)
  → Equipment depth apparent
  → Often mirrored (West = flipped East)
  → Side silhouette of weapon most prominent

DIAGONAL VIEWS (NE/NW/SE/SW):
  → 3/4 view — most dynamic reading
  → Partial face visibility
  → Armor depth visible
  → Most interesting visual angle for attack animations
```
