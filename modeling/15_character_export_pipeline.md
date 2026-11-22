# 15 — CHARACTER EXPORT PIPELINE
## GLB, glTF, FBX, and OBJ Export with Validation

> **Purpose:** Define export procedures, format-specific settings, and post-export validation.  
> **Related:** [14_blender_procedural_modeling.md](14_blender_procedural_modeling.md), [../integration/30_threejs_and_opengl_integration.md](../integration/30_threejs_and_opengl_integration.md)

---

## 15.1 PRE-EXPORT CHECKLIST

**Always complete this checklist before any export:**

```
GEOMETRY:
□ Apply all transforms (Ctrl+A > All Transforms)
  → Scale must be (1.0, 1.0, 1.0)
  → Rotation must be (0, 0, 0)
  → Location should be (0, 0, 0) for root object
□ No N-gons (> 4-sided faces)
□ No non-manifold edges
□ Merge by Distance applied (threshold: 0.001)
□ Normals recalculated (Shift+N)

MATERIALS:
□ All faces have a material assigned
□ All textures saved to disk (not just in Blender memory)
□ Texture file paths are valid
□ No packed images that failed to unpack

UV:
□ All faces have UV coordinates
□ No UV islands outside 0–1 range
□ Lightmap UV (if used) stored in UV2 channel

ARMATURE (if present):
□ Armature modifier applied to all meshes
□ Weight painting covers all vertices (no unweighted verts)
□ Pose cleared to Rest pose before export (Ctrl+A > Apply as Rest Pose)
□ All bone names follow naming convention

ANIMATION (if present):
□ Actions are named and organized in NLA editor
□ Frame range set correctly per animation
□ No redundant keyframes
```

---

## 15.2 GLB EXPORT (Recommended for Three.js)

GLB is binary glTF — single file, all assets embedded.

### Blender Export Settings

```
File → Export → glTF 2.0 (.glb/.gltf)

GENERAL:
  Format:           GLB (Binary)
  Include:          Selected Objects

TRANSFORM:
  +Y Up:            ✓ ENABLED
  (Blender is Z-up, glTF/Three.js is Y-up)

GEOMETRY:
  Apply Modifiers:  ✓ ENABLED
  UVs:              ✓ ENABLED
  Normals:          ✓ ENABLED
  Tangents:         ✓ ENABLED (if using normal maps)
  Vertex Colors:    ✓ ENABLED
  
MATERIAL:
  Materials:        Export
  Images:           JPEG or PNG
  JPEG quality:     90

ANIMATION:
  Use Current Frame: ✗ DISABLED (use full range)
  Limit to Playback Range: ✗ DISABLED
  Always Sample Animations: ✓ ENABLED
  NLA Strips:       ✓ ENABLED
  
COMPRESSION:
  Draco:            ✗ DISABLED (for maximum compatibility)
```

### Command-Line GLB Export (batch processing)

```bash
# Blender headless export
blender --background character.blend \
  --python-expr "
import bpy
bpy.ops.export_scene.gltf(
    filepath='output/character.glb',
    export_format='GLB',
    use_selection=False,
    export_yup=True,
    export_apply=True,
    export_colors=True,
    export_animations=True,
    export_nla_strips=True
)
"
```

---

## 15.3 GLTF EXPORT (Separate JSON + BIN + Textures)

Use when textures need to be managed separately:

```
Format:           GLTF Separate
Export Location:  output_folder/

Results in:
  character.gltf   (JSON scene description)
  character.bin    (Binary mesh data)
  texture_0.png    (Texture file)
  texture_1.png    (Second texture, etc.)

Three.js loading:
const loader = new THREE.GLTFLoader();
loader.load('character.gltf', (gltf) => {
    scene.add(gltf.scene);
});
```

---

## 15.4 FBX EXPORT (For Unreal/Unity/Game Engines)

FBX is widely supported but has coordinate system differences:

```
File → Export → FBX (.fbx)

SETTINGS:
  Scale:            1.0 (do NOT use Apply Scale Unit)
  Apply Scalings:   FBX All (converts scale to match FBX standard)
  Forward:          -Z Forward
  Up:               Y Up
  Apply Transform:  ✓ ENABLED
  
  Mesh:
    Smoothing:      Face (or Edge — matches Blender flat shading)
    Loose Edges:    ✗ DISABLED
    Tangent Space:  ✓ if normal maps used
  
  Armature:
    Add Leaf Bones: ✗ DISABLED (extra leaf bones confuse importers)
    Primary Bone Axis: Y Axis
    Secondary Bone Axis: X Axis
  
  Animation:
    Key All Bones:  ✓ ENABLED
    NLA Strips:     ✓ ENABLED
    Force Start/End Keying: ✓ ENABLED

COORDINATE NOTE:
  Unity: Right-click FBX → Reset Model Position may be needed
  Unreal: Character may import at 90° rotation — apply FBX Root offset
```

---

## 15.5 OBJ EXPORT (Universal Static Mesh)

OBJ for static meshes without animation:

```
File → Export → Wavefront (.obj)

SETTINGS:
  Scale:        1.0
  Forward Axis: -Z
  Up Axis:      Y
  
  Objects:      ✓ Objects as OBJ Objects
  Material Groups: ✓ Material Groups
  Smooth Groups:   ✓ (if smooth shading used)
  Normals:      ✓ ENABLED
  UVs:          ✓ ENABLED
  Vertex Colors: ✗ (OBJ format does not support vertex colors)

NOTE: OBJ exports a companion .mtl file and all textures separately.
      Always export to a dedicated folder.
```

---

## 15.6 COORDINATE SYSTEM REFERENCE

Different tools use different "up" axes:

| Tool | Up Axis | Forward Axis | Notes |
|------|---------|-------------|-------|
| Blender | Z | -Y | Default Blender world |
| glTF/GLB | Y | -Z | Web standard |
| Three.js | Y | -Z | Matches glTF |
| Unity | Y | Z | Right-handed |
| Unreal | Z | X | Left-handed |
| OpenGL | Y | -Z | Right-handed |
| DirectX | Y | Z | Left-handed |

**Always use GLB export for Three.js** — the +Y Up option handles conversion.

---

## 15.7 TEXTURE EMBEDDING STRATEGY

```
EMBEDDED (GLB):
  Advantages:  Single file, easy deployment, no path issues
  Disadvantages: Larger file, can't update textures without re-export
  Use when:    Final deployment, single character package

EXTERNAL (GLTF Separate):
  Advantages:  Textures shareable between characters, cacheable
  Disadvantages: Multiple files, path management required
  Use when:    Many characters sharing equipment textures

RECOMMENDATION:
  Development:  GLTF Separate (easier texture iteration)
  Production:   GLB (single file deployment)
```

---

## 15.8 POST-EXPORT VALIDATION

### Validation with gltf-validator (command-line)

```bash
# Install gltf-validator
npm install -g gltf-validator

# Validate exported file
gltf-validator character.glb

# Output shows:
# - Error count (should be 0)
# - Warning count (review warnings)
# - Info: triangle count, texture count, etc.
```

### Quick Three.js Visual Validation

```javascript
// Load and display in browser for quick visual check
import * as THREE from 'three';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x303030);

const camera = new THREE.PerspectiveCamera(45, window.innerWidth/window.innerHeight, 0.1, 100);
camera.position.set(0, 1, 3);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.target.set(0, 0.5, 0);  // Look at character center

// Lighting matching classic RO warm directional
const ambient = new THREE.AmbientLight(0x665A5A, 0.4);
scene.add(ambient);
const sun = new THREE.DirectionalLight(0xFFF3D9, 1.0);
sun.position.set(0.5, 1.0, 0.3);
scene.add(sun);

const loader = new GLTFLoader();
loader.load('character.glb', (gltf) => {
    scene.add(gltf.scene);
    console.log('Model loaded. Polygons:', countTriangles(gltf.scene));
}, undefined, (error) => {
    console.error('Load failed:', error);
});

function countTriangles(object) {
    let count = 0;
    object.traverse(child => {
        if (child.isMesh && child.geometry) {
            const geo = child.geometry;
            if (geo.index) count += geo.index.count / 3;
            else count += geo.attributes.position.count / 3;
        }
    });
    return Math.round(count);
}

function animate() {
    requestAnimationFrame(animate);
    controls.update();
    renderer.render(scene, camera);
}
animate();
```

---

## 15.9 EXPORT VALIDATION CHECKLIST

```
□ File loads without errors in target application
□ Mesh appears at correct scale (character height ~1m)
□ Y-axis is up (character not lying on side)
□ Textures load and display correctly
□ Vertex colors appear on mesh (if applicable)
□ Triangle count matches expected budget
□ Armature loads with correct bone hierarchy
□ All animations play correctly
□ Foot position touches ground plane (Y=0 or Z=0 depending on engine)
□ No z-fighting artifacts visible
□ Material renders with correct flat-diffuse appearance
```
