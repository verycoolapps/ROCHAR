# 20 — GLSL CHARACTER SHADER
## Complete Vertex + Fragment Shader for Ragnarok Online Characters

> **Purpose:** Production-ready GLSL 330 core shader implementing classic RO flat-diffuse rendering.  
> **Related:** [../references/07_color_and_material_language.md](../references/07_color_and_material_language.md), [25_shader_optimization.md](25_shader_optimization.md)

---

## 20.1 RO LIGHTING MODEL

Classic Ragnarok Online uses flat diffuse lighting with baked vertex colors.  
No specular highlights on most characters. Ambient is warm-tinted.

```
FinalColor = TextureColor × VertexColor × (Ambient + Diffuse)

Where:
  TextureColor = sampled from diffuse texture (RGBA)
  VertexColor  = per-vertex baked light/shadow data
  Ambient      = scene ambient color (warm, ~0.4–0.6 intensity)
  Diffuse      = directional light contribution (NdotL, clamped)
```

---

## 20.2 VERTEX SHADER

```glsl
#version 330 core

// ─── ATTRIBUTES ───────────────────────────────────────────────────────────────
layout (location = 0) in vec3  aPosition;    // Local vertex position
layout (location = 1) in vec3  aNormal;      // Vertex normal (object space)
layout (location = 2) in vec2  aTexCoord;    // UV channel 1 (diffuse)
layout (location = 3) in vec4  aVertexColor; // Per-vertex baked lighting color
layout (location = 4) in vec2  aTexCoord2;   // UV channel 2 (lightmap, optional)

// ─── UNIFORMS ─────────────────────────────────────────────────────────────────
uniform mat4 uModel;      // Object → World transform
uniform mat4 uView;       // World → Camera transform
uniform mat4 uProjection; // Camera → Clip space
uniform mat3 uNormalMatrix; // Inverse-transpose of uModel for correct normal transform

// ─── OUTPUTS TO FRAGMENT SHADER ───────────────────────────────────────────────
out vec2  vTexCoord;
out vec2  vTexCoord2;
out vec4  vVertexColor;
out vec3  vNormalWorld;    // Normal in world space (for lighting)
out vec3  vFragPosWorld;   // Fragment position in world space

void main() {
    // Transform position to world space
    vec4 worldPos = uModel * vec4(aPosition, 1.0);
    vFragPosWorld = worldPos.xyz;
    
    // Transform normal to world space using normal matrix
    // Normal matrix = inverse transpose of model matrix
    // Needed to handle non-uniform scale correctly
    vNormalWorld = normalize(uNormalMatrix * aNormal);
    
    // Pass through UV and vertex color
    vTexCoord   = aTexCoord;
    vTexCoord2  = aTexCoord2;
    vVertexColor = aVertexColor;
    
    // Final clip-space position
    gl_Position = uProjection * uView * worldPos;
}
```

---

## 20.3 FRAGMENT SHADER

```glsl
#version 330 core

// ─── UNIFORMS ─────────────────────────────────────────────────────────────────
// Textures
uniform sampler2D uDiffuseTexture;   // Main color texture (RGBA)
uniform sampler2D uLightmapTexture;  // Baked lightmap (optional, UV2)
uniform bool      uUseLightmap;      // Whether lightmap is active

// Light parameters (classic RO warm directional)
uniform vec3  uLightDirection;    // Normalized world-space light direction (toward light)
uniform vec3  uLightColor;        // Light color (default: #FFF3D9 warm white)
uniform float uLightIntensity;    // Light intensity multiplier (default: 1.0)
uniform vec3  uAmbientColor;      // Ambient color (default: #665A5A warm grey)
uniform float uAmbientIntensity;  // Ambient multiplier (default: 0.4)

// Material switches
uniform float uAlphaThreshold;   // Alpha cutoff for discard (default: 0.5)
uniform bool  uTwoSided;         // Render both faces (for hair/cloth)
uniform float uVertexColorMix;   // 0.0 = ignore vertex color, 1.0 = full multiply (default: 1.0)

// ─── INPUTS FROM VERTEX SHADER ────────────────────────────────────────────────
in vec2  vTexCoord;
in vec2  vTexCoord2;
in vec4  vVertexColor;
in vec3  vNormalWorld;
in vec3  vFragPosWorld;

// ─── OUTPUT ───────────────────────────────────────────────────────────────────
out vec4 FragColor;

void main() {
    // 1. SAMPLE DIFFUSE TEXTURE
    vec4 texColor = texture(uDiffuseTexture, vTexCoord);
    
    // 2. ALPHA TEST — discard transparent pixels immediately
    // This writes to depth buffer (no Z-sort needed for characters)
    if (texColor.a < uAlphaThreshold) {
        discard;
    }
    
    // 3. TWO-SIDED RENDERING — flip normal for back faces
    // gl_FrontFacing is false when rendering back face
    vec3 normal = vNormalWorld;
    if (uTwoSided && !gl_FrontFacing) {
        normal = -normal;
    }
    
    // 4. DIFFUSE LIGHTING — classic Lambert (N · L)
    // clamp to [0,1]: no negative light on back faces
    float NdotL = max(dot(normal, normalize(uLightDirection)), 0.0);
    vec3 diffuse = uLightColor * uLightIntensity * NdotL;
    
    // 5. AMBIENT TERM — constant light from all directions
    vec3 ambient = uAmbientColor * uAmbientIntensity;
    
    // 6. COMBINE LIGHT TERMS
    vec3 lighting = ambient + diffuse;
    
    // 7. LIGHTMAP (optional UV2 baked light)
    if (uUseLightmap) {
        vec3 lightmapColor = texture(uLightmapTexture, vTexCoord2).rgb;
        // Lightmap modulates the existing lighting
        // Classic RO: lightmap IS the lighting, overrides dynamic
        lighting = lightmapColor;
    }
    
    // 8. VERTEX COLOR MULTIPLY
    // Classic RO uses vertex colors heavily for per-vertex ambient shading
    // vVertexColor.rgb is typically a baked shadow/ambient value
    vec3 vertColor = mix(vec3(1.0), vVertexColor.rgb, uVertexColorMix);
    
    // 9. FINAL COLOR COMPOSITION
    // FinalColor = Texture × VertexColor × Lighting
    vec3 finalRGB = texColor.rgb * vertColor * lighting;
    
    // 10. OUTPUT
    FragColor = vec4(finalRGB, texColor.a);
}
```

---

## 20.4 SHADER UNIFORM SETUP (JavaScript / Three.js)

```javascript
import * as THREE from 'three';

// RO-style warm directional light defaults
const RO_LIGHT_DEFAULTS = {
    direction:        new THREE.Vector3(0.5, 1.0, 0.3).normalize(),
    color:            new THREE.Color(0xFFF3D9),   // warm white
    intensity:        1.0,
    ambientColor:     new THREE.Color(0x665A5A),   // warm grey
    ambientIntensity: 0.4,
    alphaThreshold:   0.5,
    vertexColorMix:   1.0,
    useLightmap:      false,
    twoSided:         false
};

function createROCharacterMaterial(diffuseTexture, options = {}) {
    const cfg = { ...RO_LIGHT_DEFAULTS, ...options };
    
    return new THREE.RawShaderMaterial({
        uniforms: {
            uDiffuseTexture:   { value: diffuseTexture },
            uLightmapTexture:  { value: null },
            uUseLightmap:      { value: cfg.useLightmap },
            uLightDirection:   { value: cfg.direction },
            uLightColor:       { value: cfg.color },
            uLightIntensity:   { value: cfg.intensity },
            uAmbientColor:     { value: cfg.ambientColor },
            uAmbientIntensity: { value: cfg.ambientIntensity },
            uAlphaThreshold:   { value: cfg.alphaThreshold },
            uTwoSided:         { value: cfg.twoSided },
            uVertexColorMix:   { value: cfg.vertexColorMix },
            uModel:            { value: new THREE.Matrix4() },
            uView:             { value: new THREE.Matrix4() },
            uProjection:       { value: new THREE.Matrix4() },
            uNormalMatrix:     { value: new THREE.Matrix3() }
        },
        vertexShader: CHARACTER_VERT,    // paste shader source strings
        fragmentShader: CHARACTER_FRAG,
        transparent: false,
        depthWrite: true,
        depthTest: true,
        side: cfg.twoSided ? THREE.DoubleSide : THREE.FrontSide,
        glslVersion: THREE.GLSL3
    });
}

// Update per-frame uniforms in render loop
function updateCharacterUniforms(mesh, camera) {
    const mat = mesh.material;
    mesh.updateWorldMatrix(true, false);
    
    mat.uniforms.uModel.value.copy(mesh.matrixWorld);
    mat.uniforms.uView.value.copy(camera.matrixWorldInverse);
    mat.uniforms.uProjection.value.copy(camera.projectionMatrix);
    
    // Normal matrix = inverse transpose of model matrix
    mat.uniforms.uNormalMatrix.value
        .getNormalMatrix(mesh.matrixWorld);
}
```

---

## 20.5 VARIANT: PSEUDO-METALLIC SHADER

For armor and equipment with shiny surfaces (Swordsman plate, etc.):

```glsl
// FRAGMENT SHADER ADDITION — adds specular highlight
// Add after the ambient + diffuse calculation:

// 5b. SPECULAR TERM (Blinn-Phong)
// Only for metallic/shiny materials
uniform vec3  uCameraPos;       // Camera world position
uniform float uSpecularPower;   // Shininess (16–64 for metal)
uniform float uSpecularStrength; // 0.0–1.0

vec3 viewDir  = normalize(uCameraPos - vFragPosWorld);
vec3 halfDir  = normalize(normalize(uLightDirection) + viewDir);
float NdotH   = max(dot(normal, halfDir), 0.0);
float specular = pow(NdotH, uSpecularPower) * uSpecularStrength;
vec3 specularColor = uLightColor * specular;

// Add to lighting: ambient + diffuse + specular
vec3 lighting = ambient + diffuse + specularColor;
```

---

## 20.6 VERTEX COLOR CHANNELS CONVENTION

```
ROCHAR VERTEX COLOR CHANNEL CONVENTION:
  R channel: shadow occlusion factor (0=fully shadowed, 1=fully lit)
  G channel: secondary shadow / depth cue
  B channel: reserved (default 1.0)
  A channel: material mask (0=no vertex color effect, 1=full effect)
  
  When painting vertex colors in Blender:
  → Dark corners / under chin / inside sleeves: low R (shadow)
  → Exposed top surfaces: high R (lit)
  → Hair tips can have slightly reduced R for depth
  
VERTEX COLOR IS NOT:
  → Specular data
  → Rim light
  → Emissive glow
  Those use separate materials or texture masks.
```

---

## 20.7 SHADER QUALITY CHECKLIST

```
□ No conditional branching inside hot loops (use step() / mix() instead)
□ Texture sample called once, stored in variable (not sampled twice)
□ Normal normalized in fragment shader (interpolation un-normalizes it)
□ Alpha threshold applied before any lighting calculations (early exit)
□ Vertex color mix=0 when vertex colors not painted (avoid grey multiply)
□ Lightmap UV2 valid if uUseLightmap=true
□ NormalMatrix updated every frame if object moves
□ Both FrontSide and DoubleSide variants available as separate materials
```
