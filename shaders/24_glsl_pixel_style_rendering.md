# 24 — GLSL PIXEL STYLE RENDERING
## Post-Processing, Nearest-Neighbor, and Retro Aesthetic Shaders

> **Purpose:** Implement pixel art post-processing that preserves the classic RO aesthetic on modern displays.  
> **Related:** [../references/01_ro_visual_dna.md](../references/01_ro_visual_dna.md), [25_shader_optimization.md](25_shader_optimization.md)

---

## 24.1 THE PIXEL ART RENDERING PROBLEM

```
ISSUE: Rendering pixel art at modern resolutions causes blurring.

CAUSE:  Default texture filtering (LINEAR) interpolates between pixels.
        At 4x zoom, a 64px sprite becomes 256px with smooth gradients.
        This destroys the crisp pixel art look.

SOLUTION A: Nearest-neighbor texture filtering (fastest, no shader needed)
  → texture.magFilter = THREE.NearestFilter
  → texture.minFilter = THREE.NearestFilter
  → Gives pixel-perfect upscaling with hard edges

SOLUTION B: Custom shader with crisp UV sampling
  → Snap UV coordinates to pixel boundaries in shader
  → More control over edge softening style

SOLUTION C: Render to low-res framebuffer, upscale with nearest-neighbor
  → Most authentic: entire scene at retro resolution
  → Best performance for many sprites
```

---

## 24.2 NEAREST-NEIGHBOR TEXTURE SETUP

```javascript
import * as THREE from 'three';

function loadROTexture(url) {
    const loader = new THREE.TextureLoader();
    const texture = loader.load(url);
    
    // CRITICAL: nearest-neighbor filtering for pixel art
    texture.magFilter = THREE.NearestFilter;    // Upscale: hard pixels
    texture.minFilter = THREE.NearestFilter;    // Downscale: also hard (no mip blur)
    
    // Disable mip-map generation (mips blur pixel art at distance)
    texture.generateMipmaps = false;
    
    // sRGB colorspace for albedo textures
    texture.colorSpace = THREE.SRGBColorSpace;
    
    return texture;
}
```

---

## 24.3 CRISP UV SHADER (SUBPIXEL-AWARE)

```glsl
#version 330 core
// Renders texture with snapped UVs for crisp pixel art
// Avoids bilinear blur at non-integer UV positions

uniform sampler2D uTexture;
uniform vec2      uTextureSize;   // Texture dimensions in pixels (e.g., vec2(512, 512))

in vec2 vTexCoord;
out vec4 FragColor;

void main() {
    // Calculate UV in texel space
    vec2 texelUV = vTexCoord * uTextureSize;
    
    // Snap to nearest texel center
    vec2 snappedUV = (floor(texelUV) + 0.5) / uTextureSize;
    
    vec4 color = texture(uTexture, snappedUV);
    
    if (color.a < 0.5) discard;
    FragColor = color;
}
```

---

## 24.4 LOW-RESOLUTION FRAMEBUFFER RENDERING

```javascript
/**
 * Render scene at retro resolution, then upscale to full screen.
 * This is the most authentic pixel art approach.
 */
class PixelArtRenderer {
    constructor(displayWidth, displayHeight, retroScale = 4) {
        this.retroWidth  = Math.floor(displayWidth  / retroScale);
        this.retroHeight = Math.floor(displayHeight / retroScale);
        
        // Create low-res render target
        this.renderTarget = new THREE.WebGLRenderTarget(
            this.retroWidth,
            this.retroHeight,
            {
                minFilter: THREE.NearestFilter,  // CRITICAL: nearest for upscale
                magFilter: THREE.NearestFilter,
                format: THREE.RGBAFormat,
                type: THREE.UnsignedByteType,
                generateMipmaps: false,
                depthBuffer: true,
                stencilBuffer: false
            }
        );
        
        // Full-screen quad to display the low-res render
        this.screenQuad = this._createScreenQuad();
    }
    
    _createScreenQuad() {
        const geo = new THREE.PlaneGeometry(2, 2);
        const mat = new THREE.ShaderMaterial({
            uniforms: {
                uTexture: { value: this.renderTarget.texture }
            },
            vertexShader: `
                varying vec2 vUV;
                void main() {
                    vUV = uv;
                    gl_Position = vec4(position.xy, 0.0, 1.0);
                }
            `,
            fragmentShader: `
                uniform sampler2D uTexture;
                varying vec2 vUV;
                void main() {
                    gl_FragColor = texture2D(uTexture, vUV);
                }
            `,
            depthWrite: false,
            depthTest: false
        });
        return new THREE.Mesh(geo, mat);
    }
    
    render(renderer, scene, camera) {
        // 1. Render scene to low-res target
        renderer.setRenderTarget(this.renderTarget);
        renderer.render(scene, camera);
        
        // 2. Display low-res target on screen (nearest-neighbor upscale)
        renderer.setRenderTarget(null);
        renderer.render(new THREE.Scene().add(this.screenQuad), 
                       new THREE.OrthographicCamera(-1, 1, 1, -1, 0, 1));
    }
    
    resize(displayWidth, displayHeight, retroScale) {
        const newW = Math.floor(displayWidth  / retroScale);
        const newH = Math.floor(displayHeight / retroScale);
        this.renderTarget.setSize(newW, newH);
    }
    
    dispose() {
        this.renderTarget.dispose();
    }
}
```

---

## 24.5 SCANLINE EFFECT SHADER

```glsl
#version 330 core
// Optional scanline overlay for retro CRT feel

uniform sampler2D uTexture;        // Input scene texture
uniform vec2      uResolution;     // Display resolution
uniform float     uScanlineIntensity; // 0=no effect, 0.3=subtle, 0.6=strong
uniform float     uScanlineCount;  // Lines per screen height (default: 240)

in vec2 vTexCoord;
out vec4 FragColor;

void main() {
    vec4 color = texture(uTexture, vTexCoord);
    
    // Create scanlines by modulating Y coordinate
    float scanY = vTexCoord.y * uScanlineCount;
    // Scanline: dark every other line
    float scanline = mod(floor(scanY), 2.0) < 1.0 ? 1.0 : (1.0 - uScanlineIntensity);
    
    color.rgb *= scanline;
    
    FragColor = color;
}
```

---

## 24.6 PALETTE QUANTIZATION SHADER

```glsl
#version 330 core
// Reduce color palette to N distinct colors for extra retro feel
// Mimics Game Boy / classic console color limitations

uniform sampler2D uTexture;
uniform int       uPaletteSize;   // Number of steps per channel (e.g., 8 = 3-bit)
uniform float     uDitherStrength; // 0=none, 0.5=light, 1=full

in vec2 vTexCoord;
in vec2 vScreenPos;  // Screen position in pixels
out vec4 FragColor;

// 4x4 Bayer dithering matrix
float bayerMatrix[16] = float[16](
     0.0/16.0,  8.0/16.0,  2.0/16.0, 10.0/16.0,
    12.0/16.0,  4.0/16.0, 14.0/16.0,  6.0/16.0,
     3.0/16.0, 11.0/16.0,  1.0/16.0,  9.0/16.0,
    15.0/16.0,  7.0/16.0, 13.0/16.0,  5.0/16.0
);

float getBayerValue(vec2 screenPos) {
    int x = int(mod(screenPos.x, 4.0));
    int y = int(mod(screenPos.y, 4.0));
    return bayerMatrix[y * 4 + x];
}

void main() {
    vec4 color = texture(uTexture, vTexCoord);
    
    // Add dithering offset to color before quantization
    float dither = getBayerValue(vScreenPos) * uDitherStrength;
    vec3 dithered = color.rgb + (dither - 0.5) * (1.0 / float(uPaletteSize));
    
    // Quantize: round to nearest palette step
    float step = 1.0 / float(uPaletteSize - 1);
    vec3 quantized = round(dithered / step) * step;
    quantized = clamp(quantized, 0.0, 1.0);
    
    FragColor = vec4(quantized, color.a);
}
```

---

## 24.7 PIXEL GRID OVERLAY

```glsl
#version 330 core
// Draw subtle grid lines between pixels for extra retro look

uniform sampler2D uTexture;
uniform vec2      uTextureSize;   // Actual texture resolution
uniform vec2      uDisplaySize;   // Screen display resolution
uniform float     uGridStrength;  // 0=invisible, 0.15=subtle, 0.4=strong
uniform vec3      uGridColor;     // Grid line color (default: black)

in vec2 vTexCoord;
out vec4 FragColor;

void main() {
    vec4 color = texture(uTexture, vTexCoord);
    
    // Calculate which pixel we're in and position within that pixel
    vec2 pixelPos = vTexCoord * uDisplaySize;
    vec2 fractPos = fract(pixelPos / (uDisplaySize / uTextureSize));
    
    // Create grid: dark at pixel boundaries (fractPos near 0 or 1)
    float gridX = step(1.0 - uGridStrength, fractPos.x) + step(fractPos.x, uGridStrength);
    float gridY = step(1.0 - uGridStrength, fractPos.y) + step(fractPos.y, uGridStrength);
    float grid  = clamp(gridX + gridY, 0.0, 1.0);
    
    // Blend grid color into scene
    color.rgb = mix(color.rgb, uGridColor, grid * uGridStrength);
    
    FragColor = color;
}
```

---

## 24.8 RECOMMENDED RETRO CONFIGURATION

```javascript
// Full retro rendering setup for authentic RO feel

const RETRO_CONFIG = {
    // Rendering resolution (render at 1/4 native, upscale to full)
    renderScale: 0.25,          // 1920×1080 → 480×270
    
    // Texture filtering
    textureFilter: 'nearest',   // No bilinear blur
    
    // Post-process pipeline (in order):
    effects: [
        { type: 'palette_quantize', paletteSize: 32, dithering: 0.2 },
        { type: 'scanline',         intensity: 0.15, count: 270 },
        { type: 'pixel_grid',       strength: 0.08 }
    ],
    
    // Lighting style
    lighting: {
        ambientIntensity: 0.4,
        shadowSoftness: 0,      // Hard shadows (not soft/blurred)
        vertexColorMix: 1.0     // Full vertex color influence
    }
};

// Pure RO feel: render at 480×270, nearest-neighbor upscale to 1920×1080
// No scanlines, no pixel grid — just the authentic nearest-neighbor upscale
// is enough to match the original RO look on modern displays
```
