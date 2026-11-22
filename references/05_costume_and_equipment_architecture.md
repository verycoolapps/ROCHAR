# 05 — COSTUME AND EQUIPMENT ARCHITECTURE
## Modular Costume Construction and Equipment Attachment System

> **Purpose:** Define RO's equipment slot system, modular costume construction, and attachment architecture.  
> **Related:** [06_hair_and_accessory_engine.md](06_hair_and_accessory_engine.md), [../modeling/13_modular_character_construction.md](../modeling/13_modular_character_construction.md)

---

## 5.1 RO EQUIPMENT SLOT SYSTEM

Classic Ragnarok Online defines the following equipment slots:

| Slot ID | Slot Name | Visual Location | Notes |
|---------|-----------|----------------|-------|
| `HEAD_TOP` | Upper Headgear | Above/on top of hair | Hat, crown, wings |
| `HEAD_MID` | Middle Headgear | Eye/face level | Mask, glasses, eye patch |
| `HEAD_BOT` | Lower Headgear | Mouth/chin level | Scarf, mouth mask |
| `ARMOR` | Body Armor | Chest and torso | Full torso coverage |
| `GARMENT` | Garment/Cape | Back/shoulder | Cape, wings, back decoration |
| `FOOTGEAR` | Footwear | Feet | Boots, shoes, sandals |
| `ACCESSORY_L` | Left Accessory | Wrist/hand area | Bracelet, ring |
| `ACCESSORY_R` | Right Accessory | Wrist/hand area | Bracelet, ring |
| `WEAPON` | Weapon (Primary) | Right hand dominant | Sword, staff, bow |
| `SHIELD` | Shield / Off-hand | Left hand | Shield, book, instrument |

**Weapon slot can further subdivide by weapon type:**
- One-handed weapon (pairs with shield)
- Two-handed weapon (occupies both hand slots)

---

## 5.2 MODULAR COSTUME CONSTRUCTION

The modular approach allows equipment to be independently swapped without rebuilding the character.

### Layer Stack (render/composite order)

```
LAYER 1: BASE BODY MESH (always present)
  → Bare skin geometry
  → This is the "null state" — no visible equipment

LAYER 2: BODY ARMOR
  → Fits over or replaces parts of base body
  → Example: plate chest replaces bare torso region

LAYER 3: GARMENT/CAPE
  → Attached behind shoulder/collar bone region
  → Cloth simulation or rigid (class-dependent)

LAYER 4: FOOTGEAR
  → Replaces or overlays bare feet
  → Boot height varies (ankle/knee/thigh high)

LAYER 5: WEAPON (right hand)
  → Parented to right hand bone

LAYER 6: SHIELD (left hand)
  → Parented to left hand/forearm bone

LAYER 7: ACCESSORIES (L/R)
  → Parented to respective wrist bones

LAYER 8: HEADGEAR (HEAD_BOT)
  → Parented to head bone, neck area

LAYER 9: HEADGEAR (HEAD_MID)
  → Parented to head bone, face area

LAYER 10: HEADGEAR (HEAD_TOP)
  → Parented to head bone, crown area

LAYER 11: HAIR MESH
  → Parented to head bone, behind headgear
```

---

## 5.3 TRANSFORM HIERARCHY (EQUIPMENT TO SKELETON)

```
ROOT (Hips)
│
├── Spine → Chest
│   │
│   ├── [ARMOR mesh parented here or to Hips]
│   ├── [GARMENT mesh parented to Chest]
│   │
│   ├── Shoulder.L → UpperArm.L → Forearm.L → Hand.L
│   │   ├── [SHIELD parented to Hand.L]
│   │   └── [ACCESSORY_L parented to Forearm.L or Hand.L]
│   │
│   └── Shoulder.R → UpperArm.R → Forearm.R → Hand.R
│       ├── [WEAPON parented to Hand.R]
│       └── [ACCESSORY_R parented to Forearm.R or Hand.R]
│
├── [HEAD_TOP attached to: HeadgearTop_Socket bone]
├── [HEAD_MID attached to: HeadgearMid_Socket bone]
├── [HEAD_BOT attached to: HeadgearBot_Socket bone]
│   └── All 3 socket bones are children of Head bone
│
├── UpperLeg.L → LowerLeg.L → Foot.L
│   └── [FOOTGEAR_L parented to Foot.L]
│
└── UpperLeg.R → LowerLeg.R → Foot.R
    └── [FOOTGEAR_R parented to Foot.R]
```

### Headgear Socket Bone Positions (relative to Head bone origin)

```
HeadgearTop_Socket:
  Position offset: (0, 0, +0.5H) — crown of head
  
HeadgearMid_Socket:
  Position offset: (0, +0.4H, +0.1H) — eye/face level

HeadgearBot_Socket:
  Position offset: (0, +0.3H, -0.1H) — mouth/chin level
```

---

## 5.4 EQUIPMENT MESH SPECIFICATIONS

### Armor (ARMOR slot)

```
Mesh type: Closed mesh enclosing torso
UV: Independent UV space (not shared with body)
Material: Class-specific (metal = environment tint + flat diffuse)
Clipping: May clip slightly into base body — acceptable if not visible
Polygon budget: 100–300 tris (simple plate), 300–600 tris (ornate)
```

### Cape/Garment (GARMENT slot)

```
Mesh type: Open mesh (one-sided cloth panels)
UV: Independent UV
Material: Cloth-type (no specular, vertex color for crease shading)
Attachment: Parent to Chest bone, may also attach to collar area
Rigid variant: Simple flat quad for distance rendering
Polygon budget: 40–120 tris
```

### Footgear (FOOTGEAR slot)

```
Mesh type: Closed mesh over foot/ankle/leg
UV: Independent UV, may share atlas with armor
Heights:
  Ankle boot:    covers foot + 0.1H above ankle
  Mid boot:      covers foot + 0.3H above ankle
  Thigh boot:    covers foot to mid-thigh (requires extra geometry)
Polygon budget: 40–100 tris per boot
```

### Weapon (WEAPON slot)

```
Attachment point: Right hand palm — Hand.R bone
Grip position: Weapon handle center at palm
Orientation: Varies by weapon type

SWORD:    Blade points upward along +Y of hand, edge faces forward
STAFF:    Held vertically, top extends above head when at rest
BOW:      Held in left hand (note: swap to Hand.L for archers)
HAMMER:   Handle along +Y of hand, head to side

Polygon budget: 60–300 tris (sword simple to ornate)
```

### Shield (SHIELD slot)

```
Attachment: Left forearm or hand — Hand.L/Forearm.L
Position: Faces outward from body
Face: Shield face visible when facing camera
Back: Can be lower polygon (not seen normally)
Polygon budget: 40–150 tris
```

---

## 5.5 MATERIAL COMPATIBILITY RULES

Equipment materials must be **visually compatible** with the base character material.

```
RULE 1: Texture filtering must match
  → All equipment textures should use the same filtering mode
  → Nearest-neighbor character texture + bilinear equipment = visual mismatch

RULE 2: Lighting model must match
  → All equipment uses same shader family (classic diffuse)
  → No equipment piece should introduce PBR on a flat-shaded character

RULE 3: Color temperature consistency
  → Warm ambient (#E8D8C8) on character = same warm ambient on equipment
  → Never mix warm-lit character with cold-lit equipment

RULE 4: Vertex color applied consistently
  → If base body uses vertex color tinting, so should equipment
  → Prevents equipment from looking "pasted on"
```

---

## 5.6 CLIPPING PREVENTION STRATEGIES

Equipment clipping through the base mesh is a common problem.

```
STRATEGY 1: OFFSET
  → Move equipment mesh outward by 0.005–0.02 world units
  → Enough to prevent z-fighting without visible gap

STRATEGY 2: BASE MESH REDUCTION
  → Under covered areas, simplify base mesh geometry
  → Remove polygon detail that will never be seen

STRATEGY 3: VISIBILITY MASKING
  → In 3D applications: hide base mesh faces under full-coverage armor
  → Keeps polygon budget low

STRATEGY 4: FITTED EQUIPMENT
  → Model equipment to closely follow body contour
  → Prevents most clipping at non-extreme pose angles

STRATEGY 5: ANIMATION LIMITS
  → For extreme poses, accept minor clipping
  → At game distance, minor clipping is not perceptible
```

---

## 5.7 JSON EQUIPMENT MANIFEST FORMAT

```json
{
  "character_id": "knight_male_base",
  "base_mesh": "meshes/knight_base.glb",
  "equipment": {
    "HEAD_TOP": {
      "id": "helm_iron_basinet",
      "mesh": "equipment/helms/iron_basinet.glb",
      "attachment_bone": "HeadgearTop_Socket",
      "position_offset": [0, 0, 0],
      "rotation_offset": [0, 0, 0],
      "scale": [1, 1, 1],
      "hide_hair": true,
      "material": "materials/iron_helm.mat.json"
    },
    "ARMOR": {
      "id": "armor_plate_iron",
      "mesh": "equipment/armor/plate_iron.glb",
      "attachment_bone": "Hips",
      "hide_base_torso": true,
      "material": "materials/plate_iron.mat.json"
    },
    "WEAPON": {
      "id": "sword_one_hand_iron",
      "mesh": "equipment/weapons/sword_iron_1h.glb",
      "attachment_bone": "Hand.R",
      "grip_offset": [0, 0.05, 0],
      "rotation_offset": [0, 0, 0],
      "material": "materials/iron_metal.mat.json"
    },
    "SHIELD": {
      "id": "shield_iron_round",
      "mesh": "equipment/shields/round_iron.glb",
      "attachment_bone": "Forearm.L",
      "position_offset": [0, 0.3, 0.05],
      "material": "materials/iron_metal.mat.json"
    },
    "GARMENT": null,
    "FOOTGEAR": {
      "id": "boots_iron_ankle",
      "mesh": "equipment/footgear/boots_iron_ankle.glb",
      "attachment_bone_L": "Foot.L",
      "attachment_bone_R": "Foot.R",
      "material": "materials/iron_metal.mat.json"
    }
  }
}
```

---

## 5.8 CLASS-SPECIFIC EQUIPMENT RESTRICTIONS

| Class | Cannot Equip | Required Restriction |
|-------|-------------|---------------------|
| Mage/Wizard | Heavy plate armor | Use only robe ARMOR mesh |
| Monk | Most weapons | Only fist/chain weapons |
| Archer | Swords, shields | Only bows + light armor |
| Thief/Assassin | Full plate, shields | Light armor only |
| Priest | Swords (most) | Staff/mace only |
| All casters | Two-handed physical weapons | Staff/wand only |

Always validate equipment set against class restriction table before rendering final character.
