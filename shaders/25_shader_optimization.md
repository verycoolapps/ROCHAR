# 25 — SHADER OPTIMIZATION
## Performance Techniques for Real-Time RO Character Rendering

> **Purpose:** Maximize rendering performance without sacrificing visual quality.  
> **Related:** [20_glsl_character_shader.md](20_glsl_character_shader.md), [../integration/32_quality_assurance.md](../integration/32_quality_assurance.md)

---

## 25.1 DRAW CALL REDUCTION

```
PROBLEM: Each Mesh.draw() call has CPU overhead (~0.1–0.5ms per call).
         50 characters × 3 meshes each = 150 draw calls → potential frame stutter.

SOLUTION A: TEXTURE ATLASING
  → Pack all character sprites into one atlas (512×512 or 1024×1024)
  → All characters share ONE material/shader/texture bind
  → Different characters = different UV offsets in same draw call
  → Reduces: N materials → 1 material

SOLUTION B: INSTANCED RENDERING
  → Use THREE.InstancedMesh for identical character types
  → GPU draws 100 copies of same mesh in one draw call
  → Per-instance data: position, UV offset, color tint
  → Reduces: N draw calls → 1 draw call

SOLUTION C: MESH MERGING
  → Merge static world geometry into single BufferGeometry
  → Pre-merge at load time, not per-frame
  → Use THREE.BufferGeometryUtils.mergeBufferGeometries()
  
BENCHMARK GUIDELINES:
  Target: < 100 draw calls for 60fps on mobile
  Desktop: < 500 draw calls comfortable
  Each atlas saves: (N_characters - 1) draw calls
```

---

## 25.2 INSTANCED BILLBOARD RENDERING

```javascript
import * as THREE from 'three';

/**
 * Render up to MAX_INSTANCES identical characters with one draw call.
 * Each instance has its own position, UV frame, and color tint.
 */
class InstancedBillboardRenderer {
    constructor(atlasTexture, maxInstances = 200) {
        this.maxInstances = maxInstances;
        this.activeCount  = 0;
        
        // Billboard quad geometry
        const geo = this._createBillboardGeo();
        
        // Per-instance data stored in textures or buffers
        this.instancePositions = new Float32Array(maxInstances * 4); // xyz + padding
        this.instanceFrameUVs  = new Float32Array(maxInstances * 4); // offset.xy, size.xy
        this.instanceTints     = new Float32Array(maxInstances * 4); // rgba
        
        // Create buffer attributes for per-instance data
        const posAttr   = new THREE.InstancedBufferAttribute(this.instancePositions, 4);
        const frameAttr = new THREE.InstancedBufferAttribute(this.instanceFrameUVs, 4);
        const tintAttr  = new THREE.InstancedBufferAttribute(this.instanceTints, 4);
        
        posAttr.setUsage(THREE.DynamicDrawUsage);   // Updated every frame
        frameAttr.setUsage(THREE.DynamicDrawUsage);
        tintAttr.setUsage(THREE.DynamicDrawUsage);
        
        geo.setAttribute('aInstancePos',   posAttr);
        geo.setAttribute('aInstanceFrame', frameAttr);
        geo.setAttribute('aInstanceTint',  tintAttr);
        
        this.mesh = new THREE.InstancedMesh(geo, 
            this._createMaterial(atlasTexture), maxInstances);
        this.mesh.count = 0;
        this.mesh.frustumCulled = false;
    }
    
    _createBillboardGeo() {
        const geo = new THREE.BufferGeometry();
        const positions = new Float32Array([-0.5,0,0, 0.5,0,0, 0.5,1,0, -0.5,1,0]);
        const uvs = new Float32Array([0,0, 1,0, 1,1, 0,1]);
        const idx = new Uint16Array([0,1,2, 0,2,3]);
        geo.setAttribute('aLocalPos', new THREE.BufferAttribute(positions, 3));
        geo.setAttribute('aTexCoord', new THREE.BufferAttribute(uvs, 2));
        geo.setIndex(new THREE.BufferAttribute(idx, 1));
        return geo;
    }
    
    setInstance(i, worldPos, frameOffset, frameSize, tint = [1,1,1,1]) {
        const pi = i * 4, fi = i * 4, ti = i * 4;
        this.instancePositions[pi+0] = worldPos[0];
        this.instancePositions[pi+1] = worldPos[1];
        this.instancePositions[pi+2] = worldPos[2];
        this.instanceFrameUVs[fi+0] = frameOffset[0];
        this.instanceFrameUVs[fi+1] = frameOffset[1];
        this.instanceFrameUVs[fi+2] = frameSize[0];
        this.instanceFrameUVs[fi+3] = frameSize[1];
        this.instanceTints[ti+0] = tint[0];
        this.instanceTints[ti+1] = tint[1];
        this.instanceTints[ti+2] = tint[2];
        this.instanceTints[ti+3] = tint[3];
    }
    
    commit(count) {
        this.mesh.count = count;
        // Mark attributes as needing GPU upload
        this.mesh.geometry.getAttribute('aInstancePos').needsUpdate = true;
        this.mesh.geometry.getAttribute('aInstanceFrame').needsUpdate = true;
        this.mesh.geometry.getAttribute('aInstanceTint').needsUpdate = true;
    }
}
```

---

## 25.3 PRECISION QUALIFIERS (MOBILE)

```glsl
// DESKTOP: use full highp precision everywhere (default)
// MOBILE (WebGL / OpenGL ES): reducing precision saves ALU bandwidth

// In vertex shader (less precision needed):
precision mediump float;  // 16-bit instead of 32-bit

// In fragment shader:
precision mediump float;  // Position/UV math at medium precision
// exception: use highp for UV calculations to prevent seam artifacts:
highp vec2 preciseUV = vTexCoord;  // only where needed

// RULE: 
//   highp  → all position calculations, lighting math
//   mediump → color computations, UV (unless atlas seams appear)
//   lowp   → boolean flags, simple multipliers, rarely used in practice
```

---

## 25.4 AVOID BRANCHING WITH STEP/MIX

```glsl
// BAD — GPU branches are slow (causes warp divergence):
float alpha;
if (texColor.a < 0.5) {
    discard;
} else {
    alpha = 1.0;
}

// BETTER — use discard for alpha test (unavoidable, but early exit helps):
if (texColor.a < 0.5) discard;

// BAD — conditional for hit flash:
vec3 finalColor;
if (uHitFlash) {
    finalColor = vec3(1.0);
} else {
    finalColor = texColor.rgb;
}

// GOOD — branchless with mix():
float flashFactor = float(uHitFlash) * uHitFlashAlpha;  // 0 or 1 × alpha
vec3 finalColor = mix(texColor.rgb, vec3(1.0), flashFactor);

// GOOD — step() for threshold comparisons:
float isLit = step(0.5, NdotL);  // 1.0 if NdotL >= 0.5, else 0.0
```

---

## 25.5 TEXTURE SAMPLING COST

```
TEXTURE SAMPLE COST RANKING (fastest to slowest):
  1. texture() — standard sampler2D (1 lookup)
  2. textureLod() — specific mip level (1 lookup, avoids mip calc)
  3. textureOffset() — with constant UV offset (1 lookup + offset)
  4. texture() with computed UV — still 1 lookup but more math
  5. Multiple texture() calls — sum of all lookup costs

RULES:
  → Sample each texture ONCE, store in variable, reuse
  → Never sample the same texture twice in one shader
  → For atlases: compute UV offset ONCE, pass as uniform or attribute
  → Lightmap sampler: sample only when uUseLightmap=true
    (branch OK here — entire draw call shares the same value)
  
EXAMPLE: Correct single-sample pattern:
  // CORRECT:
  vec4 texSample = texture(uDiffuse, vTexCoord);
  vec3 diffuseColor = texSample.rgb;
  float alpha = texSample.a;
  
  // WRONG:
  vec3 diffuseColor = texture(uDiffuse, vTexCoord).rgb;   // sample 1
  float alpha = texture(uDiffuse, vTexCoord).a;           // sample 2 (wasted!)
```

---

## 25.6 UNIFORM BUFFER OBJECTS (UBO)

```glsl
// Without UBO: each uniform is a separate API call (slow for many uniforms)
// With UBO: batch all uniforms into one buffer update (1 API call)

// Define UBO in shader:
layout (std140) uniform RO_Scene {
    mat4  uView;
    mat4  uProjection;
    vec3  uLightDirection;
    float uLightIntensity;
    vec3  uLightColor;
    float uAmbientIntensity;
    vec3  uAmbientColor;
    float uFogStart;
    vec3  uFogColor;
    float uFogEnd;
};

// JavaScript: create and update UBO
// Three.js doesn't directly expose UBOs, but RawShaderMaterial uniforms
// are automatically batched by Three.js's WebGL backend.
// For raw WebGL, use:
// const ubo = gl.createBuffer();
// gl.bindBuffer(gl.UNIFORM_BUFFER, ubo);
// gl.bufferData(gl.UNIFORM_BUFFER, uniformData, gl.DYNAMIC_DRAW);
// gl.uniformBlockBinding(program, blockIndex, bindingPoint);
```

---

## 25.7 OPTIMIZATION PRIORITY TABLE

```
OPTIMIZATION             | EFFORT | SPEEDUP | WHEN TO APPLY
─────────────────────────┼────────┼─────────┼──────────────────────────────────
Nearest-neighbor texture  | Low    | +5%     | Always for pixel art
Single texture sample     | Low    | +2%     | Always
Atlas → 1 material        | Medium | +30–50% | When > 10 unique characters
Alpha test vs blend       | Low    | +10%    | When transparency sort is slow
Remove branching          | Medium | +3–8%   | In hot fragment shaders
Instanced rendering       | High   | +60–80% | When > 20 same-type characters
LOD (lower poly at dist.) | High   | +20–40% | When many distant characters
Low-res framebuffer       | Medium | +40%    | When targeting low-end devices
mediump precision (mobile)| Low    | +10%    | On mobile/WebGL targets
Frustum cull sprites      | Medium | +15–25% | When many off-screen characters

PRIORITY ORDER:
  1. Atlas (if not done) — biggest win per effort
  2. Alpha test for characters (avoid transparent sort overhead)
  3. Frustum culling (skip invisible objects entirely)
  4. Instancing (for multiple same-type enemies)
  5. LOD (if rendering > 50 distant characters)
```

---

## 25.8 FRAME BUDGET GUIDE

```
TARGET: 60fps = 16.67ms total frame budget

TYPICAL DISTRIBUTION:
  CPU update (animation, physics, game logic): 3–5ms
  Draw call submission (CPU → GPU):            2–4ms
  GPU rendering (vertex + fragment shaders):   5–8ms
  Present/swap:                                1–2ms
  Headroom:                                    1–3ms

GPU VERTEX SHADER BUDGET:
  World terrain (static, few draw calls):      1ms
  Character billboards (50 chars):             2ms
  3D models + rigged characters:               3ms
  Post-processing pass:                        1ms
  Total GPU:                                   7ms target

GPU FRAGMENT SHADER BUDGET:
  Pixel fill rate is usually the bottleneck.
  At 1920×1080: 2M pixels × shader cost
  At 480×270 (retro scale): 130K pixels (15× cheaper)
  → Low-res render is a massive fragment shader speedup
```
