# 21 — GLSL WORLD AND LIGHTMAP SHADER
## Terrain, Lightmap Blending, and World Surface Rendering

> **Purpose:** Render RO-style world terrain with dual-UV lightmap compositing and water effects.  
> **Related:** [20_glsl_character_shader.md](20_glsl_character_shader.md), [../references/01_ro_visual_dna.md](../references/01_ro_visual_dna.md)

---

## 21.1 DUAL-UV LIGHTMAP CONCEPT

```
CHANNEL 1 (aTexCoord):  Albedo / Tile texture UV
  → Repeating tile texture (stone, grass, marble)
  → Typically repeats every 1 world unit
  → Source: uTileTexture sampler

CHANNEL 2 (aTexCoord2): Lightmap UV
  → Non-repeating, unique per terrain patch
  → Covers the entire map section in 0–1 range
  → Pre-baked shadows and lighting from Blender/external baker
  → Source: uLightmapTexture sampler

COMPOSITING EQUATION:
  FinalColor = TileColor × LightmapColor × GlobalLightColor

Where:
  TileColor      = texture(uTileTexture, aTexCoord)
  LightmapColor  = texture(uLightmapTexture, aTexCoord2)
  GlobalLightColor = scene ambient/fog tint
```

---

## 21.2 TERRAIN VERTEX SHADER

```glsl
#version 330 core

// ─── ATTRIBUTES ───────────────────────────────────────────────────────────────
layout (location = 0) in vec3 aPosition;    // Terrain vertex (Y = height)
layout (location = 1) in vec3 aNormal;      // Vertex normal
layout (location = 2) in vec2 aTexCoord;    // Tile UV (repeating)
layout (location = 3) in vec2 aTexCoord2;   // Lightmap UV (unique 0–1)
layout (location = 4) in vec4 aVertexColor; // Optional terrain tint

// ─── UNIFORMS ─────────────────────────────────────────────────────────────────
uniform mat4 uModel;
uniform mat4 uView;
uniform mat4 uProjection;
uniform mat3 uNormalMatrix;

// Tile texture tiling factor
uniform float uTileRepeat;  // How many times tile repeats (default: 1.0)

// ─── OUTPUTS ──────────────────────────────────────────────────────────────────
out vec2 vTileUV;
out vec2 vLightmapUV;
out vec4 vVertexColor;
out vec3 vNormalWorld;
out vec3 vFragPosWorld;
out float vFogFactor;   // Precomputed fog blend (cheaper here than fragment)

// Fog parameters
uniform float uFogStart;    // Distance where fog begins
uniform float uFogEnd;      // Distance where fog is 100%

void main() {
    vec4 worldPos = uModel * vec4(aPosition, 1.0);
    vFragPosWorld = worldPos.xyz;
    vNormalWorld  = normalize(uNormalMatrix * aNormal);
    vVertexColor  = aVertexColor;
    
    // Scale tile UV by repeat factor
    vTileUV      = aTexCoord * uTileRepeat;
    vLightmapUV  = aTexCoord2;
    
    // Fog: linear fog based on camera distance
    vec4 viewPos = uView * worldPos;
    float camDist = length(viewPos.xyz);
    vFogFactor = clamp((uFogEnd - camDist) / (uFogEnd - uFogStart), 0.0, 1.0);
    
    gl_Position = uProjection * viewPos;
}
```

---

## 21.3 TERRAIN FRAGMENT SHADER

```glsl
#version 330 core

// ─── UNIFORMS ─────────────────────────────────────────────────────────────────
uniform sampler2D uTileTexture;      // Repeating tile (stone/grass/etc.)
uniform sampler2D uLightmapTexture;  // Baked lightmap (unique per patch)
uniform bool      uUseLightmap;      // Toggle lightmap compositing

// Light (for areas without baked lightmap)
uniform vec3  uLightDirection;
uniform vec3  uLightColor;
uniform float uLightIntensity;
uniform vec3  uAmbientColor;
uniform float uAmbientIntensity;

// Fog
uniform vec3  uFogColor;    // Color of fog (usually sky color)

// Color grading
uniform vec3  uGlobalTint;  // Multiply all colors by this (zone atmosphere)
uniform float uBrightness;  // Additional brightness offset (0.0 = no change)

// ─── INPUTS ───────────────────────────────────────────────────────────────────
in vec2  vTileUV;
in vec2  vLightmapUV;
in vec4  vVertexColor;
in vec3  vNormalWorld;
in vec3  vFragPosWorld;
in float vFogFactor;

// ─── OUTPUT ───────────────────────────────────────────────────────────────────
out vec4 FragColor;

void main() {
    // 1. SAMPLE TILE TEXTURE
    vec4 tileColor = texture(uTileTexture, vTileUV);
    
    // 2. SAMPLE LIGHTMAP
    vec3 lightmapRGB = vec3(1.0); // Default: no lightmap = fully lit
    if (uUseLightmap) {
        lightmapRGB = texture(uLightmapTexture, vLightmapUV).rgb;
        // Lightmap is in sRGB space — decode to linear for correct multiply
        lightmapRGB = pow(lightmapRGB, vec3(2.2));
    }
    
    // 3. DYNAMIC LIGHTING (when no lightmap or as supplement)
    float NdotL = max(dot(vNormalWorld, normalize(uLightDirection)), 0.0);
    vec3 diffuse = uLightColor * uLightIntensity * NdotL;
    vec3 ambient = uAmbientColor * uAmbientIntensity;
    vec3 dynamicLight = ambient + diffuse;
    
    // 4. CHOOSE LIGHTING SOURCE
    vec3 finalLight;
    if (uUseLightmap) {
        // Lightmap takes over — multiply tile by lightmap
        // FinalColor = TileColor × LightmapColor
        finalLight = lightmapRGB;
    } else {
        finalLight = dynamicLight;
    }
    
    // 5. VERTEX COLOR TINT (terrain color variation)
    vec3 terrainTint = vVertexColor.rgb;
    
    // 6. GLOBAL ATMOSPHERE TINT (zone-specific color cast)
    // e.g., Prontera: neutral, Morroc: warm sandy, Ice Dungeon: cold blue
    vec3 finalRGB = tileColor.rgb * terrainTint * finalLight * uGlobalTint;
    
    // 7. BRIGHTNESS BOOST (some zones are slightly overexposed)
    finalRGB += uBrightness;
    
    // 8. GAMMA CORRECTION (output to sRGB display)
    // Only if rendering to linear framebuffer
    // finalRGB = pow(finalRGB, vec3(1.0 / 2.2));
    
    // 9. FOG BLEND
    // vFogFactor: 1.0 = no fog, 0.0 = full fog
    finalRGB = mix(uFogColor, finalRGB, vFogFactor);
    
    // 10. OUTPUT (terrain is always opaque)
    FragColor = vec4(finalRGB, 1.0);
}
```

---

## 21.4 ZONE ATMOSPHERE PRESETS

```javascript
const ZONE_ATMOSPHERES = {
    prontera: {
        fogColor:     [0.75, 0.80, 0.90],  // slight blue sky
        fogStart:     20.0,
        fogEnd:       60.0,
        globalTint:   [1.00, 0.98, 0.95],  // neutral warm
        brightness:   0.0,
        ambientColor: [0.40, 0.38, 0.35],
        lightColor:   [1.00, 0.95, 0.80]   // warm white
    },
    morroc: {
        fogColor:     [0.90, 0.75, 0.50],  // sandy haze
        fogStart:     15.0,
        fogEnd:       45.0,
        globalTint:   [1.05, 0.95, 0.80],  // warm sandy cast
        brightness:   0.02,
        ambientColor: [0.50, 0.42, 0.30],
        lightColor:   [1.10, 0.90, 0.60]   // orange sun
    },
    ice_dungeon: {
        fogColor:     [0.60, 0.70, 0.90],  // cold blue mist
        fogStart:     8.0,
        fogEnd:       30.0,
        globalTint:   [0.85, 0.90, 1.05],  // cold cast
        brightness:  -0.02,
        ambientColor: [0.35, 0.40, 0.55],
        lightColor:   [0.80, 0.85, 1.00]   // cool blue-white
    },
    payon: {
        fogColor:     [0.60, 0.65, 0.50],  // green-grey
        fogStart:     18.0,
        fogEnd:       50.0,
        globalTint:   [0.95, 1.00, 0.90],  // slight green tint
        brightness:   0.0,
        ambientColor: [0.38, 0.40, 0.32],
        lightColor:   [0.85, 0.90, 0.70]   // filtered forest light
    }
};
```

---

## 21.5 WATER SHADER

```glsl
#version 330 core
// WATER VERTEX SHADER — sine wave displacement

layout (location = 0) in vec3 aPosition;
layout (location = 1) in vec2 aTexCoord;

uniform mat4 uModel;
uniform mat4 uView;
uniform mat4 uProjection;
uniform float uTime;         // Elapsed seconds
uniform float uWaveHeight;   // Max displacement (default: 0.03)
uniform float uWaveFreq;     // Wave frequency (default: 2.5)
uniform float uWaveSpeed;    // Wave scroll speed (default: 0.8)

out vec2  vTexCoord;
out float vWaveNoise;

void main() {
    vec3 pos = aPosition;
    
    // Two sine waves at different angles for natural look
    float wave1 = sin(pos.x * uWaveFreq + uTime * uWaveSpeed) * uWaveHeight;
    float wave2 = sin(pos.z * uWaveFreq * 1.3 + uTime * uWaveSpeed * 0.7) * uWaveHeight * 0.6;
    pos.y += wave1 + wave2;
    
    vTexCoord   = aTexCoord;
    vWaveNoise  = (wave1 + wave2) / uWaveHeight; // -1 to 1 range
    
    gl_Position = uProjection * uView * uModel * vec4(pos, 1.0);
}

// ─────────────────────────────────────────────────────────────
// WATER FRAGMENT SHADER
// ─────────────────────────────────────────────────────────────
#version 330 core

uniform sampler2D uWaterTexture;  // Animated water normal/color tile
uniform float     uTime;
uniform vec3      uWaterColor;    // Base water color (zone-dependent)
uniform float     uOpacity;       // Water opacity (default: 0.75)

in vec2  vTexCoord;
in float vWaveNoise;

out vec4 FragColor;

void main() {
    // Scroll water texture at two speeds/directions
    vec2 uv1 = vTexCoord + vec2(uTime * 0.04, uTime * 0.02);
    vec2 uv2 = vTexCoord + vec2(-uTime * 0.02, uTime * 0.03);
    
    vec4 water1 = texture(uWaterTexture, uv1);
    vec4 water2 = texture(uWaterTexture, uv2);
    
    // Blend two scroll layers for depth
    vec3 waterColor = mix(water1.rgb, water2.rgb, 0.5) * uWaterColor;
    
    // Slight brightness boost at wave crests
    float crestBright = max(0.0, vWaveNoise) * 0.15;
    waterColor += crestBright;
    
    FragColor = vec4(waterColor, uOpacity);
}
```

---

## 21.6 LIGHTMAP BAKING SETTINGS (BLENDER)

```
BAKE SETTINGS FOR RO-STYLE LIGHTMAPS:

Render Engine: Cycles (for quality) or Eevee (for speed)
Bake Type:     Combined (ambient + diffuse shadows)

Render Output:
  Format:  PNG 8-bit (or EXR for HDR)
  Size:    512×512 per terrain section (small maps)
           1024×1024 per terrain section (larger areas)
  Margin:  4 pixels (prevents seams at atlas borders)
  
Sampling:
  Samples: 32–64 (more = less noise)
  
Light Setup:
  Sun: position matches game's directional light direction
  Sky: use world color matching zone atmosphere preset
  HDRI: optional for more realistic ambient

POST-PROCESS:
  1. Blur lightmap slightly (Gaussian 1px) — softens noise
  2. Boost contrast slightly to match RO's deeper shadows
  3. Convert to sRGB for storage (8-bit PNG)
  4. Load in shader with sRGB colorspace flag

LIGHTMAP UV CHANNEL:
  UV channel index: 1 (zero-indexed) = second UV set
  Blender: Object → Data → UV Maps → "LightmapUV"
```
