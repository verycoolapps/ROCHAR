# 03 — RO CHARACTER ANATOMY
## Stylized Humanoid Body Construction Guide

> **Purpose:** Guide construction of simplified humanoid anatomy for RO characters.  
> **Related:** [02_character_proportion_system.md](02_character_proportion_system.md), [../modeling/10_low_poly_modeling.md](../modeling/10_low_poly_modeling.md), [../modeling/11_topology_and_mesh_engineering.md](../modeling/11_topology_and_mesh_engineering.md)

---

## 3.1 ANATOMY PHILOSOPHY

Classic RO characters use **stylized simplified anatomy** — not realistic anatomical structure.

The goal is NOT to replicate human anatomy with low resolution.  
The goal IS to create the **impression** of a living, expressive character using minimal geometry.

**The One-Purpose Rule:**  
Every edge loop in a low-poly character must justify its existence with either:
- A visible **silhouette contribution** at game distance, OR
- A **deformation support function** (joint articulation when animated)

If an edge loop does neither, remove it.

---

## 3.2 HEAD CONSTRUCTION

### Primary Head Shape

```
STEP 1: BASE FORM
  Start: UV sphere with 8 U-segments × 6 V-segments
  → Adjust to slightly taller than wide (H:W = 1.1:1.0)
  → Head fits in bounding box of (0.85H × 1.0H × 0.65H)

STEP 2: FACE REGION DEFINITION
  Front 40% of sphere geometry = face area
  → Flatten this region slightly forward (depth: ~0.6 × width)
  → This creates the slight face-projection vs cranium

STEP 3: JAW SHAPING
  Lower 30% of head: narrow toward chin
  → Female/neutral: soft round chin
  → Male: slightly more squared, angular jaw
  → Cheek width: widest point at ~60% from crown

STEP 4: CRANIUM
  Upper 60%: smooth dome
  → Slight forehead protrusion (+0.05H forward on front face)
```

### Facial Feature Geometry (Low-Poly)

```
EYES (4–8 polygons total):
  Option A: Small recessed areas in face geometry (2–4 polys each)
  Option B: Flat-shaded quads slightly embedded into face
  → NOT full anatomical eye socket detail
  → Color, pupil, lash detail conveyed by texture

NOSE (0–2 polygons):
  → Barely indicated — small forward vertex or simple 2-vert bridge
  → Many RO characters: no geometric nose at all
  → Nose shape conveyed entirely by texture painting

MOUTH (0–4 polygons):
  → Optional slight depression in lower face
  → Expression conveyed by texture, not by lip geometry
  → Smile lines: texture only

EARS (4–8 polygons each):
  → Simple extruded loop from head side geometry
  → Bottom of ear at nose level, top at eye level
  → Often fully hidden by hair — minimal detail acceptable
```

---

## 3.3 NECK CONSTRUCTION

```
SHAPE: Truncated cylinder, slightly narrower at top than bottom

DIMENSIONS:
  Diameter (front view): 0.15–0.20 H
    Female: 0.14–0.16 H
    Male:   0.18–0.22 H
  Height:   0.10–0.15 H

RING COUNT: 6–8 vertices per ring (matches head base loop)

CONNECTIONS:
  → Top ring: merges with lowest ring of head geometry
  → Bottom ring: merges with clavicle/chest ring
  → Edge flow continuous through head → neck → chest

ANGLE: Slightly forward — neck is not perfectly vertical
  → Tilt: approximately 5–10° forward from vertical
  → Creates more natural resting posture
```

---

## 3.4 TORSO CONSTRUCTION

### Upper Torso (Chest / Shoulders)

```
SHAPE: Barrel form, widest at shoulder, narrowing to waist

DIMENSIONS (front view):
  Top (shoulder level):  1.4–1.6 H wide
  Mid (chest level):     1.1–1.3 H wide
  Bottom (waist level):  0.9–1.0 H wide

DIMENSIONS (side view):
  Depth at chest:        0.5–0.6 H
  Depth at waist:        0.45 H
  Slight chest forward protrusion

KEY TOPOLOGY RINGS:
  Ring A: Shoulder level (also defines arm socket holes)
  Ring B: Mid-chest level
  Ring C: Waist level (narrowest — the pinch point)

POLYGON COUNT: 60–100 triangles for upper torso body
```

### Lower Torso (Hips / Pelvis)

```
SHAPE: Slight widening from waist to hip socket level

DIMENSIONS:
  Top (waist):  0.9–1.0 H (matches upper torso waist ring)
  Bottom (hip): 1.0–1.2 H

COVERAGE NOTE:
  This region is typically covered by armor, skirts, or robes.
  Minimum viable geometry is acceptable when covered.
  Save polygon budget for visible areas.
```

---

## 3.5 ARM CONSTRUCTION

```
SEGMENTS (proximal to distal):
  Shoulder Socket → Upper Arm → Elbow → Forearm → Wrist → Hand

UPPER ARM:
  Shape: Cylinder, slight flare at shoulder
  Diameter: 0.15 H (female) to 0.20 H (heavy male)
  Length:   0.60–0.70 H
  Rings:    2 (shoulder, elbow) + 1 extra at elbow bend

FOREARM:
  Shape: Cylinder, tapers toward wrist
  Diameter top: 0.13–0.17 H
  Diameter wrist: 0.10–0.12 H
  Length: 0.50–0.60 H

HAND:
  Shape: Box form with slight rounded corners
  Width:  0.25–0.35 H
  Depth:  0.10 H
  Height: 0.20 H
  Fingers: Optional — often single merged mesh or 3-part simplified form
  Thumb:  Separate extruded form from side of palm

ELBOW DEFORMATION TOPOLOGY:
  → 2–3 edge loops around elbow joint for clean 90° bend
  → Loops oriented perpendicular to the arm's bend axis
  → Loop density: more loops = smoother deformation
  → Minimum: 2 loops (acceptable for game distance)
  → Preferred: 3 loops (clean deformation visible at close range)
```

---

## 3.6 LEG CONSTRUCTION

```
SEGMENTS (proximal to distal):
  Hip Socket → Thigh → Knee → Shin → Ankle → Foot

THIGH:
  Shape: Cylinder, slight flare at hip socket
  Diameter: 0.20–0.25 H
  Length:   0.65–0.75 H
  Rings:    2 (hip, knee) + 1 at knee

SHIN:
  Shape: Cylinder, tapers toward ankle
  Diameter top: 0.18–0.22 H
  Diameter ankle: 0.12–0.14 H
  Length: 0.55–0.65 H

KNEE TOPOLOGY:
  → 2 edge loops for knee articulation
  → Optional: diamond topology at kneecap if visible (no armor/greaves)

FOOT:
  Shape: Box form + slightly rounded toe
  Length: 0.35–0.45 H
  Height: 0.10–0.15 H
  Width:  0.15 H
  Toe: Slight rounded extrusion at front

ANKLE:
  → Smooth transition from cylindrical shin to box foot
  → 1 extra edge loop at ankle for boot/shoe deformation
```

---

## 3.7 BODY MASS DISTRIBUTION BY CLASS

| Class Type | Chest Mass | Arm Mass | Leg Mass | Key Visual |
|-----------|-----------|----------|----------|-----------|
| Swordsman/Knight | Heavy | Heavy | Moderate | Plate armor expands silhouette significantly |
| Crusader | Very Heavy | Heavy | Heavy | Full plate — broadest silhouette |
| Mage/Wizard | Slim | Slim | Slim | Robe conceals most leg geometry |
| Sage | Slim | Slim | Moderate | Book/staff — intellectual feel |
| Archer/Hunter | Standard | Lean | Athletic | Light leather, agile stance |
| Bard/Dancer | Slim | Standard | Slim | Performer — elegant posture |
| Thief/Assassin | Lean | Lean | Athletic | Speed-suggesting compact build |
| Rogue | Lean | Standard | Standard | Subtle — hidden beneath garments |
| Acolyte/Priest | Standard | Slim | Standard | Robe silhouette dominates |
| Monk | Stocky | Heavy | Stocky | Bare arms show muscle mass |
| Merchant/Blacksmith | Stocky | Heavy | Stocky | Trade class — broad and grounded |
| Alchemist | Standard | Standard | Standard | Labcoat/scholar look |

---

## 3.8 FACIAL ANATOMY SIMPLIFIED SPECIFICATION

For textured character face with low-poly geometry:

```
TOTAL FACE POLYGON BUDGET: 60–120 triangles

GEOMETRY PROVIDES:
  → Smooth rounded overall cranial form
  → Slight eye socket depression (2–4 polys each)
  → Optional bridge of nose (2 extra polys)
  → Optional lip area slight indent (4 polys)
  → Soft chin form

TEXTURE PROVIDES (painted directly):
  → Eye color, pupil shape, iris detail
  → Eyelash/eyelid indication
  → Eyebrow shape and color
  → Blush marks (class-specific — common in RO)
  → Lip color and shape
  → Skin tone gradient from forehead to jaw
  → Freckles, marks (character-specific)

GEOMETRY DOES NOT NEED:
  → Detailed nostril geometry
  → Individual teeth
  → Detailed iris geometry
  → Wrinkle or age lines
  → Pore-level surface detail
  → Philtrum (groove above upper lip)
```

---

## 3.9 ANATOMY VALIDATION CHECKLIST

Before proceeding to UV mapping or rigging, verify:

```
□ Head is proportionally larger than realistic (1:4 minimum H:B)
□ No visible seam or hard crease at head-neck junction
□ Shoulder width reads correctly from front silhouette
□ Arms hang naturally at sides in T-pose (not floating outward)
□ Waist is narrower than both shoulders and hips
□ Legs are parallel and correctly separated (not merged)
□ Feet are flat on the XZ ground plane (not floating, not embedded)
□ Mesh has no inverted normals (Blender: Overlay > Face Orientation → all blue)
□ No non-manifold edges (Blender: Select > Select All By Trait > Non-Manifold)
□ No duplicate vertices (Blender: Mesh > Clean Up > Merge by Distance)
□ Character reads from front, back, and both side silhouettes
□ Polygon budget within class limit (see: ../modeling/10_low_poly_modeling.md)
□ All edge loops justify their existence (silhouette OR deformation)
□ Arm length is appropriate — wrist level at approximately hip bone height
□ Eyes are on the same horizontal level (front view)
```

---

## 3.10 QUICK REFERENCE — POLYGON ALLOCATION

| Body Part | Low Budget | Standard | Detailed |
|-----------|-----------|----------|---------|
| Head | 80 tris | 150 tris | 250 tris |
| Neck | 20 tris | 32 tris | 48 tris |
| Torso | 60 tris | 100 tris | 160 tris |
| Each Arm | 40 tris | 80 tris | 120 tris |
| Each Leg | 40 tris | 80 tris | 120 tris |
| Hands | 20 tris | 40 tris | 80 tris |
| Feet | 20 tris | 32 tris | 48 tris |
| **TOTAL (body)** | **~360 tris** | **~614 tris** | **~1,026 tris** |

Equipment, hair, and clothing are additional and have their own budgets.  
See: [../modeling/10_low_poly_modeling.md](../modeling/10_low_poly_modeling.md)
