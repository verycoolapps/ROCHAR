# 12 — UV MAPPING AND TEXTURE PIPELINE
## UV Island Strategy, Seam Placement, and Texture Atlas Organization

> **Purpose:** Define UV workflows, texture atlases, pixel density, and baking pipeline for RO characters.  
> **Related:** [11_topology_and_mesh_engineering.md](11_topology_and_mesh_engineering.md), [../references/07_color_and_material_language.md](../references/07_color_and_material_language.md)

---

## 12.1 UV PHILOSOPHY FOR RO CHARACTERS

Classic RO textures are **small and direct** — the texture IS the character's surface.  
Every texel visible on the model at game distance should convey useful information.

**Three UV priorities:**
1. **Face gets the most pixels** — the most detail-critical area
2. **Visible areas get more than hidden areas** — no wasted texel budget
3. **Power-of-two dimensions** — always

---

## 12.2 TEXTURE RESOLUTION GUIDE

| Character Type | Body Texture | Face Texture | Equipment |
|---------------|-------------|-------------|---------|
| Background NPC | 128×128 | Shared with body | 64×64 |
| Standard character | 256×256 | 128×128 (separate) | 128×128 |
| Player hero | 512×512 | 256×256 or packed | 256×256 |
| Cinematic only | 1024×1024 | 512×512 | 512×512 |

**Standard production recommendation:**  
Body: 256×256, Face: 128×128 or packed into body atlas

---

## 12.3 TEXTURE ATLAS ORGANIZATION

For a standard character with one body texture + one equipment texture:

```
BODY ATLAS (256×256):
  ┌──────────────────────────────────┐
  │   FACE         │   BACK HEAD     │  Row 1: Head (top 25%)
  │   64×64        │   32×32         │
  ├────────────────┴────────────────┤
  │   FRONT TORSO  │  BACK TORSO     │  Row 2: Torso (25%)
  │   64×48        │  48×48          │
  ├─────────────────────────────────┤
  │  ARM L  │ ARM R │ LEG L │ LEG R  │  Row 3: Limbs (25%)
  │  32×48  │ 32×48 │ 32×64 │ 32×64  │
  ├─────────────────────────────────┤
  │   HANDS        │   FEET          │  Row 4: Extremities (25%)
  │   32×32        │   32×32         │
  └─────────────────────────────────┘

EQUIPMENT ATLAS (256×256):
  Organized similarly by equipment slot
  Each item gets proportional space to its visual importance
```

---

## 12.4 MANUAL SEAM PLACEMENT GUIDE

Seams should be placed where they will be least visible and most natural:

```
HEAD:
  Primary seam: Ear hairline (hidden by hair in most poses)
  Secondary seam: Chin-to-crown going behind ear
  
  Avoid: Center of forehead, center of face

TORSO:
  Side seams: Along armpit-to-hip side lines
  → These are hidden in front/back view
  
  Avoid: Center of chest/back (most visible areas)

ARMS:
  Seam: Inner arm (armpit to elbow to wrist, inner side)
  → Hidden when arm is at rest position (held against body)
  
  Avoid: Outer arm (always visible)

LEGS:
  Seam: Inner leg (crotch to ankle, inner side)
  → Hidden when legs are together
  
  Avoid: Outer thigh (always visible)

HANDS:
  Seam: Between each finger (acceptable for game asset)
  → Palm side of thumb

FEET:
  Seam: Under foot (sole area, never seen)
  → Heel to toe along sole
```

---

## 12.5 UV UNWRAP WORKFLOW IN BLENDER

### Smart UV Project (Rapid Workflow)

```
STEP 1: Select all faces (A in Edit Mode)
STEP 2: UV → Smart UV Project
  Settings:
    Angle Limit: 66°
    Island Margin: 0.03
    Area Weight: 0.0
STEP 3: Review islands in UV editor
  → Identify any poorly unwrapped areas (severe stretching)
STEP 4: Manually re-seam and re-unwrap those areas
STEP 5: Pack Islands (UV > Pack Islands, margin 0.02)
```

### Manual Seam Workflow (Higher Quality)

```
STEP 1: Mark all seams
  → Edit Mode → Select edges to be seams
  → Edge > Mark Seam (Ctrl+E > Mark Seam)
  
STEP 2: Unwrap
  → Select all (A)
  → UV → Unwrap (U)
  
STEP 3: Check for stretching
  → UV Editor → Overlays → Stretch
  → Blue = no stretch, Red = high stretch
  → Identify and fix red areas by adding more seams or adjusting UVs

STEP 4: Scale islands for pixel density
  → Face area gets larger UV island (more pixels)
  → Back of head gets smaller island

STEP 5: Pack Islands
  → Blender built-in: UV > Pack Islands
  → Or use UVPackmaster add-on for better packing
```

---

## 12.6 LIGHTMAP UV CHANNEL (UV2)

The world terrain and some character elements use a second UV channel for lightmaps.

```
SETUP IN BLENDER:
1. In Properties > Object Data > UV Maps
2. Add new UV map: "UVMap2" (or "LightmapUV")
3. Select UVMap2 as active
4. Unwrap with Smart UV Project (angle: 90°, no overlaps allowed)
   → Lightmap UVs MUST NOT overlap
   → Overlapping UVs cause baking artifacts

LIGHTMAP UV RULES:
  → Every face must have a unique position in UV space
  → No UV islands can overlap
  → Leave 2–4 pixel margin between islands (at target resolution)
  → Islands should be roughly proportional to face area in world space

TYPICAL LIGHTMAP RESOLUTION:
  Character lightmap: 128×128 to 256×256
  World terrain tile: 512×512 to 1024×1024
```

---

## 12.7 PIXEL DENSITY CALCULATION

```
FORMULA:
  Texels per world unit = (texture_width × island_coverage) / (surface_area_in_world_units)

EXAMPLE:
  256×256 texture, face island covers 15% of UV space (0.15)
  Face surface area ≈ 0.08 × 0.09 m = 0.0072 m²
  
  Coverage area in texels: 256 × 256 × 0.15 = 9,830 texels
  Face surface = 0.0072 m²
  Texel density: √(9,830 / 0.0072) ≈ 1,168 texels/m
  
  At game distance (500 pixels/m visual), this is MORE than enough.
  At 300 pixel/m visual → 1168 / 300 = 3.9× oversampled → fine.

TARGET DENSITY:
  Characters: 500–1500 texels per meter (at 256×256 body texture)
  Higher = more detail but larger texture
  Lower = visible pixelation at close range
```

---

## 12.8 TEXTURE BAKING WORKFLOW IN BLENDER

For baking from high-poly to low-poly (cinematic pipeline):

```
SETUP:
1. Create high-poly version (can be subdivided for base)
2. Create low-poly target mesh (the game asset)
3. Position them exactly overlapping
4. In Render Properties → Render Engine → Cycles

BAKE DIFFUSE COLOR:
1. Select low-poly (active), then high-poly (selected)
2. In Properties → Render → Bake
3. Bake Type: Diffuse
4. Direct: OFF, Indirect: OFF, Color: ON
5. Selected to Active: ON
6. Extrusion: 0.02 (ray offset to catch nearby high-poly)
7. Max Ray Distance: 0.1
8. Click Bake

BAKE NORMAL MAP:
1. Same setup
2. Bake Type: Normal
3. Space: Tangent Space
4. Click Bake

SAVE TEXTURES:
  UV Editor → Image → Save As
  Format: PNG (supports transparency)
  Bit depth: 8-bit for standard, 16-bit for normals
```

---

## 12.9 PIXEL BLEEDING AND PADDING

```
WHAT IS PIXEL BLEEDING:
  When a UV island edge renders incorrectly, the adjacent blank UV space
  can appear as a colored line at the island border.

PREVENTION:
1. Add 2–4 pixel padding when saving textures
   → In Blender: Image > Pack UV Islands with margin 0.02–0.04
   
2. Dilate painted edges outward
   → In image editor: dilate/grow edge colors by 2–4 pixels
   → This fills the transparent border area with adjacent color

3. Mip-map consideration:
   → Even at 1× scale, mip-mapping can sample adjacent islands
   → Always maintain at least 2 pixel gap at target mip level

BLENDER BAKING:
  → Bake settings → Margin: set to 4–8 pixels
  → This automatically extends baked textures beyond UV islands
```

---

## 12.10 UV QUALITY VALIDATION

Before exporting:

```
□ No UV islands overlap (critical — causes incorrect texture mapping)
□ No UV islands outside the 0–1 UV space (causes texture wrapping)
□ Face receives appropriate pixel density (largest island)
□ No severe stretch (Stretch overlay < 0.3 for most areas)
□ Seams are in inconspicuous locations
□ Lightmap UV (UV2) has zero overlapping islands
□ Padding is sufficient (2–4 pixels minimum)
□ Texture is power-of-two: 64, 128, 256, 512, 1024
□ Transparent regions have dilated edge color (no hard cutoff lines)
```
