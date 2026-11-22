# 33 — TROUBLESHOOTING AND DIAGNOSTICS
## Symptom → Cause → Fix Reference

> **Purpose:** Rapid diagnosis and resolution of all common ROCHAR pipeline issues.  
> **Related:** All modules — this file cross-references every system.

---

## 33.1 VISUAL PROBLEMS

### Character Floating Above Ground

```
SYMPTOM: Character hovers above terrain in scene.

CAUSE A: Billboard foot anchor miscalibrated
  → Check: billboard quad Y=0 at foot (see Module 18)
  → Fix:   geometry.translate(0, 0.5, 0) with scale factor applied

CAUSE B: Character world position Y offset non-zero
  → Check: billboard.position.y !== 0
  → Fix:   billboard.position.set(x, 0, z)

CAUSE C: Bone Root not at Y=0 in export
  → Check: In Blender, Root bone head at (0,0,0)
  → Fix:   Select Root bone → Edit Mode → move head to (0,0,0)

CAUSE D: GLB Y-up conversion added offset
  → Check: gltf.scene.position.y after loading
  → Fix:   gltf.scene.position.y = 0  (manual override after load)
```

---

### Character Appears Completely Black

```
SYMPTOM: Character renders as solid black silhouette.

CAUSE A: Lightmap UV2 not provided but uUseLightmap=true
  → Fix:   mat.uniforms.uUseLightmap.value = false
  
CAUSE B: Ambient intensity = 0
  → Fix:   mat.uniforms.uAmbientIntensity.value = 0.4

CAUSE C: Texture not loaded (placeholder null texture)
  → Check: texture.image in browser devtools Network tab
  → Fix:   Verify texture URL is accessible, no 404 error

CAUSE D: Vertex colors all zero (black)
  → Fix:   mat.uniforms.uVertexColorMix.value = 0  (disable vertex color)
  
CAUSE E: Output color space mismatch
  → Fix:   renderer.outputColorSpace = THREE.SRGBColorSpace
           texture.colorSpace = THREE.SRGBColorSpace
```

---

### Texture Appears Blurry

```
SYMPTOM: Pixel art texture looks blurry/interpolated.

CAUSE: Linear texture filtering (default in Three.js)
  → Fix:
    texture.magFilter = THREE.NearestFilter;
    texture.minFilter = THREE.NearestFilter;
    texture.generateMipmaps = false;
    texture.needsUpdate = true;

NOTE: Must set before texture is uploaded to GPU.
      If texture already loaded, set needsUpdate=true to re-upload.
```

---

### Magenta Fringe Around Sprite

```
SYMPTOM: Purple/magenta outline pixels visible around sprite edges.

CAUSE A: Chroma-key tolerance too low (not removing near-magenta pixels)
  → Fix:   Increase uChromaKeyTolerance from 0.3 to 0.5

CAUSE B: Mip-map blending mixing magenta with character colors
  → Fix:   generateMipmaps = false (disable mips for sprite atlas)
  
CAUSE C: Alpha threshold too high (keeping near-transparent magenta fringe pixels)
  → Fix:   Lower alphaTest from 0.5 to 0.3 (but watch for ghost outlines)
  
CAUSE D: Original sprite not properly converted (magenta_to_alpha missed pixels)
  → Fix:   Re-run magenta_to_alpha() with higher tolerance
           magenta_to_alpha('input.bmp', 'output.png', tolerance=40)
```

---

### UV Seam Visible on 3D Model

```
SYMPTOM: Visible line/seam on character mesh where UV islands join.

CAUSE A: Insufficient atlas padding
  → Fix:   Increase frame padding from 1px to 4px
           Rebake texture with 4px margin in Blender

CAUSE B: Linear texture filtering interpolating across UV island border
  → Fix:   NearestFilter (see blurry texture fix above)
  
CAUSE C: UV seam placed on visible polygon edge
  → Fix:   Move UV seam to hidden edge (under arm, inside collar, etc.)
  
CAUSE D: Mip-map level 1+ bleeding across island border
  → Fix:   Disable mips if seam persists
```

---

### Character Tilts When Camera Pitches

```
SYMPTOM: Billboard character tilts/rotates when camera looks up or down.

CAUSE: Using spherical billboard (camera up) instead of cylindrical (world up)
  
VERTEX SHADER FIX:
  // WRONG (spherical):
  vec3 camUp = vec3(uView[0][1], uView[1][1], uView[2][1]);  // camera up
  
  // CORRECT (cylindrical):
  vec3 worldUp = vec3(0.0, 1.0, 0.0);  // always world up

  Then use worldUp (not camUp) in billboard construction.
  See Module 22 vertex shader section 22.1.
```

---

### Z-Fighting (Flickering Surface)

```
SYMPTOM: Two surfaces at nearly same depth flicker/alternate.

CAUSE A: Shadow sprite and terrain at same Z depth
  → Fix:   shadow worldPos.y -= 0.005  (slight offset below terrain)
  
CAUSE B: Decal/overlay on exact same surface
  → Fix:   Use polygonOffsetFactor in Three.js:
           material.polygonOffset = true;
           material.polygonOffsetFactor = -1;
           material.polygonOffsetUnits = -1;

CAUSE C: Camera near plane too far (reduces depth precision)
  → Fix:   camera.near = 0.1  (default is fine, don't set to 0.01)
           Keep near/far ratio < 1000 for good depth precision
```

---

## 33.2 ANIMATION PROBLEMS

### Sprite Facing Wrong Direction

```
SYMPTOM: When character moves north, sprite shows southern-facing direction.

CAUSE A: Direction mapping inverted in movementToDirection()
  → Fix:   Negate dz: movementToDirection(dx, -dz)

CAUSE B: Camera orientation not factored in
  → Fix:   Use getSpriteFacingDirection() instead of movementToDirection()
           (passes camera to compute camera-relative direction)

CAUSE C: Atlas direction ordering different from ROCHAR convention
  → Check: metadata.directions array order
  → Fix:   Remap: if atlas uses N=0 instead of S=0, adjust index
```

---

### Animation Stutters / Irregular Speed

```
SYMPTOM: Animation plays unevenly (too fast then slow).

CAUSE A: deltaTime accumulation error
  → Check: frameTimer += deltaTime (not +=deltaTime*1000 mix)
  → Fix:   Ensure consistent units: seconds (not ms)
           THREE.Clock.getDelta() returns SECONDS
           requestAnimationFrame callback dt is MILLISECONDS
           Pick one and be consistent.

CAUSE B: Browser tab loses focus (deltaTime spikes on refocus)
  → Fix:   Cap deltaTime: const dt = Math.min(deltaTime, 0.05);

CAUSE C: Frame timer not reset on state change
  → Fix:   this.frameTimer = 0  in play() method (see Module 19)
```

---

### Dead Animation Loops

```
SYMPTOM: Character falls, then stands up and falls again (loops).

CAUSE: Loop flag set to true for dead animation.

FIX in ANIMATION_STATES:
  DEAD: {
    loop: false,
    holdLastFrame: true,  // ← This must be true
    ...
  }

FIX in _advanceFrame():
  if (animDef.holdLastFrame) {
    this.currentFrameIndex = animDef.frame_count - 1;
    return;  // Don't fire onComplete, don't reset
  }
```

---

## 33.3 EXPORT AND FORMAT PROBLEMS

### GLB Too Large (>5MB)

```
SYMPTOM: GLB file unexpectedly large.

CAUSE A: Textures too large
  → Check: gltf-validator --show-texture-info character.glb
  → Fix:   Scale textures down: 2048×2048 → 512×512 for standard chars

CAUSE B: Multiple redundant animations sampled at high FPS
  → Fix:   Export with "Optimize Animation" checked in Blender
           Reduces keyframes on static bones

CAUSE C: Draco compression not enabled (optional)
  → Fix:   Enable Draco geometry compression (may reduce mesh 50–80%)
           Note: requires GLTFLoader with DracoLoader extension

CAUSE D: Image format PNG instead of JPEG for opaque textures
  → Fix:   Use JPEG at 90% for opaque textures (much smaller)
           Keep PNG only for textures with alpha
```

---

### GLB Character Sideways or Upside-Down

```
SYMPTOM: Character loads rotated 90° or 180° in Three.js.

CAUSE: Coordinate system mismatch (Blender Z-up vs Three.js Y-up)

FIX in Blender export:
  glTF 2.0 Export → Transform → "+Y Up" = ENABLED
  
FIX post-load (if needed):
  gltf.scene.rotation.x = -Math.PI / 2;  // Rotate 90° if still wrong

VERIFY:
  After load, check: gltf.scene.rotation — should be (0, 0, 0)
  Character should stand upright with Y-axis pointing up
```

---

## 33.4 DEBUG SHADER (VISUALIZE UV/NORMALS)

```glsl
#version 330 core
// DEBUG FRAGMENT SHADER — visualize UV, normals, or vertex colors
// Swap this for the production shader to diagnose visual issues

uniform int uDebugMode;
// 0 = UV coordinates (red=U, green=V)
// 1 = World normal (RGB = XYZ)
// 2 = Vertex colors (direct display)
// 3 = Lightmap UV2 (red=U, green=V)

in vec2 vTexCoord;
in vec2 vTexCoord2;
in vec4 vVertexColor;
in vec3 vNormalWorld;

out vec4 FragColor;

void main() {
    if (uDebugMode == 0) {
        // UV1: show R=U, G=V, B=0
        FragColor = vec4(fract(vTexCoord), 0.0, 1.0);
    } else if (uDebugMode == 1) {
        // Normal: convert -1..1 to 0..1 for display
        FragColor = vec4(vNormalWorld * 0.5 + 0.5, 1.0);
    } else if (uDebugMode == 2) {
        // Vertex color direct
        FragColor = vVertexColor;
    } else if (uDebugMode == 3) {
        // UV2 (lightmap)
        FragColor = vec4(fract(vTexCoord2), 0.0, 1.0);
    }
}
```

---

## 33.5 BROWSER WEBGL DEBUG SETUP

```javascript
// Enable WebGL debug context (shows GPU errors in console)
const canvas = document.getElementById('canvas');
const gl = canvas.getContext('webgl2', { 
    antialias: false,
    preserveDrawingBuffer: true  // For screenshots
});

// Wrap gl to log all errors
function wrapGL(gl) {
    const handler = {
        get(target, prop) {
            const val = target[prop];
            if (typeof val === 'function') {
                return (...args) => {
                    const result = val.apply(target, args);
                    const err = target.getError();
                    if (err !== 0) {
                        console.error(`WebGL error after ${prop}: ${err} (0x${err.toString(16)})`);
                    }
                    return result;
                };
            }
            return val;
        }
    };
    return new Proxy(gl, handler);
}

// Three.js debug: log shader compile errors
renderer.debug.checkShaderErrors = true;  // Always on during development

// Check for WebGL capabilities:
console.log('WebGL2:', !!canvas.getContext('webgl2'));
console.log('Max texture size:', gl.getParameter(gl.MAX_TEXTURE_SIZE));
console.log('Max vertex attribs:', gl.getParameter(gl.MAX_VERTEX_ATTRIBS));
```

---

## 33.6 QUICK DIAGNOSTIC DECISION TREE

```
PROBLEM: Character looks wrong
├── Completely black? → Check ambient, vertex colors, texture loading
├── Blurry? → Set NearestFilter on all textures
├── Missing/wrong colors? → Check colorSpace (SRGBColorSpace for albedo)
├── Floating? → Check billboard Y=0 foot anchor
├── Tilting with camera? → Use worldUp not camUp in billboard shader
├── Z-fighting? → Add depth bias / polygon offset
└── Fringe pixels? → Increase alpha test threshold or chroma key tolerance

PROBLEM: Animation wrong  
├── Wrong direction? → Check movementToDirection() sign conventions
├── Stuttering? → Cap deltaTime, check unit consistency (seconds vs ms)
├── Dead loops? → Set holdLastFrame=true, loop=false
├── State won't change? → Check priority system in play()
└── Missing frames? → Verify atlas metadata frame indices

PROBLEM: Export/Format
├── Sideways? → Enable "+Y Up" in Blender GLB export
├── Too large? → Downscale textures, enable Optimize Animation
├── Missing animations? → Check NLA editor, re-export with NLA Strips ON
└── Load fails? → Run gltf-validator, check browser console for errors
```
