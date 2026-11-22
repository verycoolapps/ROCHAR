# 02 — CHARACTER PROPORTION SYSTEM
## Measurement, Analysis, and Silhouette Construction Framework

> **Purpose:** Define RO-specific proportion analysis, measurement methodology, and silhouette construction.  
> **Related:** [03_ro_character_anatomy.md](03_ro_character_anatomy.md), [04_class_and_character_archetypes.md](04_class_and_character_archetypes.md), [../modeling/10_low_poly_modeling.md](../modeling/10_low_poly_modeling.md)

---

## 2.1 THE RO PROPORTION SCALE

Ragnarok Online characters use a **chibi-stylized proportion system** that deviates significantly from realistic human anatomy.

### Head-to-Body Ratio by Character Type

| Character Type | Head Units | Total Height | H:B Ratio |
|---------------|-----------|-------------|-----------|
| Novice (pre-job) | 1 unit | 3.0–3.5 units | 1:2.5 |
| Standard playable character | 1 unit | 4.0–4.5 units | 1:4 |
| Transcendent / Third class | 1 unit | 4.5–5.0 units | 1:4.5 |
| Child-type NPC | 1 unit | 3.0 units | 1:3 |
| Monster (humanoid) | Variable | Variable | Per design |

**Standard production reference:** Use **1:4** as the default for all playable characters unless reference data dictates otherwise.

---

## 2.2 MEASUREMENT METHODOLOGY

### Unit System
Define **1 H = head height** (crown to chin, not including hair volume).

```
HEAD HEIGHT (H):
  Crown to chin = 1.0 H

BODY SEGMENTS (top to ground, standard 1:4 character):
  Neck + upper chest:    0.3 H
  Mid chest to waist:    0.5 H
  Waist to hip joint:    0.3 H
  Hip joint to knee:     0.7 H
  Knee to ankle:         0.6 H
  Ankle to ground:       0.2 H
  ─────────────────────────────
  TOTAL BODY (sans head): ~2.6 H

TOTAL CHARACTER HEIGHT: 1.0 (head) + 2.6 (body) ≈ 3.6–4.5 H
```

### Horizontal Proportions

```
SHOULDER WIDTH:    1.4–1.6 H wide  (measured across outer shoulder)
HIP WIDTH:         1.0–1.2 H wide
HEAD WIDTH:        0.75–0.85 H wide
HAND SIZE:         0.3 H  (intentionally large for readability)
FOOT LENGTH:       0.4 H  (stubby, reads clearly at distance)
FOOT WIDTH:        0.15 H
```

---

## 2.3 PROPORTION GRID SYSTEM

For blocking out character silhouette before modeling:

```
+──────────────────+
│      HEAD        │  Row 1: Head (1.0 H)
+──────────────────+
│  NECK + CHEST    │  Row 2: Upper torso (0.5 H)
+──────────────────+
│  LOWER TORSO     │  Row 3: Lower torso + hips (0.5 H)
+──────────────────+
│  UPPER LEGS      │  Row 4: Thighs (0.7 H)
+──────────────────+
│  LOWER LEGS      │  Row 5: Shins (0.6 H)
+──────────────────+
│  FEET            │  Row 6: Feet (0.2–0.3 H)
+──────────────────+
```

Each row is a modeling priority zone. Build them in order: head → torso → arms → legs → feet.

---

## 2.4 GENDER PROPORTION DIFFERENCES

### Male Characters
- Shoulder width: **1.6 H** (wider relative to hips)
- Hip width: **1.1 H**
- Neck diameter: **0.2 H** (thicker)
- Jaw: more angular, slightly squared chin
- Arms: slightly more muscular cylinders
- Overall silhouette: **T-shape** (shoulders broader than hips)

### Female Characters
- Shoulder width: **1.3 H** (narrower)
- Hip width: **1.2 H** (at or slightly wider than shoulders)
- Neck diameter: **0.15 H** (slimmer)
- Face: softer features, eyes proportionally larger
- Arms: slimmer cylinders
- Overall silhouette: **hourglass or straight** (not exaggerated)

---

## 2.5 HEAD PROPORTION DETAIL

The head is the **most recognizable part** of RO characters. These sub-proportions define identity.

```
HEAD (front view):
  Width : Height = 1.0 : 1.1 (slightly taller than wide)

EYE LINE POSITION:
  Located at 50–55% from crown (classic anime/chibi positioning)
  (Realistic humans: eyes at ~50% but skull above makes it appear higher)

EYE DIMENSIONS:
  Each eye width: ~25% of head width
  Eye height:     ~18–22% of face height
  Eye separation: ~30% of head width between inner eye edges

NOSE:
  Minimal geometric presence
  Simple dot, small curved line, or basic bridge indication
  Located at: ~68–72% from crown

MOUTH:
  Small, centered
  Located at: ~78–82% from crown
  Width: ~40% of head width

CHIN:
  Soft, rounded (female/neutral) or subtly squared (strong male)
  Chin-to-crown: 100% of face height

EAR PLACEMENT:
  Aligns with eye line
  Bottom of ear aligns with nose line
  Width: ~8% of head width
  Often hidden by hairstyle
```

---

## 2.6 SILHOUETTE PRIORITY RULES

RO characters must read clearly at small viewport sizes. Priority order for silhouette:

1. **CLASS-DEFINING ELEMENT** must dominate the read
   - Knight/Swordsman: broad shoulders + armor bulk
   - Wizard/Mage: tall/wide distinctive hat + staff
   - Hunter/Archer: bow shape visible at distance, light build
   - Thief/Assassin: lean build + dark asymmetric hat
   - Priest/Acolyte: staff + flowing robe silhouette
   - Blacksmith/Merchant: hammer + stocky build

2. **HAIR SILHOUETTE** must be distinct and not merge with the body read

3. **EQUIPMENT SILHOUETTE** enhances but does not obscure primary body read

4. **BODY MASS** anchors the character on the ground plane visually

---

## 2.7 PROPORTION MEASUREMENT FROM REFERENCE

When analyzing a reference screenshot:

```
STEP 1: ISOLATE CHARACTER
  → Crop to character bounding box
  → Remove or neutralize background
  → Normalize brightness/contrast if too dark

STEP 2: FIND HEAD UNIT
  → Top of head = highest point of hair/hat (not accessory ornament)
  → Bottom of head = chin line (not neck)
  → This vertical distance = 1 H

STEP 3: APPLY GRID OVERLAY
  → Count how many H units the full body spans (ground to top of head)
  → Record measured H:B ratio

STEP 4: EXTRACT LANDMARK POSITIONS
  → Eye line height (% from crown)
  → Shoulder height (H units from ground)
  → Waist height (H units from ground)
  → Hip joint height (H units from ground)
  → Knee height (H units from ground)
  → Ankle height (H units from ground)

STEP 5: DOCUMENT EQUIPMENT EFFECTS
  → Equipment that inflates shoulder width
  → Headgear that extends apparent head height
  → Footwear that adds height
  → Armor that obscures true body proportions
```

---

## 2.8 PROPORTION DEVIATION ALLOWANCES

| Feature | Standard | Allowed Deviation | Notes |
|---------|----------|-------------------|-------|
| Head H:B ratio | 1:4 | ±0.5 H | Smaller = more serious; larger = more youthful |
| Eye size | 25% head width | ±10% | Larger = more youthful; smaller = stern |
| Hand size | 0.3 H | +0.1 / -0.05 | Combat classes slightly larger hands |
| Foot length | 0.4 H | ±0.1 | Boots may extend apparent foot size |
| Shoulder width | 1.5 H | ±0.2 | Gender-dependent, class-dependent |

Deviations outside these ranges will appear unintentionally distorted unless it is a monster or NPC design with deliberate extremes.

---

## 2.9 PROPORTION BLUEPRINT TEMPLATE

**Always generate this document before beginning any character modeling session.**

```
═══════════════════════════════════════════════════
CHARACTER PROPORTION BLUEPRINT
═══════════════════════════════════════════════════
Character Name:     ________________________
Character Class:    ________________________
Gender:             ________________________
Reference Source:   ________________________
Reference Quality:  [ ] Excellent  [ ] Good  [ ] Limited
Date:               ________________________

HEAD UNIT (H):
  Measured in pixels: ______px
  Equivalent world units: ______ m

HEAD-TO-BODY RATIO:   1 : ______
Pathway selected:     [ ] Sprite  [ ] 3D Model

VERTICAL SEGMENTS:
  Head:              1.0 H
  Neck/Upper Chest:  _____ H
  Torso:             _____ H
  Hips:              _____ H
  Upper Legs:        _____ H
  Lower Legs:        _____ H
  Feet:              _____ H
  ─────────────────────────
  TOTAL:             _____ H

HORIZONTAL MEASUREMENTS:
  Shoulder width:    _____ H
  Hip width:         _____ H
  Head width:        _____ H

FACIAL LANDMARKS:
  Eye line:          _____% from crown
  Nose:              _____% from crown
  Mouth:             _____% from crown

EQUIPMENT THAT AFFECTS PROPORTIONS:
  __________________________________________

CLASS-SPECIFIC DEVIATIONS:
  __________________________________________

ANIMATION CONSIDERATIONS:
  __________________________________________
═══════════════════════════════════════════════════
```

---

## 2.10 PROPORTION SCALING FOR WORLD UNITS

When building in Blender for export to Three.js or game engines:

```
RECOMMENDED SCALE:
  1 H (head unit) = 0.25 meters in Blender world units

Resulting character heights:
  Novice (3.5H):   0.875 m
  Standard (4.0H): 1.00 m
  Advanced (4.5H): 1.125 m

This ensures the character height is realistic at ~1 meter,
appropriate for RO's compact world scale.

Blender units → Meters: 1 BU = 1 m
Character stands on the world XY plane at Z=0.
Head is at approximately Z=1.0 (standard).
```
