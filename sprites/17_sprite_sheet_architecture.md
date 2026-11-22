# 17 — SPRITE SHEET ARCHITECTURE
## Atlas Layout, Frame Organization, and Metadata System

> **Purpose:** Define sprite atlas dimensions, frame organization, metadata format, and packing strategy.  
> **Related:** [16_sprite_reconstruction.md](16_sprite_reconstruction.md), [19_sprite_animation_system.md](19_sprite_animation_system.md)

---

## 17.1 ATLAS LAYOUT PRINCIPLES

A sprite atlas (sprite sheet) packs multiple animation frames into one texture.

**Three layout types:**

```
TYPE A: STRIP (single row)
  ┌────┬────┬────┬────┬────┬────┬────┬────┐
  │ F0 │ F1 │ F2 │ F3 │ F4 │ F5 │ F6 │ F7 │
  └────┴────┴────┴────┴────┴────┴────┴────┘
  
  Pros: Simple to implement
  Cons: Wastes space for different-sized frames, long atlas
  Use: Single animation only

TYPE B: GRID (rows of animations)
  ┌────┬────┬────┬────┐  ← Animation 1 (Idle, 4 frames)
  │ I0 │ I1 │ I2 │ I3 │
  ├────┼────┼────┼────┤  ← Animation 2 (Walk, 4 frames)
  │ W0 │ W1 │ W2 │ W3 │
  ├────┼────┼────┼────┤  ← Animation 3 (Attack, 4 frames)
  │ A0 │ A1 │ A2 │ A3 │
  └────┴────┴────┴────┘
  
  Pros: Easy to manage, predictable frame addresses
  Cons: All frames same size (wastes space for small frames)
  Use: Uniform frame size animations

TYPE C: PACKED (MaxRects algorithm)
  ┌──────────┬──┬────────┐
  │  Large   │S0│ Walk 0 │
  │  Frame   ├──┤        │
  │          │S1├────────┤
  ├──────────┤  │ Walk 1 │
  │ Idle 0   │  │        │
  └──────────┴──┴────────┘
  
  Pros: Maximum texture efficiency
  Cons: Complex metadata required
  Use: Final production atlas
```

---

## 17.2 FRAME DIMENSIONS STANDARDIZATION

```
STANDARD FRAME SIZES (choose one for a character):

Body sprites:
  Small character:    48 × 64 px
  Standard:           64 × 96 px  ← Recommended
  Standard HD:        128 × 192 px (2× scale)

Head sprites:
  Standard:           32 × 40 px
  Standard HD:        64 × 80 px

Weapon sprites:
  One-hand:           32 × 48 px
  Two-hand/Staff:     48 × 64 px
  Bow:                48 × 48 px
  Spear/Long:         32 × 80 px

Shadow sprite:
  Standard:           48 × 24 px (wider than tall)

PADDING:
  Between frames: 2 pixels (prevents texture bleeding)
  Atlas border:   2 pixels minimum
```

---

## 17.3 METADATA JSON FORMAT

Complete specification for ROCHAR sprite metadata:

```json
{
  "schema": "rochar-sprite-v1.0",
  "atlas": "character_knight_body.png",
  "atlas_width": 512,
  "atlas_height": 512,
  "frame_width": 64,
  "frame_height": 96,
  "pivot": {
    "x": 0.5,
    "y": 1.0,
    "description": "0.5 = horizontal center, 1.0 = bottom of frame (foot anchor)"
  },
  "character_id": "knight_male_base",
  "directions": ["S", "SW", "W", "NW", "N", "NE", "E", "SE"],
  "mirrored_directions": {
    "SW": "SE",
    "W": "E",
    "NW": "NE"
  },
  "animations": {
    "idle": {
      "frame_count": 4,
      "fps": 4,
      "loop": true,
      "priority": 0,
      "directions": {
        "S":  {"frames": [0, 1, 2, 3],   "offsets": [[0,0],[0,-1],[0,-1],[0,0]]},
        "N":  {"frames": [4, 5, 6, 7]},
        "E":  {"frames": [8, 9, 10, 11]},
        "NE": {"frames": [12, 13, 14, 15]},
        "SE": {"frames": [16, 17, 18, 19]}
      }
    },
    "walk": {
      "frame_count": 8,
      "fps": 8,
      "loop": true,
      "priority": 1,
      "directions": {
        "S":  {"frames": [20, 21, 22, 23, 24, 25, 26, 27]},
        "N":  {"frames": [28, 29, 30, 31, 32, 33, 34, 35]},
        "E":  {"frames": [36, 37, 38, 39, 40, 41, 42, 43]},
        "NE": {"frames": [44, 45, 46, 47, 48, 49, 50, 51]},
        "SE": {"frames": [52, 53, 54, 55, 56, 57, 58, 59]}
      }
    },
    "attack_1h": {
      "frame_count": 7,
      "fps": 15,
      "loop": false,
      "priority": 5,
      "events": [
        {"frame": 3, "event": "hit_connect", "description": "Weapon contact frame"}
      ],
      "directions": {
        "S":  {"frames": [60, 61, 62, 63, 64, 65, 66]}
      }
    },
    "hurt": {
      "frame_count": 3,
      "fps": 20,
      "loop": false,
      "priority": 8,
      "directions": {
        "S": {"frames": [80, 81, 82]}
      }
    },
    "dead": {
      "frame_count": 5,
      "fps": 8,
      "loop": false,
      "hold_last_frame": true,
      "priority": 10,
      "directions": {
        "S": {"frames": [90, 91, 92, 93, 94]}
      }
    }
  }
}
```

---

## 17.4 UV COORDINATE CALCULATION FROM METADATA

Converting frame index to UV coordinates for shader/renderer use:

```javascript
/**
 * Calculate UV coordinates for a specific sprite frame.
 * @param {Object} metadata - Loaded sprite metadata JSON
 * @param {number} frameIndex - Frame index from metadata
 * @returns {Object} {u0, v0, u1, v1} UV coordinates (0–1 range)
 */
function getFrameUV(metadata, frameIndex) {
    const cols = Math.floor(metadata.atlas_width / metadata.frame_width);
    const rows = Math.floor(metadata.atlas_height / metadata.frame_height);
    
    const col = frameIndex % cols;
    const row = Math.floor(frameIndex / cols);
    
    const fw = metadata.frame_width / metadata.atlas_width;    // frame width in UV
    const fh = metadata.frame_height / metadata.atlas_height;  // frame height in UV
    
    // Adjust for 2px padding
    const pad_u = 2 / metadata.atlas_width;
    const pad_v = 2 / metadata.atlas_height;
    
    return {
        u0: col * fw + pad_u,
        v0: row * fh + pad_v,
        u1: (col + 1) * fw - pad_u,
        v1: (row + 1) * fh - pad_v
    };
}

// Update shader UV uniforms
function setFrameUniforms(material, metadata, frameIndex) {
    const uv = getFrameUV(metadata, frameIndex);
    material.uniforms.uFrameOffset.value.set(uv.u0, uv.v0);
    material.uniforms.uFrameSize.value.set(uv.u1 - uv.u0, uv.v1 - uv.v0);
}
```

---

## 17.5 ATLAS SIZE SELECTION GUIDE

```
RECOMMENDED ATLAS SIZES:

Single animation (one character, all directions):
  → 256×256 (standard) or 512×256 (wider)

Single character, all animations:
  → 512×512 to 1024×1024

Multiple characters shared atlas:
  → 1024×1024 to 2048×2048 (mobile limit: 2048×2048)
  → Requires packed layout

MAXIMUM TEXTURE SIZES:
  Mobile WebGL (iOS/Android): 4096×4096 (iOS limit usually 4096)
  Desktop WebGL:              8192×8192 (most modern GPUs)
  Classic RO target:          512×512 per character (original scale)
  
RULE: Never exceed 2048×2048 for mobile-targeted content.
```

---

## 17.6 TRANSPARENT BORDER PADDING

Padding between frames prevents color bleeding during mip-mapping:

```
MINIMUM PADDING:
  Mip-level 0 (full res): 1 pixel minimum
  With mip-mapping:       4 pixels recommended
  For blurred edges:      8 pixels
  
PADDING STRATEGY:
  → Each frame has 2px empty border in the atlas
  → Atlas edges have 2px padding from each frame

IMPLEMENTATION IN PACKING:
  When using TexturePacker or manual packing:
  → Extrude mode: Extend edge pixels 2 times outward
  → This fills padding with adjacent color (prevents seams)
  
BLENDER EQUIVALENT:
  → Bake margin: 4 pixels
```

---

## 17.7 DIRECTION ORDERING CONVENTION

Standard direction order for arrays/indices in ROCHAR:

```
INDEX | DIRECTION | DEGREES | NOTES
  0   |     S     |    0°   | South (facing camera) — most detailed
  1   |    SW     |   45°   | 
  2   |     W     |   90°   | Often mirror of E
  3   |    NW     |  135°   | Often mirror of NE
  4   |     N     |  180°   | North (back to camera)
  5   |    NE     |  225°   |
  6   |     E     |  270°   |
  7   |    SE     |  315°   | Often mirror of SW

OPTIMIZATION: If mirroring W from E:
  Directions needed: S, SW, N, NE, E = 5 unique directions
  W  = flip E  horizontally
  NW = flip NE horizontally
  SE = flip SW horizontally
  → 37.5% fewer sprites needed
```
