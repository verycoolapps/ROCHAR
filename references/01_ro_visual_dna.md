# 01 — RAGNAROK ONLINE VISUAL DNA
## The Defining Aesthetic of Classic RO

> **Purpose:** Establish the foundational visual principles ROCHAR must preserve.  
> **Related:** [02_character_proportion_system.md](02_character_proportion_system.md), [07_color_and_material_language.md](07_color_and_material_language.md)

---

## 1.1 WHAT MAKES RO LOOK LIKE RO

Classic Ragnarok Online possesses an immediately recognizable visual signature formed by:

1. **Low-polygon 3D environments** with relatively flat geometry and dense tile-like patterns
2. **2D billboard-rendered characters** that maintain facing orientation toward the camera
3. **Heavily stylized character proportions** — compact bodies, oversized expressive heads
4. **Saturated, vivid color palettes** with clean color blocking rather than gradient realism
5. **Baked lighting** in world textures rather than real-time per-pixel illumination
6. **Vertex color contributions** to environment geometry tinting
7. **Minimal specular response** — surfaces read as matte or subtly glossy
8. **Hand-drawn texture quality** with visible pixel-art characteristics at close range
9. **Directional sprite animation** using 8-direction facing sets

---

## 1.2 THE VISUAL TRIANGLE

```
           ENVIRONMENT
          (Low-poly 3D)
              /\
             /  \
            /    \
           /      \
      CHARACTERS  EFFECTS
     (2D sprites  (particle
     or 3D mesh)   / STR)
```

The visual coherence of RO depends on these three layers remaining visually compatible.  
When one layer adopts radically different rendering characteristics, the unified aesthetic breaks.

**Rule:** If your character looks more modern than the world it inhabits, you have failed the visual triangle test.

---

## 1.3 LIGHTING PHILOSOPHY

Classic RO lighting operates on a **simplified directional model**:

```
FinalColor = TextureColor × VertexColor × (Ambient + Diffuse)
Diffuse    = max(dot(Normal, LightDir), 0.0) × LightColor
```

**Key characteristics:**
- Single dominant directional light source (typically top-left or top-right)
- Warm ambient fill (soft peach/warm white — NEVER cold blue)
- Baked shadows painted directly into texture, not real-time shadow maps
- No bump mapping or normal maps in classic implementation
- No physically-based reflectance model
- Minimal or absent specular highlight

**What this means for ROCHAR:**  
Applying modern PBR materials to RO characters instantly destroys the aesthetic.  
The correct approach is a simplified diffuse shader with intentionally **zero or near-zero specular**.

---

## 1.4 TEXTURE CHARACTERISTICS

| Property | Classic RO Value |
|----------|-----------------|
| Resolution | 64×64 to 256×256 typical |
| Format | BMP/PNG with optional transparency |
| Filtering | Nearest-neighbor or low-quality bilinear |
| Color depth | 8-bit palette or 24-bit RGB |
| Alpha channel | 1-bit (opaque/transparent) or 8-bit |
| Compression | None or basic |
| Normal maps | Not present in classic assets |
| Roughness maps | Not applicable — fixed material model |
| Metalness maps | Not applicable |

**Characteristic visual quality:** Clean flat colors with hand-painted shading baked into the diffuse layer. Gradients are subtle. Outlines may be baked in. Specular highlights, if present, are painted as bright spots — not dynamically computed.

---

## 1.5 ENVIRONMENT GEOMETRY

Classic RO world geometry (GND format) uses a **tile-based approach**:

- Each tile is a quadrilateral aligned to a uniform grid
- Terrain height per tile corner (allows smooth hills without fine geometry)
- Each tile face has its own texture reference (no texture atlasing in original)
- Lightmap UVs stored in a **separate UV channel** from diffuse UVs
- Vertex colors contribute ambient occlusion approximation and tinting
- 3D objects (RSM models) placed into the world scene (RSW format)
- Simple alpha-blend foliage and transparent surfaces sorted manually

**Common polygon budget per visible map section:**
- Terrain: 2,000–8,000 triangles
- Foreground objects: 500–3,000 triangles
- Background details: 200–1,000 triangles

---

## 1.6 CHARACTER VISUAL REPRESENTATION

Classic RO character representation varies by context:

| Context | Representation | Notes |
|---------|---------------|-------|
| Field/town walking | 2D directional sprite billboarded | 8 directions, animated frames |
| Equipment display | 2D sprite layers composited | Head, body, weapon, shadow layers |
| World objects, some NPCs | Low-poly 3D RSM models | Simple geometry, baked texture |
| Mounts/vehicles | Low-poly 3D RSM models | Animated via RSM animation keys |
| Skill effects | STR particle/effect files | 2D effect sprites, additive blended |

**CRITICAL:** Do not assume all characters are fully-rigged 3D humanoids.  
Classic RO characters are **primarily sprite-based**. 3D is used for environment objects and mounts.

---

## 1.7 COLOR AND MOOD VOCABULARY

| World Zone | Dominant Palette | Mood |
|-----------|-----------------|------|
| Prontera | Warm stone, golden brown, verdant green | Safety, civilization |
| Payon | Bamboo greens, wood browns, lantern orange | Eastern, calm, traditional |
| Geffen | Purple-grey stone, mystical blues, arcane violet | Magic, arcane power |
| Morroc | Desert tan, terracotta, dry sand, sun-bleached | Harsh, arid, dangerous |
| Izlude | Ocean blue, sandy shores, sky cyan | Nautical, breezy, light |
| Glast Heim | Dark grey stone, ash tones, purple-black | Gothic, dread, undead |
| Comodo | Sandy beach, tropical green, vivid blue water | Relaxed, festive |
| Lutie | Snow white, candy red, sky blue, pine green | Festive, childlike wonder |

Character design should harmonize with the intended zone palette while maintaining **individual class identity**.

---

## 1.8 VISUAL MISTAKES TO AVOID

| Mistake | Why It Breaks the Aesthetic |
|---------|---------------------------|
| PBR metallic/roughness maps on armor | Hyper-realistic reflections incompatible with original flat style |
| High-resolution normal maps on skin | Creates micro-detail that reads as "modern remake," not classic |
| Subsurface scattering on skin | Characters look biological rather than stylized illustration |
| AO baking too dark | Flattens the colorful palette into muddy grey |
| Realistic hair simulation (strands) | Loses angular stylized silhouette that defines RO hair |
| Smooth subdivision on body mesh | Loses intentional low-poly character — too "3D animated movie" |
| Monochromatic color scheme | RO characters have distinct multi-color costumes for readability |
| Western anatomical proportions (1:7+ head-to-body) | Too tall and realistic — destroys chibi identity |
| Cold blue ambient light | RO ambient is warm — cold blue reads as modern game, not classic RO |
| Real-time dynamic shadows | Classic RO has baked shadows — dynamic shadows look anachronistic |

---

## 1.9 THE NOSTALGIA COEFFICIENT

A successful ROCHAR output must trigger recognition in a player who last played RO in 2004 — without explanation.

**The Nostalgia Test:**
1. Show the character at game-view distance (~500px height)
2. Ask: "Does this feel like RO?"
3. If yes without qualification needed → **PASS**
4. If it needs "it's RO-inspired but modernized" → **PARTIAL PASS** (acceptable for 3D pathway)
5. If it looks like a generic MMORPG or generic anime → **FAIL**

**The nostalgia coefficient is not just visual. It includes:**
- Scale and proportion feel (chibi, not realistic)
- Color blocking readability (vivid blocks, not photorealistic gradients)
- Silhouette uniqueness per class (hat, weapon, posture)
- Animation cadence (sprite animations have characteristic timing)
- Material behavior (flat, not shiny)

---

## 1.10 RENDERING RESOLUTION AND SCALE CONTEXT

Classic RO was rendered at 800×600 or 1024×768.  
Characters occupied approximately 50–120 pixels in height at typical distance.

**Implications for asset creation:**
- Detail below 4px at game distance is invisible — do not add it to geometry
- Texture detail below 2px is invisible — do not add it to textures
- Silhouette must read clearly at thumbnail scale (100×100px or smaller)
- Color should distinguish class and equipment at a glance — no reliance on fine texture detail

---

## 1.11 QUICK DIAGNOSTIC CHECKLIST

Before declaring any ROCHAR output complete, verify:

```
□ Does the character have the correct chibi proportions? (not realistic)
□ Are the colors vivid and cleanly blocked? (not desaturated or gradient-heavy)
□ Is the lighting simplified? (no PBR, no dynamic shadows)
□ Does the silhouette communicate class/role in under 3 seconds?
□ Does the texture filtering match the aesthetic? (nearest-neighbor preferred)
□ Is vertex color applied where appropriate?
□ Does the character pass the nostalgia test?
□ Is there any accidental modernity (subsurface scattering, metalness, etc.)?
```
