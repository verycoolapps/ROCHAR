# 11 — TOPOLOGY AND MESH ENGINEERING
## Edge Flow, Mesh Quality, and Deformation Topology

> **Purpose:** Define mesh quality standards, edge flow principles, and topology validation for RO characters.  
> **Related:** [10_low_poly_modeling.md](10_low_poly_modeling.md), [../animation/26_character_rigging.md](../animation/26_character_rigging.md)

---

## 11.1 TOPOLOGY PHILOSOPHY FOR RO CHARACTERS

RO characters require a balance between:
- **Minimal polygons** (performance, aesthetic)
- **Clean deformation** (animation quality at joints)
- **Readable normals** (lighting correctness)

**The Rule of Deformation:**  
Every joint that bends in animation requires at least **2 edge loops** perpendicular to the bend axis within the deformation zone, and preferably **3** for clean deformation.

---

## 11.2 EDGE FLOW PRINCIPLES

### Good Edge Flow (Humanoid Deformation)

Edge loops should flow in **rings around the body axis**, perpendicular to the direction of movement at joints:

```
ARM:
  Correct: Rings around the arm at shoulder, mid-upper, elbow, mid-forearm, wrist
  
  SHOULDER ──┬── UPPER ARM ──┬── ELBOW ──┬── FOREARM ──┬── WRIST
             │               │            │             │
           ring A          ring B        ring C       ring D
  
  At elbow: rings must be CLOSER TOGETHER (more density)
  Away from elbow: rings can be farther apart

LEG:
  HIP ──┬── THIGH ──┬── KNEE ──┬── SHIN ──┬── ANKLE
        │            │           │          │
      ring A       ring B      ring C     ring D
```

### ASCII Topology Diagram — Elbow Region

```
Good topology at elbow:
  ─────────────────────
  ───────┐   ┌─────────  ← Edge loop A (above elbow, close)
         │   │
  ───────┘   └─────────  ← Edge loop B (at elbow, closest)
  ─────────────────────
  ─────────────────────  ← Edge loop C (below elbow, close)

Bad topology at elbow:
  ─────────────────────
  
  (long gap — no loops)
  
  ─────────────────────
  
  Result: sharp crease on bend instead of smooth deformation
```

### ASCII Topology Diagram — Good vs Bad Quad Flow

```
GOOD: Quad grid flowing along body axis
  ┌───┬───┬───┐
  │   │   │   │  Columns run along limb direction
  ├───┼───┼───┤  Rows form rings around circumference
  │   │   │   │
  └───┴───┴───┘

BAD: N-gon or star pattern
  ┌───────────┐
  │     *     │  Star pattern → unpredictable normals
  │   * | *   │  Causes shading artifacts
  └───────────┘
```

---

## 11.3 N-GON ELIMINATION

N-gons (polygons with > 4 edges) cause problems with UV mapping, shading, and game engine import.

### How to Find N-Gons in Blender

```
METHOD 1: Select by Trait
  → Edit Mode → Select → Select All By Trait
  → Select: Faces by Sides
  → Type: Greater Than, 4 sides
  → This selects all N-gons

METHOD 2: Face Counts Overlay
  → View Overlay → Statistics
  → Look for triangle vs quad counts

METHOD 3: Manual inspection
  → Ctrl+Alt+Shift+M (Select Non-Manifold)
  → Problematic areas highlight
```

### N-Gon Elimination Procedure

```
STEP 1: Identify N-gon (polygon with 5+ vertices)
STEP 2: Analyze the shape
  → Is it on a flat surface? → Triangulate with Ctrl+T
  → Is it on a curved surface? → Must manually connect vertices

STEP 3: Manual fix using Knife Tool (K)
  → Cut from one vertex to another across the N-gon
  → Create a quad + smaller polygon
  → Continue cutting until all faces are quads or tris

STEP 4: Validate
  → Re-run N-gon selection → should return 0 results
```

---

## 11.4 TRIANGLE VS QUAD STRATEGY

```
USE QUADS:
  → On all curved surfaces that need clean subdivision
  → At all deformation joints
  → On UV-mapped surfaces where distortion matters

USE TRIANGLES:
  → On flat termination caps (top of cylindrical finger, foot sole)
  → At topology transitions (connecting different loop densities)
  → On surfaces that will never deform

STRATEGY FOR RO CHARACTERS:
  → 80–90% quads on body
  → 10–20% triangles on flat areas, termination points
  → Zero N-gons on final mesh
```

---

## 11.5 NORMAL DIRECTION VALIDATION

Incorrect normals cause dark patches, inside-out rendering, and incorrect lighting.

### Blender Normal Validation

```
STEP 1: Enable Face Orientation overlay
  → 3D Viewport → Overlay dropdown → Face Orientation
  → Blue = outward-facing (correct)
  → Red = inward-facing (wrong)

STEP 2: Select all red faces
  → Edit Mode → Select → Select All By Trait → Face by Normal
  → Or: visually identify red areas

STEP 3: Flip selected normals
  → Mesh → Normals → Flip

STEP 4: Recalculate normals (whole mesh)
  → Select All (A)
  → Mesh → Normals → Recalculate Outside (Shift+N)
  → This works for water-tight meshes

STEP 5: Handle inside-out geometry
  → If Recalculate doesn't work, geometry may be inverted
  → Select problematic area
  → S → -1 to mirror/invert (use with caution)
```

---

## 11.6 SMOOTH VS FLAT SHADING

### For RO Aesthetic

```
FLAT SHADING (recommended for most RO geometry):
  → Each face has one surface normal (average of its vertices)
  → Creates faceted appearance
  → Looks deliberately low-poly → correct for RO
  → Apply: Select all → Right-click → Shade Flat

SMOOTH SHADING (selective use):
  → Each vertex normal is averaged from surrounding faces
  → Creates smooth appearance
  → Use ONLY on:
    - Head/face geometry (smooth the face shape)
    - Curved accessories (orbs, round gems)
  → Apply: Select smooth faces → Right-click → Shade Smooth

MIXED APPROACH (recommended):
  1. Apply Flat Shading to entire mesh
  2. Select face areas that should appear smooth (head, face)
  3. Apply Smooth Shading to those faces only
  4. Add Edge Split modifier at 30–45° threshold:
     → Preserves sharp edges (armor plates)
     → Softens gentle curves (head, shoulder curve)
```

---

## 11.7 NON-MANIFOLD EDGE DETECTION AND REPAIR

Non-manifold geometry causes problems in UV unwrapping, physics, and export.

```
WHAT IS NON-MANIFOLD:
  → Edge shared by more than 2 faces (interior geometry)
  → Edge shared by 0 faces (floating edge)
  → Vertex shared by non-connected face groups (bowtie vertex)

DETECT IN BLENDER:
  → Edit Mode → Select → Select All By Trait → Non-Manifold
  → Or: Alt+Shift+Ctrl+M

COMMON CAUSES:
  → Extrusion without actual face creation
  → Deleted faces leaving boundary edges
  → Duplicate geometry at same position

REPAIR STEPS:
  1. After selection, examine each highlighted region
  2. Floating faces: delete them (they serve no purpose)
  3. Holes: fill with F key (creates face to close hole)
  4. Interior faces: select and delete if truly internal
  5. Bowtie vertices: merge with M or dissolve vertex
```

---

## 11.8 MESH DENSITY DISTRIBUTION

Not all body areas need equal polygon density:

```
HIGH DENSITY AREAS (more polygons):
  → Face (most expressive, most seen close)
  → Hands (complex geometry, visible during attack)
  → Shoulder/chest (class identity)
  → Elbow/knee joints (deformation)

LOW DENSITY AREAS (fewer polygons):
  → Back of head (mostly covered by hair)
  → Covered torso areas (hidden by armor)
  → Inner arm / armpit (not visible in standard poses)
  → Bottom of feet (never seen from above)
  → Covered leg areas (under armor/greaves)

RULE: "Polygons where eyes go, not where eyes never go."
```

---

## 11.9 MESH INTEGRITY VALIDATION CHECKLIST

Before proceeding to UV mapping:

```
GEOMETRY:
□ No N-gons (polygon sides > 4)
□ No non-manifold edges
□ No duplicate vertices (Merge by Distance = 0 merged)
□ No zero-area faces
□ All normals facing outward (Face Orientation = all blue)
□ No interior faces

SCALE AND TRANSFORM:
□ Object scale is (1.0, 1.0, 1.0) [Ctrl+A > Apply All Transforms]
□ Object rotation is (0, 0, 0) after apply
□ Mesh centered correctly on origin

DEFORMATION:
□ Minimum 2 edge loops at each major joint
□ Edge loops at shoulder, elbow, wrist
□ Edge loops at hip, knee, ankle
□ No poles (vertex with 5+ edges) at joints

BUDGET:
□ Triangle count within budget for target quality level
□ No unnecessary loops on flat areas
```
