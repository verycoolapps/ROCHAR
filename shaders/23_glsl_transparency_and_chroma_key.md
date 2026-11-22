# 23 — GLSL TRANSPARENCY AND CHROMA KEY
## Alpha Modes, Magenta Removal, and Blending Techniques

> **Purpose:** Handle all RO transparency methods correctly — alpha test, alpha blend, chroma-key, and effect modes.  
> **Related:** [../sprites/16_sprite_reconstruction.md](../sprites/16_sprite_reconstruction.md), [22_glsl_billboard_shader.md](22_glsl_billboard_shader.md)

---

## 23.1 TRANSPARENCY MODE COMPARISON

```
MODE           | GPU COST  | DEPTH WRITE | SORTING NEEDED | USE CASE
───────────────┼───────────┼─────────────┼────────────────┼──────────────────────
Alpha Test     | Very Low  | YES         | No             | Characters, trees, fences
Alpha Blend    | Medium    | NO          | YES (manual)   | Semi-transparent effects
Additive Blend | Medium    | NO          | No (order safe) | Fire, lightning, aura
Chroma Key     | Very Low  | YES         | No             | Legacy RO sprites (magenta)
Premult Alpha  | Medium    | NO          | YES            | High-quality sprites
```

---

## 23.2 ALPHA TEST FRAGMENT SHADER

```glsl
#version 330 core

uniform sampler2D uTexture;
uniform float     uAlphaThreshold;  // Default: 0.5

in vec2 vTexCoord;
out vec4 FragColor;

void main() {
    vec4 color = texture(uTexture, vTexCoord);
    
    // Hard cutoff: either fully visible or fully invisible
    // Writes to depth buffer → no Z-sort needed
    if (color.a < uAlphaThreshold) {
        discard;
    }
    
    FragColor = color;
}
```

```
ALPHA TEST PROS:
  → Fastest transparent rendering mode
  → Writes depth buffer (correct occlusion with other objects)
  → No Z-sorting required
  → Perfect for pixel art (matches 1-bit transparency expectation)

ALPHA TEST CONS:
  → Hard jagged edges at threshold boundary
  → Cannot render semi-transparent regions
  → Mip-mapping causes threshold shifts (use alphaTest: 0.5 to compensate)
  
THREE.JS SETUP:
  material.alphaTest = 0.5;
  material.transparent = false;
  material.depthWrite = true;
```

---

## 23.3 ALPHA BLEND FRAGMENT SHADER

```glsl
#version 330 core

uniform sampler2D uTexture;

in vec2 vTexCoord;
out vec4 FragColor;

void main() {
    vec4 color = texture(uTexture, vTexCoord);
    
    // No discard — allow semi-transparent pixels to blend
    // GPU uses: OutputColor = SrcColor * SrcAlpha + DstColor * (1 - SrcAlpha)
    FragColor = color;
}
```

```javascript
// Three.js alpha blend setup:
material.transparent = true;
material.depthWrite = false;      // Don't write depth (would occlude objects behind)
material.depthTest = true;        // Still test depth (occluded by solid objects)
material.blending = THREE.NormalBlending;  // SrcAlpha, OneMinusSrcAlpha

// REQUIRED: sort transparent objects back-to-front each frame
// Three.js does this automatically for Mesh objects with transparent material
// For custom render loops, sort manually by distance from camera
```

---

## 23.4 CHROMA KEY (MAGENTA REMOVAL) SHADER

```glsl
#version 330 core
// CHROMA KEY — removes magenta (#FF00FF) backgrounds from legacy RO sprites
// Use when loading sprites that use magenta transparency instead of alpha

uniform sampler2D uTexture;
uniform float     uChromaKeyTolerance;  // Distance threshold (default: 0.4)
uniform vec3      uChromaKeyColor;      // Target color (default: vec3(1,0,1) = magenta)

in vec2 vTexCoord;
out vec4 FragColor;

void main() {
    vec4 color = texture(uTexture, vTexCoord);
    
    // Distance from magenta in RGB color space
    // Using Euclidean distance: sqrt((R-1)² + (G-0)² + (B-1)²)
    // Max distance from pure magenta = sqrt(3) ≈ 1.732
    float dist = distance(color.rgb, uChromaKeyColor);
    
    // If pixel is close to magenta → transparent
    if (dist < uChromaKeyTolerance) {
        discard;
    }
    
    // Optionally: soften edges near chroma boundary
    // float alpha = smoothstep(uChromaKeyTolerance * 0.8, uChromaKeyTolerance, dist);
    // FragColor = vec4(color.rgb, alpha);
    
    // Hard edge version (matches classic RO behavior):
    FragColor = color;
}
```

```
CHROMA KEY TOLERANCE GUIDE:
  Too low (< 0.2): misses magenta variants (purple fringe remains)
  Optimal (0.3–0.5): removes magenta cleanly
  Too high (> 0.6): removes skin tones and other colors near magenta
  
SAFE THRESHOLD: 0.4 for most RO sprites
ADJUST DOWN to 0.3 if characters have purple/pink clothing
ADJUST UP to 0.5 if magenta fringe persists at edges
```

---

## 23.5 PREMULTIPLIED ALPHA SHADER

```glsl
#version 330 core
// Premultiplied alpha: RGB channels already multiplied by alpha
// Higher quality than straight alpha — no "dark outline" fringing

uniform sampler2D uTexture;  // Stored as premultiplied (R=r*a, G=g*a, B=b*a, A=a)

in vec2 vTexCoord;
out vec4 FragColor;

void main() {
    vec4 color = texture(uTexture, vTexCoord);
    
    // No alpha multiply needed — already premultiplied
    // The blend equation used by GPU must be:
    // OutputColor = SrcColor * 1 + DstColor * (1 - SrcAlpha)
    // (Note: SrcFactor = ONE, not SRC_ALPHA)
    
    FragColor = color;
}
```

```javascript
// Three.js premultiplied alpha blend:
material.transparent = true;
material.blending = THREE.CustomBlending;
material.blendSrc   = THREE.OneFactor;         // Not SrcAlphaFactor
material.blendDst   = THREE.OneMinusSrcAlphaFactor;
material.blendEquation = THREE.AddEquation;

// Convert standard PNG to premultiplied:
// R *= A/255, G *= A/255, B *= A/255 for each pixel
// Python: 
// import numpy as np
// from PIL import Image
// img = Image.open('sprite.png').convert('RGBA')
// data = np.array(img, dtype=float)
// alpha = data[:,:,3:4] / 255.0
// data[:,:,:3] *= alpha
// Image.fromarray(data.astype(np.uint8), 'RGBA').save('premult.png')
```

---

## 23.6 ADDITIVE BLEND (EFFECTS)

```glsl
#version 330 core
// Additive blend: adds color to whatever is behind
// Perfect for fire, lightning, aura, holy light, magic effects

uniform sampler2D uTexture;
uniform float     uIntensity;  // Effect intensity multiplier

in vec2 vTexCoord;
out vec4 FragColor;

void main() {
    vec4 color = texture(uTexture, vTexCoord);
    
    // Alpha controls how much of the additive color is applied
    // Output RGB is multiplied by alpha before addition
    vec3 addColor = color.rgb * color.a * uIntensity;
    
    // Alpha=0 in output tells GPU: don't modify destination alpha
    FragColor = vec4(addColor, 0.0);
}
```

```javascript
// Three.js additive blend:
material.transparent = true;
material.blending = THREE.AdditiveBlending;
material.depthWrite = false;  // Don't occlude anything behind

// Additive blend equation:
// OutputColor = SrcColor * SrcAlpha + DstColor * 1
// Objects drawn in front of this effect stay visible
// Order-independent: no Z-sorting needed!
```

---

## 23.7 EFFECT TYPE → BLEND MODE REFERENCE TABLE

```
EFFECT TYPE          | BLEND MODE    | DEPTH WRITE | NOTES
─────────────────────┼───────────────┼─────────────┼──────────────────────
Character sprite     | Alpha Test    | YES         | Use threshold=0.5
Shadow sprite        | Alpha Blend   | NO          | Sort after characters
Hit spark            | Additive      | NO          | No sorting needed
Holy/Heal aura       | Additive      | NO          | No sorting needed
Buff status ring     | Additive      | NO          | No sorting needed
Fire/Explosion       | Additive      | NO          | No sorting needed
Poison cloud         | Alpha Blend   | NO          | Sort needed
Water ripple         | Alpha Blend   | NO          | Sort needed
Glass/Crystal        | Alpha Blend   | NO          | Sort needed
Legacy magenta sprite| Chroma Key    | YES         | No sorting needed
```

---

## 23.8 RENDER ORDER STRATEGY

```javascript
// Set render order for correct layering in Three.js
// Lower renderOrder = rendered first (farther back)

shadowMesh.renderOrder       = 1;  // Ground shadows first
terrainMesh.renderOrder      = 2;  // Terrain
characterMeshes.renderOrder  = 3;  // Characters (alpha test, writes depth)
effectMeshes.renderOrder     = 4;  // Additive effects (no depth write)
uiOverlays.renderOrder       = 10; // UI always on top

// For alpha-blend objects (need explicit sort):
renderer.sortObjects = true;  // Three.js sorts transparent objects by Z

// Manual sort if needed:
function sortByDistance(objects, camera) {
    objects.sort((a, b) => {
        const dA = camera.position.distanceToSquared(a.position);
        const dB = camera.position.distanceToSquared(b.position);
        return dB - dA;  // farthest first
    });
}
```

---

## 23.9 EDGE SOFTENING FOR SPRITE OUTLINES

```glsl
#version 330 core
// Soft alpha test — slightly smoother edges than hard discard
// Trade-off: may cause minor sorting artifacts near character edges

uniform sampler2D uTexture;
uniform float     uAlphaEdgeLow;   // Start of soft zone (default: 0.3)
uniform float     uAlphaEdgeHigh;  // End of soft zone (default: 0.7)

in vec2 vTexCoord;
out vec4 FragColor;

void main() {
    vec4 color = texture(uTexture, vTexCoord);
    
    // Hard discard below low threshold (fully transparent)
    if (color.a < uAlphaEdgeLow) discard;
    
    // Smooth step in the soft zone
    float alpha = smoothstep(uAlphaEdgeLow, uAlphaEdgeHigh, color.a);
    
    FragColor = vec4(color.rgb, alpha);
}
```

```
NOTE: Soft edges require material.transparent = true for blending.
This introduces Z-sort requirements for characters near each other.
Recommendation: use hard alpha test for characters, soft only for UI/effects.
```
