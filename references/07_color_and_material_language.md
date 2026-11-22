# 07 — COLOR AND MATERIAL LANGUAGE
## RO Color Palette Philosophy and Stylized Material Behavior

> **Purpose:** Define color palette, texture approach, material behavior, and color combination rules.  
> **Related:** [01_ro_visual_dna.md](01_ro_visual_dna.md), [../shaders/20_glsl_character_shader.md](../shaders/20_glsl_character_shader.md)

---

## 7.1 COLOR PHILOSOPHY

Ragnarok Online uses a **vivid, saturated, clean-blocking** color approach.

**Three principles:**
1. **Saturation over realism** — colors are more vivid than real-world equivalents
2. **Clean blocking over gradients** — color areas have clear boundaries
3. **Contrast for readability** — colors are chosen to read clearly at game distance

---

## 7.2 SKIN TONE PALETTE

RO characters use a range of stylized skin tones, all slightly warmer/more vivid than real skin:

| Tone Name | Hex | RGB | Usage |
|-----------|-----|-----|-------|
| Light Cream | `#F5D5B0` | (245, 213, 176) | Pale/fair characters |
| Warm Peach | `#E8B88A` | (232, 184, 138) | Standard fair |
| Golden Tan | `#C89060` | (200, 144, 96) | Tanned adventurers |
| Medium Brown | `#A06840` | (160, 104, 64) | Darker skin tone |
| Deep Brown | `#7A4828` | (122, 72, 40) | Deep skin tone |

**Shadow on skin (painted in texture):**
Multiply base skin by approximately 0.7–0.75 for shadow areas.  
Example: `#F5D5B0` shadow → `#B09878`

**Highlight on skin:**
Add 15–20 to each channel in highlight areas.  
Keep highlights on forehead, cheek, nose, chin only.

---

## 7.3 CLASS-SPECIFIC COLOR ASSOCIATIONS

| Class | Primary Color | Secondary | Accent | Avoid |
|-------|-------------|-----------|--------|-------|
| Swordsman | Iron grey `#6B6B6B` | Leather brown | Gold `#C8A951` | Pastels |
| Knight | Steel silver `#A8A8A8` | Royal blue `#1A3A6B` | Gold trim | Bright primary |
| Mage | Deep purple `#4A2060` | Midnight blue | Arcane gold | Browns |
| Wizard | Void purple `#2D0040` | Black | Gold glow | Warm tones |
| Archer | Forest green `#2D5A1B` | Leather `#7A4A28` | Natural cream | Bright neon |
| Thief | Shadow black `#1A1A1A` | Dark navy | Blood red | Light pastels |
| Acolyte | Holy white `#F5F0E8` | Pale blue | Divine gold | Dark/evil tones |
| Merchant | Earth brown `#5C3A1E` | Commerce blue | Coin gold | Aggressive red |
| Blacksmith | Iron grey `#5A5A5A` | Forge orange | Heat red | Soft colors |
| Alchemist | Scholar green `#2D5A2D` | Lab white | Potion colors | None |

---

## 7.4 MATERIAL BEHAVIOR — CLASSIC RO MODEL

Classic RO does NOT use PBR. All surfaces follow a flat diffuse model.

### Flat Diffuse (Standard)
```
Appearance:   Matte, texture-driven
Specular:     Zero to minimal (< 0.05)
Reflections:  None
Roughness:    Conceptually 1.0
Examples:     Cloth, leather, skin, painted wood
```

### Pseudo-Metallic (Without PBR)
```
Appearance:   Metal suggested through:
              - Lighter texture value at center (highlight baked in)
              - Darker at edges (shadow baked in)
              - Single highlight streak painted in texture
              - Vertex color environment contribution
              
Specular:     Small highlight (0.1–0.2 Phong), or painted in texture
Examples:     Plate armor, sword blade, iron buckle

HOW TO ACHIEVE:
  → Paint texture with strong highlight (near-white) at top of plate
  → Add mid-tone in middle
  → Dark tone at recessed areas / edges
  → Apply slight warm vertex color (R:+10, G:+8, B:0) for warm metal feel
  → NO environment reflection maps
```

### Cloth Material
```
Appearance:   Soft, slightly wrinkled
Specular:     Zero
Shadow:       Painted in texture as darker value along fold lines
Highlight:    Baked into texture as subtle bright line along raised folds
Examples:     Robe, cape, shirt, skirt
```

### Leather Material
```
Appearance:   Slightly textured, matte
Value:        Medium-dark (0.3–0.5 brightness)
Specular:     Zero to minimal
Shadow:       Baked at seams and fold areas
Examples:     Armor padding, belt, boot upper, glove
```

---

## 7.5 CLOTHING SATURATION GUIDELINES

| Material Type | HSL Saturation | Notes |
|--------------|---------------|-------|
| Skin | 35–45% | Moderate saturation |
| Cloth (vivid) | 70–90% | Full saturation for class identity |
| Cloth (neutral) | 30–50% | Undershirts, lining |
| Leather | 40–60% | Natural but noticeable |
| Metal | 10–25% | Desaturated — metallic grey |
| Hair | 60–90% | High saturation for recognition |
| Magic effects | 85–100% | Maximum saturation |
| Stone/terrain | 10–30% | Low saturation |

---

## 7.6 COLOR COMBINATION RULES FOR RO CLASSES

### Triadic Scheme (classic RO approach)
Most RO characters use a **3-color system**:
```
COLOR 1 (60% area): Dominant class color — clothing
COLOR 2 (30% area): Secondary garment or armor
COLOR 3 (10% area): Accent — trim, buckles, weapon glow, eye color
```

### Examples by Class

```
KNIGHT:
  60%: Steel grey armor (#A8A8A8)
  30%: Royal blue cape (#1A3A6B)
  10%: Gold trim, buckles (#C8A951)

WIZARD:
  60%: Deep purple robe (#2D0040)
  30%: Black underlayer (#0A0A0A)
  10%: Arcane gold magic lines (#FFD700)

HUNTER:
  60%: Forest green coat (#2D5A1B)
  30%: Brown leather (#7A4A28)
  10%: Natural cream undershirt (#F0E8D0)
```

### Complementary Accent Rule
The accent color (10%) should be **complementary to** or **high-contrast against** the dominant color.

| Dominant Color | Complementary Accent |
|---------------|---------------------|
| Purple/Blue | Gold/Orange |
| Green | Red/Crimson |
| Red | Teal/Cyan |
| Brown/Earth | Sky Blue |
| Black | White, Gold, or Bright Red |

---

## 7.7 SHADOW AND HIGHLIGHT PAINTING IN TEXTURE

Classic RO textures carry **baked lighting information**. The light source is assumed from the top-left.

```
BAKED LIGHT DIRECTION: Top-left (45° vertical, 315° horizontal)

SURFACE FACING TOP-LEFT:   Add +20–30 value → near-white highlight
SURFACE FACING FORWARD:    Use base color
SURFACE FACING DOWN:       Subtract 30–40 value → shadow
SURFACE FACING AWAY:       Subtract 50–60 value → deep shadow

EXAMPLE for cloth with base color RGB(80, 60, 160) [purple]:
  Highlight:   RGB(110, 90, 190)
  Base:        RGB(80, 60, 160)
  Mid-shadow:  RGB(55, 40, 120)
  Deep shadow: RGB(30, 20, 80)
```

---

## 7.8 HEX COLOR REFERENCE LIBRARY

### Common RO Material Colors

```
METALS:
  Iron (raw):         #6B6B6B  — dull grey
  Iron (polished):    #A8A8A8  — bright grey
  Gold (deep):        #C8A951  — classic RO gold
  Gold (bright):      #FFD700  — magical glow gold
  Bronze:             #8B6914  — worn brass/bronze

WOODS:
  Oak (light):        #9C6B3C  — light wooden weapon
  Oak (dark):         #5C3A1E  — dark wood staff
  Bamboo:             #7A8C3C  — green-grey bamboo

CLOTHS:
  Knight Royal Blue:  #1A3A6B
  Mage Purple:        #4A2060
  Robe Dark Purple:   #2D0040
  Acolyte White:      #F5F0E8
  Thief Black:        #1A1A1A
  Archer Green:       #2D5A1B
  Merchant Brown:     #5C3A1E

LEATHERS:
  Light Leather:      #9C6B3C
  Medium Leather:     #7A4A28
  Dark Leather:       #5C3014

MAGICAL EFFECTS:
  Holy White:         #FFFEF0  — aura
  Holy Gold:          #FFE050  — divine
  Fire Orange:        #FF6600
  Ice Blue:           #80C0FF
  Poison Green:       #40CC40
  Shadow Black:       #1A001A
  Thunder Yellow:     #FFFF40
```

---

## 7.9 AMBIENT LIGHT COLOR SETTINGS

Classic RO uses a warm ambient:

```
STANDARD OUTDOOR:
  Ambient: RGB(102, 97, 90)   — warm grey
  Light:   RGB(255, 243, 217) — warm sunlight

DUNGEON/INDOOR:
  Ambient: RGB(50, 45, 55)    — cool dark
  Light:   RGB(180, 160, 220) — cool purple torchlight

MAGICAL ZONE:
  Ambient: RGB(60, 50, 80)    — arcane dark
  Light:   RGB(200, 180, 255) — arcane purple-white

UNDERWATER:
  Ambient: RGB(40, 60, 80)    — deep blue
  Light:   RGB(100, 160, 220) — refracted blue
```

Always ensure **ambient is never pure black** (0,0,0) and never pure white.  
Pure black ambient makes the shadow side invisible. Pure white makes lighting flat.
