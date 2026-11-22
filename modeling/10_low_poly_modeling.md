# 10 — LOW-POLY MODELING
## Polygon Budget, Box Modeling Workflow, and Silhouette-First Methodology

> **Purpose:** Define practical low-poly modeling procedures, polygon budgets, and RO-specific geometry construction.  
> **Related:** [11_topology_and_mesh_engineering.md](11_topology_and_mesh_engineering.md), [../references/03_ro_character_anatomy.md](../references/03_ro_character_anatomy.md)

---

## 10.1 POLYGON BUDGET TABLES

### Player Character Body (Nude Base Mesh)

| Quality Level | Triangle Count | Use Case |
|--------------|---------------|---------|
| Distance LOD | 150–300 tris | > 50m from camera |
| Standard | 500–800 tris | Normal play distance |
| High detail | 900–1,500 tris | Close inspection |
| Cinematic | 2,000–4,000 tris | Rendered cutscene |

**Recommended for real-time:** Standard (500–800 tris)

### Complete Character (Body + Hair + Base Equipment)

| Element | Standard Budget |
|---------|----------------|
| Body mesh | 500–800 tris |
| Hair | 100–200 tris |
| Facial detail | 60–120 tris (included in body) |
| Base outfit/armor | 200–400 tris |
| Weapon | 80–200 tris |
| Shield | 60–120 tris |
| Footgear | 80–150 tris |
| Headgear | 60–200 tris |
| **Total (standard)** | **~1,200–2,100 tris** |

### NPC and Monster Budgets

| Type | Triangle Count |
|------|---------------|
| Background NPC | 200–400 tris |
| Quest NPC | 400–800 tris |
| Small monster | 200–600 tris |
| Standard monster | 600–1,200 tris |
| Boss monster | 1,500–4,000 tris |

---

## 10.2 BOX MODELING STEP-BY-STEP

Box modeling starts with simple primitive shapes and progressively refines them.

### STAGE 1: Initial Block-In (5 minutes)

```
STEP 1: Create the torso box
  → Add Cube
  → Scale: X=1.5H, Y=0.6H, Z=1.1H  (H = head unit = 0.25m)
  → Position: Center at chest level

STEP 2: Create the head sphere
  → Add UV Sphere: 8 segments, 6 rings
  → Scale to 0.85H × 1.0H × 0.65H (width × height × depth)
  → Position: Above torso, neck connection visible

STEP 3: Create arm cylinders
  → Add Cylinder: 6 vertices
  → Duplicate for upper arm and forearm
  → Scale to diameter 0.17H, length 0.65H (upper arm)
  → Rotate and position at shoulder

STEP 4: Create leg cylinders
  → Add Cylinder: 6 vertices
  → Scale to diameter 0.22H, length 0.70H (thigh)
  → Duplicate for shin

STEP 5: Foot boxes
  → Add Cube
  → Scale: X=0.15H, Y=0.40H, Z=0.12H
  → Position at end of shin
```

### STAGE 2: Shape Refinement (20–30 minutes)

```
STEP 6: Head shaping
  → Enter Edit Mode on head mesh
  → Select lower 30% vertices → scale in X/Z, pull forward in Y (jaw)
  → Select upper 60% → smooth dome shape
  → Select front face vertices → flatten slightly toward camera

STEP 7: Torso shaping
  → Loop cut at waist level
  → Scale waist ring inward (narrower than chest and hips)
  → Loop cut at shoulder level
  → Extrude shoulder caps outward for shoulder geometry

STEP 8: Merge geometry
  → Join all body pieces into one mesh (Ctrl+J)
  → Merge at neck: select head bottom ring and torso top ring
  → Bridge Edge Loops (Edge > Bridge Edge Loops)
  → Merge at shoulder: similarly bridge torso arm socket to upper arm top
  → Continue for all body connections

STEP 9: Hip connection
  → Bridge leg tops to torso bottom
  → Slight crotch indent between legs

STEP 10: Hand and foot connection
  → Bridge forearm wrist to hand box
  → Bridge shin ankle to foot box
```

### STAGE 3: Silhouette Verification (5 minutes)

```
STEP 11: Check from all angles
  → Front view (numpad 1): verify proportion grid
  → Side view (numpad 3): verify depth, posture
  → Top view (numpad 7): verify shoulder/hip width
  → Perspective view: rotate to check overall silhouette

STEP 12: Silhouette overlay
  → Set viewport to Solid, enable Face Orientation
  → Check all normals face outward (blue)
  → Check silhouette matches proportion blueprint
```

---

## 10.3 SILHOUETTE-FIRST DESIGN PRINCIPLES

The silhouette is the foundation of RO character recognition.

```
PRINCIPLE 1: DESIGN TOP-TO-BOTTOM
  Build from the most silhouette-defining element first:
  → Hat/headgear first (most recognizable from above)
  → Head second
  → Torso/armor third
  → Weapons fourth
  → Legs/feet last (most covered by armor/robes)

PRINCIPLE 2: CHECK FROM 10 METERS
  In the viewport, zoom out until character is ~100px tall.
  If you can't identify class from silhouette → redesign.

PRINCIPLE 3: EXAGGERATE CLASS MARKERS
  → Knight shoulders: make them 20% wider than "realistic"
  → Wizard hat: make it 20% taller than "real"
  → Thief hat: make tilt more extreme
  → This exaggeration is what makes RO look like RO

PRINCIPLE 4: WEAPON IS PART OF SILHOUETTE
  → Always test silhouette WITH weapon equipped
  → Character silhouette without weapon may be incomplete class read
```

---

## 10.4 POLYGON REDUCTION TECHNIQUES

When a mesh exceeds budget:

```
TECHNIQUE 1: MERGE FAR VERTICES
  → Select vertices within 0.01 world units → merge
  → Useful for mesh import cleanup

TECHNIQUE 2: DISSOLVE INTERIOR LOOPS
  → Select edge loop on flat area
  → Dissolve edges (X > Dissolve Edges)
  → Removes loop without changing silhouette

TECHNIQUE 3: DECIMATE MODIFIER
  → Apply with ratio 0.8 to start
  → Check: does silhouette still read? Yes → continue reducing
  → Stop when silhouette begins to degrade
  → Apply modifier and clean up remaining issues

TECHNIQUE 4: BACK-FACE REMOVAL
  → Faces permanently hidden by armor → delete them
  → Recessed areas that never face camera → simplify

TECHNIQUE 5: LOSSLESS WELD
  → Find T-junctions or redundant vertices
  → Merge by distance (M > Merge by Distance, 0.001 threshold)
```

---

## 10.5 BLENDER-SPECIFIC TOOLS AND SETTINGS

### Essential Tools for Low-Poly Work

```
TOOLS:
  Knife Tool (K):         Add vertices anywhere on a face
  Loop Cut (Ctrl+R):      Add circular edge loops
  Bridge Edge Loops:      Connect two parallel edge loops
  Merge by Distance (M):  Remove duplicate vertices
  Dissolve Edges (X):     Remove edge without leaving hole
  Extrude (E):            Extend geometry

VIEWPORT SETTINGS:
  Overlay > Statistics:   Shows triangle count in real-time
  Overlay > Face Orientation: Shows normal direction (blue=correct)
  Shading > Matcap:       Shows form without texture distraction

HOTKEYS:
  Alt+Z:  X-ray mode (see through for internal vertex work)
  G+G:    Slide vertex along edge
  P:      Separate selection to new object
  J:      Join vertices with an edge (face fill)
```

### Statistics Display Setup (Blender 3.x+)

```
1. In 3D Viewport: Overlay dropdown
2. Enable "Statistics"
3. In top-left corner, look for "Tris: XXXX"
4. While in Edit Mode, selected geometry shows separate count
```

---

## 10.6 LOD (LEVEL OF DETAIL) STRATEGY

```
LOD0 (Full detail, < 5m):   Full triangle budget
LOD1 (Medium, 5–20m):       50% reduction — dissolve fine detail loops
LOD2 (Distance, 20–50m):    25% of LOD0 — only silhouette-essential loops
LOD3 (Very far, > 50m):     Use billboard sprite instead of 3D mesh

AUTOMATED LOD GENERATION:
  Blender: Decimate Modifier at 0.5 for LOD1, 0.25 for LOD2
  → Always manually review results
  → Automated decimation often breaks UV mapping — rebake if needed

SWITCHING STRATEGY:
  Three.js: LOD class with distance thresholds
  OpenGL: Render multiple VBOs, select by camera distance
```

---

## 10.7 COMMON LOW-POLY MISTAKES AND FIXES

| Mistake | Detection | Fix |
|---------|-----------|-----|
| N-gons on curved surface | Select All > Select N-gons | Knife to triangulate |
| Floating internal geometry | X-ray mode inspection | Delete internal faces |
| Non-uniform scale | Object Properties > Scale ≠ 1 | Ctrl+A > Apply Scale |
| Wrong face orientation | Face Orientation overlay | Select + Alt+N > Flip |
| Excessive edge loops on flat areas | Manual inspection | Dissolve unnecessary loops |
| Vertices not merged at seams | Merge by Distance result | Merge closer vertices |
| Body part not centered on axis | Snap cursor to geometry | Recenter origin |

---

## 10.8 SUBDIVISION SURFACE VS MANUAL EDGE LOOPS

```
USE SUBDIVISION SURFACE WHEN:
  → Creating a base mesh to then retopologize
  → Generating smooth reference silhouette
  → Character will be baked from high-poly to low-poly

DO NOT USE SUBDIVISION SURFACE WHEN:
  → Final mesh IS the game asset
  → You need precise polygon count control
  → You are building a low-poly mesh intended to look low-poly (RO aesthetic)

RO AESTHETIC NOTE:
  Classic RO characters look intentionally low-poly.
  Subdivision Surface destroys this aesthetic.
  When in doubt: use manual edge loops and embrace flat polygon facets.
  The angular quality IS the aesthetic.
```
