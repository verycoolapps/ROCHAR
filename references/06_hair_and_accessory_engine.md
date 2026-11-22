# 06 — HAIR AND ACCESSORY ENGINE
## RO Hair Geometry, Accessory Systems, and Attachment Framework

> **Purpose:** Define hair geometry construction, accessory types, and attachment point system.  
> **Related:** [05_costume_and_equipment_architecture.md](05_costume_and_equipment_architecture.md), [07_color_and_material_language.md](07_color_and_material_language.md)

---

## 6.1 HAIR DESIGN PHILOSOPHY

Classic RO hairstyles are **angular, stylized, and silhouette-driven** — not physically simulated.

The goal is NOT realistic hair strands. The goal IS:
- An immediately recognizable hair shape from any game-view distance
- Clean angular planes that read as "hair" through shape alone
- Vivid color blocking with hand-painted texture
- Consistent appearance across all 8 directions (sprite system)

**The Angular Mass Principle:**  
RO hair is built as **geometric masses** — large flat planes and angular extruded forms — not individual strands or smooth curves.

---

## 6.2 HAIR TAXONOMY

### By Length Category

| Category | Description | Typical Polygon Budget |
|----------|-------------|----------------------|
| Short | Covers ear tops, neck exposed | 40–80 tris |
| Medium | To shoulder level | 80–150 tris |
| Long | Below shoulder to mid-back | 150–280 tris |
| Very Long | To waist | 280–500 tris |

### By Style Category

```
STRAIGHT VOLUME: Flat planes dropping from crown, smooth outline
SPIKY: Angular points projecting outward — common for male characters
SWEPT: Asymmetric, layered planes tilting to one side
BRAIDED: Defined rope-like forms (geometric cylinder pairs)
CURLY/WAVY: S-curved planes — angular approximation only
TWIN TAILS: Two symmetrical hanging volumes (female characters common)
WILD: Multiple overlapping angular planes at different angles
```

---

## 6.3 MULTI-LAYER HAIR MESH ARCHITECTURE

Build hair in **three stacked layers**:

```
LAYER 1: BASE LAYER (foundation)
  Purpose: Defines the primary hair volume and skull coverage
  Material: Opaque diffuse — the main hair color
  Polygon count: 60–70% of total hair budget
  
LAYER 2: OVERLAY LAYER (detail)
  Purpose: Adds secondary planes for hair tips, bangs, side strands
  Material: Alpha-tested (edges fade to transparent)
  Polygon count: 25–30% of total hair budget
  Position: Slightly above base layer (0.005–0.01 world units)

LAYER 3: HIGHLIGHT LAYER (optional)
  Purpose: Specular/shine baked into geometry as bright vertex color
  Material: Alpha-blended, additive blend possible
  Polygon count: 5–10% of total hair budget
  Used for: Glossy hair styles, magical glowing hair effects
```

---

## 6.4 HAIR GEOMETRY CONSTRUCTION STEPS

```
STEP 1: SKULL CAP
  → Create a dome mesh conforming to the top of the head
  → This is the foundation — all hair builds on top of this
  → Polygon count: 20–40 tris

STEP 2: PRIMARY VOLUME PLANES
  → Extrude or add flat planes for main hair direction
  → Short hair: 2–4 planes covering top and sides
  → Long hair: 4–8 planes including back-hanging sections
  → Each plane represents a "clump" of hair, not a strand

STEP 3: FRINGE/BANGS
  → Separate geometry for front-facing hair
  → Usually 2–4 triangular planes visible in front view
  → Key for character recognition (bang style = part of identity)

STEP 4: SIDE AND BACK VOLUMES
  → Planes that define hair boundary at ears, neck, back
  → Long hair: add "tail" planes hanging from back of skull cap

STEP 5: TIPS AND FLYAWAYS (overlay layer)
  → Small triangular alpha-tested planes at hair ends
  → Gives organic, non-mechanical silhouette to hair outline

STEP 6: VERTEX COLOR FOR SHADING
  → Darken base of hair at roots (0.6 brightness)
  → Brighten tips (1.0 brightness)
  → Add highlight streak to overlay layer (1.2–1.5 brightness for bloom effect)
```

---

## 6.5 HAIR MATERIAL SPECIFICATIONS

```glsl
// Hair material approach (simplified diffuse + vertex color)

// Vertex shader: pass vertex color to fragment
out vec4 vVertexColor;
vVertexColor = aVertexColor;

// Fragment shader for base hair layer (opaque):
vec4 hairColor = texture(uHairTexture, vTexCoord);
hairColor.rgb *= vVertexColor.rgb;  // vertex color shading
FragColor = hairColor;

// Fragment shader for overlay layer (alpha-tested):
vec4 hairColor = texture(uHairTexture, vTexCoord);
if (hairColor.a < 0.5) discard;
hairColor.rgb *= vVertexColor.rgb;
FragColor = hairColor;
```

**Texture approach:**
- One texture per hair style (not shared with body)
- Resolution: 64×64 to 128×128
- Color baked in, gradients painted manually
- Edge pixels: fully transparent (not premultiplied)
- Filtering: Nearest-neighbor for pixel art look

---

## 6.6 ACCESSORY TYPES AND VISUAL CATEGORIES

RO accessories span a wide range of visual complexity:

| Category | Examples | Typical Polygons |
|----------|---------|-----------------|
| Hats | Tophat, beret, brimmed hat, helmet | 60–200 tris |
| Wings | Angel wings, demon wings, butterfly | 40–150 tris |
| Horns | Devil horns, animal horns, crown spikes | 20–60 tris |
| Ribbons | Hair bow, back ribbon, decorative tie | 20–60 tris |
| Flowers | Flower crown, corsage, petal decoration | 20–80 tris |
| Masks | Face mask, eye mask, decorative mask | 40–120 tris |
| Animal ears | Cat ears, bunny ears, fox ears | 20–60 tris per pair |
| Glasses | Spectacles, goggles, monocle | 20–80 tris |
| Scarves/Necklaces | Neck wrap, pendant | 20–60 tris |
| Crowns | Tiara, king crown, laurel wreath | 60–150 tris |

---

## 6.7 HEADGEAR ATTACHMENT SYSTEM

Three headgear slots each have a defined attachment socket bone:

```
HEAD_TOP SLOT (HeadgearTop_Socket):
  Purpose: Hats, crowns, wings, horns, oversized accessories
  Bone position: +0.5H above head bone origin (crown of head)
  World position: At highest point of bare head/hair
  
  Design rule: Accessories at this slot MUST be recognizable from afar
  → Size: at least 0.4H × 0.4H bounding box to read at distance
  → Avoid tiny accessories that disappear at game view distance

HEAD_MID SLOT (HeadgearMid_Socket):
  Purpose: Glasses, eye masks, face accessories
  Bone position: +0.4H above head, forward +0.3H
  World position: At eye/face level, forward of head geometry
  
  Design rule: Accessories must not obscure face recognition
  → Semi-transparent or framing accessories preferred

HEAD_BOT SLOT (HeadgearBot_Socket):
  Purpose: Mouth masks, scarves, chin accessories, neck decorations
  Bone position: +0.1H above head bone, forward +0.3H
  World position: At mouth/chin level
```

---

## 6.8 WING GEOMETRY CONSTRUCTION

Wings are a commonly used RO accessory requiring special geometry:

```
FEATHERED WINGS:
  → Base: 1–2 large curved planes forming primary wing outline
  → Secondary: 3–5 smaller planes for feather tier separation
  → Material: Alpha-tested, feather texture with transparent edges
  → Size at HEAD_TOP: typically 1.5–2.5H wide when spread
  → Folded variant: same geometry, different bone pose

INSECT WINGS (butterfly/fairy):
  → 2–4 flat elliptical planes
  → Material: Semi-transparent, alpha-blended
  → Iridescent texture: can use vertex color gradient
  
DEVIL/ANGEL WINGS:
  → Rigid angular planes (demon style)
  → Or feathered construction above (angel style)
  → Color key: white/gold = angel, black/red = demon
```

---

## 6.9 HAIR COLOR CONVENTIONS IN RO

| Color | Character Archetype | Notes |
|-------|-------------------|-------|
| Black | Serious, strong, mature | Most common male color |
| Brown | Natural, balanced, friendly | Common for acolyte/merchant |
| Blonde/Gold | Heroic, noble, chosen | Knight, paladin archetypes |
| Silver/White | Sage, ancient power, special | Transcendent classes often |
| Red | Passionate, aggressive, fire | Assassin, some warrior |
| Blue | Magic, water, calm power | Mage, wizard archetypes |
| Green | Nature, forest, growth | Archer, hunter archetypes |
| Pink/Lavender | Feminine, playful, magical | Dancer, female classes |
| Purple | Mystic, arcane, powerful | High mage classes |

---

## 6.10 COMMON HAIR MISTAKES AND CORRECTIONS

| Mistake | Correction |
|---------|-----------|
| Hair built from tube/cylinder geometry | Use flat planes — angular mass approach |
| Smooth subdivision applied to hair | Keep flat shading on hair geometry |
| Hair Z-fights with head geometry | Move hair overlay layer +0.005 units outward |
| Transparency sorting artifacts | Render hair front-to-back, use alpha test for opaque core |
| Hair color too desaturated | RO hair is vivid — push saturation higher than natural |
| Hair merges with body silhouette | Increase contrast between hair color and body/clothing |
| Identical geometry for all 8 directions | Adjust visibility of some hair planes per direction |
| Hair accessory too small to read | Minimum 0.3H × 0.3H bounding box for HEAD_TOP items |
| Wing clip through body | Move wings attachment offset backward by 0.1H |
| Long hair clips through cape | Disable long hair render layer when GARMENT is equipped |
