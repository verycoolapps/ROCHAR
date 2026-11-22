# 30 — THREE.JS AND OPENGL INTEGRATION
## Full Scene Setup, Render Loop, and Character Management

> **Purpose:** Complete Three.js integration code for displaying RO characters in a web browser.  
> **Related:** [../modeling/15_character_export_pipeline.md](../modeling/15_character_export_pipeline.md), [../shaders/22_glsl_billboard_shader.md](../shaders/22_glsl_billboard_shader.md)

---

## 30.1 SCENE INITIALIZATION

```javascript
import * as THREE from 'three';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

class ROCharacterScene {
    constructor(canvas) {
        this.canvas     = canvas;
        this.characters = new Map();   // id → ROCharacter
        this.billboards = new Map();   // id → billboard mesh
        this.clock      = new THREE.Clock();
        
        this._initRenderer();
        this._initScene();
        this._initCamera();
        this._initLighting();
        this._initControls();
        
        // Start render loop
        this.renderer.setAnimationLoop(() => this.update());
        
        // Handle resize
        window.addEventListener('resize', () => this.resize());
    }
    
    _initRenderer() {
        this.renderer = new THREE.WebGLRenderer({
            canvas:    this.canvas,
            antialias: false,       // No antialiasing (preserves pixel art edges)
            alpha:     false,
            powerPreference: 'high-performance'
        });
        
        // Classic RO: render at native resolution (no MSAA needed for pixel art)
        this.renderer.setPixelRatio(window.devicePixelRatio);
        this.renderer.setSize(window.innerWidth, window.innerHeight);
        
        // Output encoding: sRGB for correct gamma
        this.renderer.outputColorSpace = THREE.SRGBColorSpace;
        
        // Shadows: disabled for performance (RO uses baked shadows)
        this.renderer.shadowMap.enabled = false;
        
        // Sorting: needed for transparent sprites
        this.renderer.sortObjects = true;
    }
    
    _initScene() {
        this.scene = new THREE.Scene();
        
        // Classic RO sky color (Prontera zone default)
        this.scene.background = new THREE.Color(0xC8D8E8);
        
        // Fog matching zone atmosphere
        this.scene.fog = new THREE.Fog(0xC8D8E8, 20, 60);
    }
    
    _initCamera() {
        const aspect = window.innerWidth / window.innerHeight;
        this.camera = new THREE.PerspectiveCamera(45, aspect, 0.1, 100);
        
        // Classic RO: slightly elevated isometric-style view
        this.camera.position.set(0, 4, 8);
        this.camera.lookAt(0, 0.5, 0);
    }
    
    _initLighting() {
        // Ambient: warm grey (from zone atmospheres reference)
        const ambient = new THREE.AmbientLight(0x665A5A, 0.4);
        this.scene.add(ambient);
        
        // Directional: warm sun light
        const sun = new THREE.DirectionalLight(0xFFF3D9, 1.0);
        sun.position.set(0.5, 1.0, 0.3).normalize();
        this.scene.add(sun);
    }
    
    _initControls() {
        this.controls = new OrbitControls(this.camera, this.canvas);
        this.controls.target.set(0, 0.5, 0);
        this.controls.minDistance = 2;
        this.controls.maxDistance = 30;
        this.controls.maxPolarAngle = Math.PI * 0.6; // Don't go below ground
    }
}
```

---

## 30.2 CHARACTER LOADING

```javascript
class ROCharacterScene {
    // ... (continued)
    
    /**
     * Load a 3D GLB character and add to scene.
     */
    async loadCharacter(id, glbUrl, options = {}) {
        const loader  = new GLTFLoader();
        const gltf    = await loader.loadAsync(glbUrl);
        
        // Fix texture colorspace for RO (sRGB for albedo)
        gltf.scene.traverse(child => {
            if (child.isMesh && child.material) {
                const mat = child.material;
                if (mat.map) mat.map.colorSpace = THREE.SRGBColorSpace;
                
                // Nearest-neighbor for pixel art textures
                if (mat.map) {
                    mat.map.magFilter = THREE.NearestFilter;
                    mat.map.minFilter = THREE.NearestFilter;
                    mat.map.generateMipmaps = false;
                }
                
                // Classic RO material settings
                mat.transparent = false;
                mat.alphaTest   = 0.5;
                mat.depthWrite  = true;
                mat.side        = THREE.FrontSide;
            }
        });
        
        // Set up animation mixer
        const animator = new ROCharacterAnimator(gltf);
        animator.play('idle');
        
        // Position on ground (Y=0)
        gltf.scene.position.set(
            options.x ?? 0, 
            0, 
            options.z ?? 0
        );
        
        this.scene.add(gltf.scene);
        this.characters.set(id, {
            scene:    gltf.scene,
            animator: animator,
            gltf:     gltf
        });
        
        return gltf.scene;
    }
    
    /**
     * Load a sprite-based billboard character.
     */
    async loadBillboardCharacter(id, atlasUrl, metadataUrl, options = {}) {
        // Load atlas texture
        const texLoader = new THREE.TextureLoader();
        const atlas     = await texLoader.loadAsync(atlasUrl);
        atlas.magFilter = THREE.NearestFilter;
        atlas.minFilter = THREE.NearestFilter;
        atlas.generateMipmaps = false;
        atlas.colorSpace = THREE.SRGBColorSpace;
        
        // Load metadata
        const metaResp = await fetch(metadataUrl);
        const metadata = await metaResp.json();
        
        // Create animation controller
        const animCtrl = new SpriteAnimationController(metadata);
        animCtrl.play('idle');
        
        // Create billboard mesh (see 22_glsl_billboard_shader.md)
        const billboard = createROBillboard(
            atlas,
            metadata.world_width ?? 0.7,
            metadata.world_height ?? 1.0
        );
        
        billboard.position.set(options.x ?? 0, 0, options.z ?? 0);
        this.scene.add(billboard);
        
        this.billboards.set(id, {
            mesh:     billboard,
            animCtrl: animCtrl,
            metadata: metadata
        });
        
        return billboard;
    }
    
    /**
     * Main update loop.
     */
    update() {
        const delta = this.clock.getDelta();
        
        // Update 3D character animators
        this.characters.forEach(char => {
            char.animator.update(delta);
        });
        
        // Update billboard animations and face camera
        this.billboards.forEach(billboard => {
            billboard.animCtrl.update(delta);
            
            // Update direction based on camera-relative facing
            const dir = getSpriteFacingDirection(
                billboard.mesh.position,
                this.camera,
                billboard.facingAngle ?? 0
            );
            billboard.animCtrl.setDirection(dir);
            
            // Push new frame to shader
            updateBillboard(
                billboard.mesh,
                billboard.animCtrl,
                this.camera,
                billboard.metadata
            );
        });
        
        this.controls.update();
        this.renderer.render(this.scene, this.camera);
    }
    
    resize() {
        const w = window.innerWidth;
        const h = window.innerHeight;
        this.camera.aspect = w / h;
        this.camera.updateProjectionMatrix();
        this.renderer.setSize(w, h);
    }
    
    dispose() {
        this.renderer.setAnimationLoop(null);
        this.characters.forEach(char => {
            char.animator.dispose();
        });
        this.renderer.dispose();
    }
}
```

---

## 30.3 TEXTURE LOADING SETTINGS

```javascript
// Correct Three.js texture settings for each RO texture type:

function configureROTexture(texture, type = 'diffuse') {
    switch (type) {
        case 'diffuse':
            // Character/terrain color texture
            texture.colorSpace      = THREE.SRGBColorSpace;
            texture.magFilter       = THREE.NearestFilter;
            texture.minFilter       = THREE.NearestFilter;
            texture.generateMipmaps = false;
            break;
        
        case 'lightmap':
            // Baked lightmap (linear space, can use mips for smooth look)
            texture.colorSpace      = THREE.LinearSRGBColorSpace;
            texture.magFilter       = THREE.LinearFilter;
            texture.minFilter       = THREE.LinearMipmapLinearFilter;
            texture.generateMipmaps = true;
            break;
        
        case 'sprite_atlas':
            // Sprite sheet (nearest-neighbor, sRGB)
            texture.colorSpace      = THREE.SRGBColorSpace;
            texture.magFilter       = THREE.NearestFilter;
            texture.minFilter       = THREE.NearestFilter;
            texture.generateMipmaps = false;
            texture.wrapS = THREE.ClampToEdgeWrapping;
            texture.wrapT = THREE.ClampToEdgeWrapping;
            break;
        
        case 'normal_map':
            // Normal/bump map (always linear, never sRGB)
            texture.colorSpace      = THREE.LinearSRGBColorSpace;
            texture.magFilter       = THREE.LinearFilter;
            texture.minFilter       = THREE.LinearFilter;
            break;
    }
    
    return texture;
}
```

---

## 30.4 RENDER ORDER AND LAYER MANAGEMENT

```javascript
// Render order constants (lower = rendered first)
const RENDER_ORDER = {
    TERRAIN:        0,
    SHADOW:         1,
    CHARACTER:      2,   // Alpha test, writes depth
    EFFECT_BLEND:   3,   // Alpha blend (needs depth test)
    EFFECT_ADDITIVE:4,   // Additive blend (no depth sort needed)
    UI_OVERLAY:     10
};

function applyRenderOrder(mesh, type) {
    mesh.renderOrder = RENDER_ORDER[type] ?? 2;
    
    switch (type) {
        case 'CHARACTER':
            mesh.material.transparent = false;
            mesh.material.depthWrite  = true;
            mesh.material.depthTest   = true;
            mesh.material.alphaTest   = 0.5;
            break;
        
        case 'SHADOW':
            mesh.material.transparent = true;
            mesh.material.depthWrite  = false;
            mesh.material.depthTest   = true;
            mesh.material.blending    = THREE.NormalBlending;
            break;
        
        case 'EFFECT_ADDITIVE':
            mesh.material.transparent = true;
            mesh.material.depthWrite  = false;
            mesh.material.blending    = THREE.AdditiveBlending;
            break;
    }
}
```

---

## 30.5 WEBGL RENDERER SETTINGS FOR RO LOOK

```javascript
// Settings that produce closest match to classic RO visual feel:

const renderer = new THREE.WebGLRenderer({
    antialias: false,  // No antialiasing (crisp pixel edges)
    alpha: false       // No alpha on main canvas (opaque background)
});

// Tone mapping: neutral (no HDR processing for classic look)
renderer.toneMapping          = THREE.NoToneMapping;
renderer.toneMappingExposure  = 1.0;

// No physical lights (RO uses non-physical flat diffuse)
renderer.physicallyCorrectLights = false;  // Three.js r149 and earlier
// (In r150+: renderer.useLegacyLights = true)

// Shadow maps: disabled (RO uses baked vertex/lightmap shadows)
renderer.shadowMap.enabled = false;

// Pixel ratio: device native (don't artificially upscale)
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
```
