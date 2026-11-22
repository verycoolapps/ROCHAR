# 18 — BILLBOARD RENDERING
## Camera-Facing Sprite Mathematics and Implementation

> **Purpose:** Implement mathematically correct billboard rendering with stable foot anchoring.  
> **Related:** [../shaders/22_glsl_billboard_shader.md](../shaders/22_glsl_billboard_shader.md), [../references/08_sprite_visual_identity.md](../references/08_sprite_visual_identity.md)

---

## 18.1 BILLBOARD TYPES

```
TYPE A: SPHERICAL BILLBOARD
  → Quad always faces camera exactly
  → Both horizontal AND vertical orientation follows camera
  → Problem: Character tilts when camera pitches up/down
  → Use for: Effects, particles, icons

TYPE B: CYLINDRICAL BILLBOARD (Y-locked)  ← Recommended for characters
  → Quad rotates horizontally to face camera
  → Vertical axis (Y/up) remains fixed to world up
  → Character remains upright regardless of camera pitch
  → Correct for: Characters, trees, any upright objects
  → This is what classic RO uses

TYPE C: AXIS-ALIGNED BILLBOARD
  → Quad faces along a specific world axis
  → Use for: Horizontal effects (ground splashes, shadows)
```

---

## 18.2 CAMERA-FACING MATHEMATICS

### Extracting Camera Basis Vectors from View Matrix

The view matrix (`V`) transforms world space to camera space.  
Its rows contain the camera's right, up, and forward vectors in world space.

```glsl
// View matrix column layout (column-major OpenGL):
// View = [r0 r1 r2 tx]
//        [u0 u1 u2 ty]
//        [f0 f1 f2 tz]
//        [0  0  0  1 ]

// EXTRACTING BASIS VECTORS (from view matrix):

// Camera RIGHT vector (world space):
vec3 camRight = vec3(uView[0][0], uView[1][0], uView[2][0]);
// Note: in column-major matrix, [col][row]
// uView[0][0] = row 0, col 0 = first column's x
// Because view = [col0 | col1 | col2 | col3]
// col0 = (R.x, U.x, F.x, 0) in a RIGHT-HANDED system
// SO: camRight = vec3(uView[0][0], uView[1][0], uView[2][0])

// Camera UP vector (world space):
vec3 camUp = vec3(uView[0][1], uView[1][1], uView[2][1]);

// Camera FORWARD vector (world space, points INTO screen):
vec3 camForward = vec3(uView[0][2], uView[1][2], uView[2][2]);

// WORLD UP (for cylindrical billboard):
vec3 worldUp = vec3(0.0, 1.0, 0.0);
```

**Verification:** `dot(camRight, camUp)` should be ≈ 0.0 (perpendicular).

---

## 18.3 BILLBOARD VERTEX CONSTRUCTION

The billboard is a quad with 4 vertices (or 2 triangles = 6 vertices).

```
QUAD VERTEX LAYOUT (local space before billboard transform):

  v0 (-0.5, 1.0, 0)   v1 (0.5, 1.0, 0)
         ┌────────────────┐
         │                │
         │   Sprite area  │
         │                │
         └────────────────┘
  v3 (-0.5, 0.0, 0)   v2 (0.5, 0.0, 0)

Y=0 is the FOOT ANCHOR (bottom of quad = character feet)
Y=1 is the TOP of the sprite (head or higher)

This means:
  → Character's foot position = world anchor position
  → Character stands on the ground, not floats
```

### Billboard Transform in Vertex Shader

```glsl
#version 330 core

layout (location = 0) in vec3 aLocalPos;  // local quad vertex position
layout (location = 1) in vec2 aTexCoord;

uniform mat4 uView;
uniform mat4 uProjection;
uniform vec3 uWorldPosition;  // character foot anchor in world space
uniform vec2 uScale;          // (sprite_world_width, sprite_world_height)

out vec2 vTexCoord;

void main() {
    // Extract camera right vector from view matrix
    vec3 camRight = vec3(uView[0][0], uView[1][0], uView[2][0]);
    
    // For CYLINDRICAL billboard: use world up (not camera up)
    // This keeps character upright when camera pitches
    vec3 worldUp = vec3(0.0, 1.0, 0.0);
    
    // Compute final world position of this vertex:
    // Start at foot anchor
    // Offset right by local X * width
    // Offset up by local Y * height
    vec3 worldPos = uWorldPosition 
                  + camRight  * aLocalPos.x * uScale.x
                  + worldUp   * aLocalPos.y * uScale.y;
    
    vTexCoord = aTexCoord;
    gl_Position = uProjection * uView * vec4(worldPos, 1.0);
}
```

---

## 18.4 FOOT ANCHOR CALCULATION

The foot anchor is the world-space position where the character's feet touch the ground.

```
WORLD_ANCHOR_POSITION = character position on terrain

The billboard geometry extends UPWARD from this point.
This ensures the character appears to stand on the terrain.

VERTICAL SCALE:
  sprite_world_height = character_height_in_meters
  
  For standard character: sprite_world_height = 4.0 × H = 4.0 × 0.25m = 1.0m

HORIZONTAL SCALE:
  sprite_world_width = character_width_in_meters
  
  For standard character: sprite_world_width ≈ 0.6–0.8m

SCALE CALCULATION FROM SPRITE:
  If frame is 64×96 px and character is 1.0m tall:
  world_height = 1.0m
  world_width  = 1.0 × (64/96) ≈ 0.667m
```

---

## 18.5 DEPTH TESTING AND ALPHA SORTING

Transparent billboards require correct Z-sorting:

```
RULE: Transparent objects must render BACK-TO-FRONT from camera.
      Otherwise, objects behind draw over objects in front.

STRATEGY 1: MANUAL Z-SORT (CPU)
  → Each frame, sort billboard array by distance from camera
  → Farthest from camera = rendered first
  
  JavaScript example:
  billboards.sort((a, b) => {
      const distA = cameraPos.distanceToSquared(a.position);
      const distB = cameraPos.distanceToSquared(b.position);
      return distB - distA;  // farthest first
  });

STRATEGY 2: ALPHA TEST ONLY (no sorting needed)
  → Use alpha test discard instead of alpha blend
  → Writes to depth buffer (no transparency sorting needed)
  → Downside: hard edges on sprite outline
  → For classic RO: acceptable (original sprites use 1-bit transparency)
  
STRATEGY 3: DEPTH PEELING (advanced)
  → Multi-pass rendering for correct overlap
  → Not needed for classic RO look
  → Only if high-quality semi-transparent effects required

RECOMMENDED FOR RO:
  → Use alpha test for characters (hard edges match pixel art)
  → Use alpha blend + Z-sort for effects/particles only
```

---

## 18.6 BILLBOARD SCALE COMPENSATION

As camera FOV changes, screen-space sprite size changes too.

```
MAINTAINING WORLD-SCALE CONSISTENCY:
  The billboard shader above already maintains correct world scale —
  the scale is in meters, not screen pixels.
  
  As camera zooms in (smaller FOV), sprites appear larger on screen.
  This is CORRECT behavior — same as 3D objects.
  
IF SCREEN-SPACE CONSTANT SIZE IS DESIRED (icons, UI sprites):
  Add screen-space scaling to the vertex shader:
  
  float distToCamera = length(uCameraPos - uWorldPosition);
  float screenScaleFactor = distToCamera * tan(uFOV_half);
  vec3 worldPos = uWorldPosition 
                + camRight * aLocalPos.x * uScale.x * screenScaleFactor
                + worldUp  * aLocalPos.y * uScale.y * screenScaleFactor;
  
  NOTE: This makes sprite look the same size regardless of distance.
  NOT recommended for characters — they should get smaller at distance.
```

---

## 18.7 THREE.JS IMPLEMENTATION

```javascript
import * as THREE from 'three';

/**
 * Create a RO-style character billboard.
 * Uses custom ShaderMaterial for correct foot anchoring.
 */
function createROBillboard(spriteTexture, worldWidth, worldHeight) {
    // Create a unit quad with foot anchor at Y=0
    const geometry = new THREE.PlaneGeometry(1, 1);
    // Translate so bottom of plane is at y=0
    geometry.translate(0, 0.5, 0);
    // Scale to world units
    geometry.scale(worldWidth, worldHeight, 1);
    
    const material = new THREE.ShaderMaterial({
        uniforms: {
            uSpriteTexture:  { value: spriteTexture },
            uAlphaThreshold: { value: 0.5 },
            uView:           { value: new THREE.Matrix4() },
            uWorldPosition:  { value: new THREE.Vector3() },
            uScale:          { value: new THREE.Vector2(worldWidth, worldHeight) },
            uFrameOffset:    { value: new THREE.Vector2(0, 0) },
            uFrameSize:      { value: new THREE.Vector2(1, 1) }
        },
        vertexShader: BILLBOARD_VERT,    // from shaders/22_glsl_billboard_shader.md
        fragmentShader: BILLBOARD_FRAG,
        transparent: false,
        side: THREE.DoubleSide,
        depthWrite: true,
        depthTest: true,
        alphaTest: 0.5  // discard pixels below 50% alpha
    });
    
    const mesh = new THREE.Mesh(geometry, material);
    return mesh;
}

// Update all billboard materials in render loop
function updateBillboardUniforms(camera, billboards) {
    const viewMatrix = camera.matrixWorldInverse;
    billboards.forEach(billboard => {
        const mat = billboard.material;
        mat.uniforms.uView.value.copy(viewMatrix);
        mat.uniforms.uWorldPosition.value.copy(billboard.position);
    });
}

// Three.js alternative: use THREE.Sprite (simpler, less control)
function createSimpleSprite(spriteTexture) {
    const mat = new THREE.SpriteMaterial({ 
        map: spriteTexture,
        alphaTest: 0.5
    });
    // THREE.Sprite is spherical billboard by default
    // For cylindrical (Y-locked), use custom shader above
    return new THREE.Sprite(mat);
}
```

---

## 18.8 SHADOW SPRITE IMPLEMENTATION

```glsl
// === SHADOW BILLBOARD VERTEX SHADER ===
// Shadow is a Y-axis billboard (always faces up)

void main() {
    // Shadow lies flat on the ground plane
    // No camera orientation needed — just world XZ plane

    vec3 worldPos = uWorldPosition 
                  + vec3(aLocalPos.x * uScale.x, 0.0, aLocalPos.y * uScale.y);
    
    // Push shadow slightly below terrain to avoid z-fighting
    worldPos.y -= 0.005;
    
    vTexCoord = aTexCoord;
    gl_Position = uProjection * uView * vec4(worldPos, 1.0);
}

// Shadow fragment: render with partial opacity
void main() {
    vec4 color = texture(uSpriteTexture, vTexCoord);
    // Shadow sprite is typically a soft ellipse
    color.rgb = vec3(0.0);     // Pure black
    color.a *= 0.5;            // Semi-transparent
    
    if (color.a < 0.01) discard;
    FragColor = color;
}
```
