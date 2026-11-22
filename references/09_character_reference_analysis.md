# 09 — CHARACTER REFERENCE ANALYSIS
## Systematic Reference Analysis Workflow

> **Purpose:** Define systematic workflow for analyzing character references before modeling or recreation.  
> **Related:** [02_character_proportion_system.md](02_character_proportion_system.md), [../modeling/10_low_poly_modeling.md](../modeling/10_low_poly_modeling.md)

---

## 9.1 ANALYSIS PHILOSOPHY

Never begin modeling without a completed reference analysis.  
A 10-minute analysis prevents hours of rebuilding.

The analysis produces a **CHARACTER SPECIFICATION DOCUMENT** that serves as the blueprint for all subsequent work.

---

## 9.2 REFERENCE QUALITY ASSESSMENT

Before beginning analysis, assess reference quality:

```
TIER 1 — IDEAL REFERENCE:
  ✓ High resolution (> 512px character height)
  ✓ Front, side, and back views available
  ✓ Clean background (no overlapping elements)
  ✓ Neutral pose (T-pose or standard idle)
  ✓ All equipment visible
  → Proceed directly to full analysis

TIER 2 — ACCEPTABLE REFERENCE:
  ✓ Moderate resolution (200–512px character height)
  ✓ Front view available, side implied
  ~ Some background overlap
  ~ Standard game pose (walking/idle frame)
  → Proceed with proportional assumptions noted

TIER 3 — LIMITED REFERENCE:
  ~ Low resolution (< 200px character height)
  ~ Single angle only
  ~ Dynamic action pose (harder to measure proportions)
  ~ Partial equipment visibility
  → Note all assumptions explicitly; flag for review

TIER 4 — INSUFFICIENT:
  ✗ < 100px character height
  ✗ Severely distorted or perspective-warped
  ✗ Character partially obscured
  → Request better reference or use class archetype defaults from ../references/04_class_and_character_archetypes.md
```

---

## 9.3 STEP-BY-STEP ANALYSIS WORKFLOW

### PHASE 1: PREPARATION

```
1. Load reference image into image editing tool
2. Adjust levels/curves for maximum detail visibility
3. Increase contrast if colors are washed out
4. Create a clean copy to work on (preserve original)
5. Scale image if needed: zoom in until character is at least 300px tall
```

### PHASE 2: HEAD UNIT MEASUREMENT

```
1. Find the HIGHEST point of the character (crown of head, excluding floating accessories)
   → If wearing a hat, the hat is PART of the head measurement
   
2. Find the CHIN LINE (not neck — chin)

3. Measure this vertical distance:
   → In pixels: use ruler tool or note pixel coordinates
   → This = 1 H (head unit)

4. Record: HEAD_UNIT = _____ pixels
```

### PHASE 3: PROPORTION GRID

```
1. Create a horizontal guide at:
   → Top of head (0 H)
   → Each 1H interval downward
   
2. Count how many H units reach the character's feet

3. Record: TOTAL_HEIGHT = _____ H

4. Identify where body landmarks fall on the grid:
   → Eye line: ___% from crown  (should be 50–55% for RO)
   → Shoulder: ___H from ground
   → Waist:    ___H from ground
   → Hips:     ___H from ground
   → Knee:     ___H from ground
   → Ankle:    ___H from ground
```

### PHASE 4: HORIZONTAL MEASUREMENTS

```
1. At shoulder level, measure total width in H units
   → Record: SHOULDER_WIDTH = ___ H
   
2. At hip level, measure width
   → Record: HIP_WIDTH = ___ H

3. Measure head width at widest point
   → Record: HEAD_WIDTH = ___ H

4. Measure hand size
   → Record: HAND_SIZE = ___ H

5. Measure foot length
   → Record: FOOT_LENGTH = ___ H
```

### PHASE 5: SILHOUETTE ANALYSIS

```
1. Duplicate the character layer
2. Fill it with solid black
3. Examine the pure silhouette

ASK:
  → What is the most prominent silhouette element?
     (Answer: hat, weapon, shoulder armor, cape, etc.)
  
  → Can I identify the CLASS from silhouette alone?
     (If no → the silhouette needs strengthening)
  
  → Are there any silhouette elements that obscure class identity?
  
4. Sketch a simplified 10-stroke silhouette (or describe it in text)
```

### PHASE 6: COLOR PALETTE EXTRACTION

```
METHOD: Manual sampling (K-means approximation)

1. Identify the 5–8 dominant colors:
   a. Sample the largest color area first (usually clothing)
   b. Sample each distinct material (skin, metal, cloth, leather)
   c. Sample accessories (belt, buckle, trim)
   d. Sample hair
   e. Sample eyes
   
2. For each color, record:
   → HEX value: #______
   → Approximate HSL: H:___, S:___%, L:___%
   → Material type: [cloth/leather/metal/skin/hair/etc.]
   → Location on character: [describe]

RESULT FORMAT:
  PALETTE:
    1. Primary clothing: #______ (60% area, cloth material)
    2. Secondary armor:  #______ (20% area, metal material)  
    3. Skin:             #______ (10% area, skin material)
    4. Hair:             #______ (5% area, hair material)
    5. Accent/trim:      #______ (3% area, metal material)
    6. Eyes:             #______ (1% area)
    7. Other:            #______ (1% area)
```

### PHASE 7: EQUIPMENT IDENTIFICATION

```
Systematically identify each equipment slot:

HEAD_TOP:   [ ] Present  Description: ___________
            [ ] Absent

HEAD_MID:   [ ] Present  Description: ___________
            [ ] Absent

HEAD_BOT:   [ ] Present  Description: ___________
            [ ] Absent

ARMOR:      Type: [light/medium/heavy/robe]
            Material: [cloth/leather/plate/mail]
            Color: #___ Description: ___________

GARMENT:    [ ] Cape  [ ] Wings  [ ] None
            Description: ___________

FOOTGEAR:   Height: [ankle/mid/thigh]
            Material: ___________

WEAPON:     Type: [sword/staff/bow/hammer/dagger/etc.]
            Held in: [right/left/both]
            Length estimate: ___ H

SHIELD:     [ ] Present  Type: ___________
            [ ] Absent
```

### PHASE 8: SPECIAL FEATURES

```
Identify any character-specific visual elements:

□ Visible aura or glow effects
□ Unusual skin color (non-standard)
□ Tattoo or body markings
□ Visible tail, horns, or non-human features
□ Unique posture (non-standard stance)
□ Additional props not in equipment slots
□ Class-unique visual signature
□ Character expression (default face expression)
```

---

## 9.4 CHARACTER SPECIFICATION DOCUMENT TEMPLATE

```
═══════════════════════════════════════════════════════════
ROCHAR CHARACTER SPECIFICATION
Generated: ___________
═══════════════════════════════════════════════════════════

IDENTITY
  Name:          ____________________
  Class:         ____________________
  Gender:        ____________________
  Special type:  [ ] Player  [ ] NPC  [ ] Monster  [ ] Custom

REFERENCE QUALITY: [ ] Tier 1  [ ] Tier 2  [ ] Tier 3  [ ] Tier 4
REFERENCE SOURCE:  ____________________

PATHWAY SELECTED: [ ] Sprite (A)  [ ] 3D Model (B)  [ ] Both

─────────────────────────────────────────────────────────
PROPORTIONS

HEAD UNIT = _____ px / _____ world units
H:B RATIO = 1 : _____

VERTICAL:
  Total height: _____ H
  Neck/Chest:   _____ H
  Torso:        _____ H
  Hips:         _____ H
  Thighs:       _____ H
  Shins:        _____ H
  Feet:         _____ H

HORIZONTAL:
  Shoulder width: _____ H
  Hip width:      _____ H
  Head width:     _____ H

FACIAL:
  Eye line:    _____% from crown
  Nose:        _____% from crown
  Mouth:       _____% from crown

─────────────────────────────────────────────────────────
COLOR PALETTE

  1. ____________: #______ — [material] — [location]
  2. ____________: #______ — [material] — [location]
  3. ____________: #______ — [material] — [location]
  4. ____________: #______ — [material] — [location]
  5. ____________: #______ — [material] — [location]

─────────────────────────────────────────────────────────
EQUIPMENT

  HEAD_TOP: ____________________
  HEAD_MID: ____________________
  HEAD_BOT: ____________________
  ARMOR:    ____________________ [light/medium/heavy/robe]
  GARMENT:  ____________________
  FOOTGEAR: ____________________
  WEAPON:   ____________________ (R/L/both)
  SHIELD:   ____________________
  ACC_L:    ____________________
  ACC_R:    ____________________

─────────────────────────────────────────────────────────
SILHOUETTE

  Dominant silhouette element: ____________________
  Class identifiable from silhouette? [ ] Yes  [ ] No
  Notes: ____________________________________________

─────────────────────────────────────────────────────────
SPECIAL FEATURES

  ____________________________________________
  ____________________________________________

─────────────────────────────────────────────────────────
ASSUMPTIONS MADE (TIER 2/3 reference)

  1. ____________________________________________
  2. ____________________________________________
  3. ____________________________________________

─────────────────────────────────────────────────────────
NEXT STEPS

  [ ] Generate proportion blueprint (02_character_proportion_system)
  [ ] Select modeling pathway
  [ ] Begin geometry construction (10_low_poly_modeling)
  [ ] Design material layout (07_color_and_material_language)
═══════════════════════════════════════════════════════════
```

---

## 9.5 MULTIPLE ANGLE RECONCILIATION

When you have both front and side references but they disagree:

```
RULE: Front reference takes PRIORITY for:
  → Width measurements
  → Facial proportions
  → Equipment symmetry

RULE: Side reference takes PRIORITY for:
  → Depth measurements
  → Silhouette profile
  → Equipment depth (shield, cape, weapon)

CONFLICT RESOLUTION:
  → If front says shoulder width = 1.5H but side looks narrower:
    Use front measurement; side view foreshortens width
  → If front says hair is medium-length but side shows long:
    Use side measurement; front view often hides back hair length
  → Document all resolved conflicts in ASSUMPTIONS section
```

---

## 9.6 REFERENCE ANALYSIS QUICK-RUN (TIER 2/3 SHORTCUT)

When reference quality is limited and time is constrained:

```
1. Identify class → load archetype defaults from 04_class_and_character_archetypes.md
2. Sample 3 key colors: dominant clothing, skin, hair
3. Identify weapon type only (most class-defining element)
4. Use standard 1:4 H:B ratio
5. Document all assumptions explicitly
6. Flag for visual review after first mesh block-in
```

This produces a 70% accurate specification in 3 minutes rather than a 95% accurate specification in 20 minutes. Acceptable for rapid prototyping.
