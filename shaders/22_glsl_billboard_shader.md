# 22 — GLSL BILLBOARD SHADER
## Complete Cylindrical Billboard with Frame-Based UV Animation

> **Purpose:** Production billboard shader matching classic RO sprite rendering.  
> **Related:** [../sprites/18_billboard_rendering.md](../sprites/18_billboard_rendering.md), [../sprites/19_sprite_animation_system.md](../sprites/19_sprite_animation_system.md)

---

## 22.1 BILLBOARD VERTEX SHADER (CYLINDRICAL, FOOT-ANCHORED)

```glsl
#version 330 core

// ─── ATTRIBUTES ───────────────────────────────────────────────────────────────
// The quad is in LOCAL space with foot anchor at Y=0:
//   v0=(-0.5, 1.0, 0)  v1=(0.5, 1.0, 0)
//   v3=(-0.5, 0.0, 0)  v2=(0.5, 0.0, 0)
layout (location = 0) in vec3 aLocalPos;   // Local quad vertex (-0.5–0.5, 0–1, 0)
layout (location = 1) in vec2 aTexCoord;   // Full-quad UV (0–1 range)

// ─── UNIFORMS ─────────────────────────────────────────────────────────────────
uniform mat4  uView;           // View matrix (camera space)
uniform mat4  uProjection;     // Projection matrix
uniform vec3  uWorldPosition;  // Character's foot anchor in world space (Y=ground)
uniform vec2  uScale;          // (sprite_world_width, sprite_world_height) in meters

// Frame-based UV animation
// The full atlas is in [0,1] × [0,1]; we show a sub-rectangle
uniform vec2  uFrameOffset;    // UV of top-left corner of current frame
uniform vec2  uFrameSize;      // UV size of one frame (width, height)

// Shadow offset (push billboard slightly away from occluded objects)
uniform float uDepthBias;      // Depth bias in clip space (default: 0.0)

// ─── OUTPUTS ──────────────────────────────────────────────────────────────────
out vec2 vFrameUV;      // UV within the current animation frame

void main() {
    // ── EXTRACT CAMERA BASIS FROM VIEW MATRIX ─────────────────────────────────
    // View matrix stores camera basis in its rows (column-major storage)
    // camRight:   row 0 of view matrix = view[0][0], view[1][0], view[2][0]
    // camUp_view: row 1 of view matrix = view[0][1], view[1][1], view[2][1]
    
    vec3 camRight = vec3(uView[0][0], uView[1][0], uView[2][0]);
    
    // CYLINDRICAL billboard: use world up instead of camera up
    // This keeps the character upright when camera pitches up/down
    // If we used camUp, the character would tilt with camera pitch
    vec3 worldUp = vec3(0.0, 1.0, 0.0);
    
    // ── CONSTRUCT BILLBOARD VERTEX IN WORLD SPACE ─────────────────────────────
    // aLocalPos.x: horizontal (-0.5 to 0.5) → scale by width, orient by camRight
    // aLocalPos.y: vertical   (0.0 to 1.0)  → scale by height, orient by worldUp
    // Foot anchor is uWorldPosition (Y = ground level)
    vec3 worldPos = uWorldPosition
                  + camRight * aLocalPos.x * uScale.x
                  + worldUp  * aLocalPos.y * uScale.y;
    
    // ── FRAME UV CALCULATION ──────────────────────────────────────────────────
    // aTexCoord is 0–1 within the quad (bottom-left origin)
    // Map to the sub-rectangle of the current animation frame in the atlas
    // uFrameOffset = top-left UV of the frame in the atlas
    // uFrameSize   = width/height of one frame in UV space
    //
    // Note: texture V-axis may need flipping depending on atlas origin convention
    // If atlas is stored top-to-bottom: v = uFrameOffset.y + aTexCoord.y * uFrameSize.y
    // If atlas is stored bottom-to-top: v = uFrameOffset.y + (1.0 - aTexCoord.y) * uFrameSize.y
    vFrameUV = uFrameOffset + aTexCoord * uFrameSize;
    
    // ── FINAL CLIP POSITION ───────────────────────────────────────────────────
    vec4 clipPos = uProjection * uView * vec4(worldPos, 1.0);
    
    // Apply depth bias (small constant offset to avoid Z-fighting with floor)
    clipPos.z -= uDepthBias * clipPos.w;
    
    gl_Position = clipPos;
}
```

---

## 22.2 BILLBOARD FRAGMENT SHADER

```glsl
#version 330 core

// ─── UNIFORMS ─────────────────────────────────────────────────────────────────
uniform sampler2D uAtlasTexture;    // Full sprite atlas texture
uniform float     uAlphaThreshold;  // Alpha cutoff for discard (default: 0.5)

// Color modulation (for damage flash, status effects, etc.)
uniform vec4  uColorTint;           // Multiply color (default: vec4(1,1,1,1))
uniform float uBrightnessBoost;     // Additive brightness (default: 0.0)

// Hit flash effect
uniform bool  uHitFlash;            // When true, flash white
uniform float uHitFlashAlpha;       // Flash intensity 0–1

// Shadow (reduce brightness for characters in shadow zones)
uniform float uShadowFactor;        // 0=fully shadowed, 1=fully lit

// ─── INPUTS ───────────────────────────────────────────────────────────────────
in vec2 vFrameUV;

// ─── OUTPUT ───────────────────────────────────────────────────────────────────
out vec4 FragColor;

void main() {
    // 1. SAMPLE ATLAS AT FRAME UV
    vec4 texColor = texture(uAtlasTexture, vFrameUV);
    
    // 2. ALPHA TEST — discard transparent pixels
    // Classic RO sprites have hard 1-bit transparency
    if (texColor.a < uAlphaThreshold) {
        discard;
    }
    
    // 3. BASE COLOR × TINT
    vec3 finalRGB = texColor.rgb * uColorTint.rgb;
    float finalA  = texColor.a   * uColorTint.a;
    
    // 4. SHADOW FACTOR (darkness in shadowed areas)
    finalRGB *= uShadowFactor;
    
    // 5. BRIGHTNESS BOOST (backlit characters, aura effects)
    finalRGB += uBrightnessBoost;
    
    // 6. HIT FLASH (white flash when taking damage)
    // Blend toward white at uHitFlashAlpha intensity
    if (uHitFlash) {
        finalRGB = mix(finalRGB, vec3(1.0), uHitFlashAlpha);
    }
    
    // 7. OUTPUT
    FragColor = vec4(finalRGB, finalA);
}
```

---

## 22.3 THREE.JS BILLBOARD QUAD GEOMETRY

```javascript
import * as THREE from 'three';

/**
 * Create the quad geometry for a billboard.
 * Foot anchor at Y=0 (vertex Y ranges 0–1 before scaling).
 */
function createBillboardGeometry() {
    const geometry = new THREE.BufferGeometry();
    
    // 4 vertices: foot at Y=0, top at Y=1
    // aLocalPos positions (-0.5 to 0.5 horizontal, 0 to 1 vertical)
    const positions = new Float32Array([
        -0.5, 0.0, 0.0,  // bottom-left (foot left)
         0.5, 0.0, 0.0,  // bottom-right (foot right)
         0.5, 1.0, 0.0,  // top-right (head right)
        -0.5, 1.0, 0.0   // top-left (head left)
    ]);
    
    // UV: standard 0–1 for full quad
    // These get remapped in the vertex shader using uFrameOffset/uFrameSize
    const uvs = new Float32Array([
        0.0, 0.0,  // bottom-left
        1.0, 0.0,  // bottom-right
        1.0, 1.0,  // top-right
        0.0, 1.0   // top-left
    ]);
    
    // Two triangles (CCW winding, standard)
    const indices = new Uint16Array([0, 1, 2, 0, 2, 3]);
    
    geometry.setAttribute('aLocalPos', new THREE.BufferAttribute(positions, 3));
    geometry.setAttribute('aTexCoord', new THREE.BufferAttribute(uvs, 2));
    geometry.setIndex(new THREE.BufferAttribute(indices, 1));
    
    return geometry;
}

/**
 * Create a fully configured RO billboard mesh.
 */
function createROBillboard(atlasTexture, worldWidth, worldHeight) {
    const geometry = createBillboardGeometry();
    
    const material = new THREE.RawShaderMaterial({
        uniforms: {
            uView:            { value: new THREE.Matrix4() },
            uProjection:      { value: new THREE.Matrix4() },
            uWorldPosition:   { value: new THREE.Vector3() },
            uScale:           { value: new THREE.Vector2(worldWidth, worldHeight) },
            uAtlasTexture:    { value: atlasTexture },
            uFrameOffset:     { value: new THREE.Vector2(0, 0) },
            uFrameSize:       { value: new THREE.Vector2(1, 1) },
            uAlphaThreshold:  { value: 0.5 },
            uColorTint:       { value: new THREE.Vector4(1, 1, 1, 1) },
            uBrightnessBoost: { value: 0.0 },
            uHitFlash:        { value: false },
            uHitFlashAlpha:   { value: 0.0 },
            uShadowFactor:    { value: 1.0 },
            uDepthBias:       { value: 0.0001 }
        },
        vertexShader:   BILLBOARD_VERT,  // paste GLSL source string
        fragmentShader: BILLBOARD_FRAG,
        transparent:    false,
        depthWrite:     true,
        depthTest:      true,
        side:           THREE.DoubleSide,
        glslVersion:    THREE.GLSL3
    });
    
    const mesh = new THREE.Mesh(geometry, material);
    mesh.frustumCulled = false; // Billboard is repositioned every frame, skip culling
    return mesh;
}
```

---

## 22.4 PER-FRAME UPDATE

```javascript
/**
 * Update billboard shader uniforms each frame.
 * Call inside requestAnimationFrame loop.
 */
function updateBillboard(billboard, animController, camera, metadata) {
    const mat = billboard.material;
    
    // Update camera matrices
    mat.uniforms.uView.value.copy(camera.matrixWorldInverse);
    mat.uniforms.uProjection.value.copy(camera.projectionMatrix);
    
    // Update foot anchor position (follows parent object)
    mat.uniforms.uWorldPosition.value.copy(billboard.parent?.position ?? billboard.position);
    
    // Get current animation frame
    const frameIndex = animController.getCurrentSpriteFrame();
    
    // Convert frame index to UV coordinates from atlas metadata
    const atlasW = metadata.atlas_width;
    const atlasH = metadata.atlas_height;
    const frameW = metadata.frame_width;
    const frameH = metadata.frame_height;
    const cols   = Math.floor(atlasW / frameW);
    
    const col = frameIndex % cols;
    const row = Math.floor(frameIndex / cols);
    
    const pad_u = 1.0 / atlasW;  // 1px padding in UV space
    const pad_v = 1.0 / atlasH;
    
    mat.uniforms.uFrameOffset.value.set(
        (col * frameW) / atlasW + pad_u,
        (row * frameH) / atlasH + pad_v
    );
    mat.uniforms.uFrameSize.value.set(
        (frameW / atlasW) - 2 * pad_u,
        (frameH / atlasH) - 2 * pad_v
    );
}
```

---

## 22.5 SHADOW SPRITE SHADER VARIANT

```glsl
// SHADOW VERTEX SHADER — flat ellipse on ground plane
#version 330 core

layout (location = 0) in vec3 aLocalPos;
layout (location = 1) in vec2 aTexCoord;

uniform mat4  uView;
uniform mat4  uProjection;
uniform vec3  uWorldPosition;  // Character foot anchor
uniform vec2  uShadowScale;    // (width, depth) of shadow ellipse

out vec2 vTexCoord;

void main() {
    // Shadow lies flat on ground (XZ plane)
    // Local X → world X, local Y → world Z
    vec3 worldPos = uWorldPosition + vec3(
        aLocalPos.x * uShadowScale.x,
        -0.003,                       // Tiny Y-offset prevents Z-fight
        aLocalPos.y * uShadowScale.y  // Using Y of local space for Z in world
    );
    
    vTexCoord = aTexCoord;
    gl_Position = uProjection * uView * vec4(worldPos, 1.0);
}

// SHADOW FRAGMENT SHADER
#version 330 core

uniform sampler2D uShadowTexture;  // Soft ellipse texture (white gradient on black)
uniform float     uShadowOpacity;  // 0.0–1.0 (default: 0.4)

in vec2 vTexCoord;
out vec4 FragColor;

void main() {
    float alpha = texture(uShadowTexture, vTexCoord).r;
    alpha *= uShadowOpacity;
    if (alpha < 0.01) discard;
    // Pure black shadow with radial fade
    FragColor = vec4(0.0, 0.0, 0.0, alpha);
}
```

---

## 22.6 HORIZONTAL SPRITE FLIP (MIRRORED DIRECTIONS)

```glsl
// In billboard vertex shader, to mirror the sprite horizontally:
// Add this uniform:
uniform bool uFlipHorizontal;

// In UV calculation, flip X component if mirroring:
vec2 quadUV = aTexCoord;
if (uFlipHorizontal) {
    quadUV.x = 1.0 - quadUV.x;
}
vFrameUV = uFrameOffset + quadUV * uFrameSize;
```

```javascript
// JavaScript: set flip based on sprite direction
function setDirectionUniforms(material, direction, metadata) {
    const mirroredDirs = metadata.mirrored_directions ?? {};
    
    // Check if this direction is a mirror of another
    const isMirrored = direction in mirroredDirs;
    const sourceDir  = isMirrored ? mirroredDirs[direction] : direction;
    
    material.uniforms.uFlipHorizontal.value = isMirrored;
    
    return sourceDir;  // Use source direction to look up frame indices
}
```
