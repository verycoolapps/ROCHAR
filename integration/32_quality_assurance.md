# 32 — QUALITY ASSURANCE
## 100-Point Scoring Rubric and Automated Validation

> **Purpose:** Systematic quality gate ensuring every ROCHAR output meets production standards.  
> **Minimum Passing Score: 90/100**  
> **Related:** [../integration/31_character_pipeline_orchestration.md](31_character_pipeline_orchestration.md), [../33_troubleshooting_and_diagnostics.md](../33_troubleshooting_and_diagnostics.md)

---

## 32.1 SCORING CATEGORIES OVERVIEW

```
CATEGORY                    | MAX | WEIGHT
────────────────────────────┼─────┼────────
1. Visual Fidelity           |  10 | High
2. Polygon Efficiency        |  10 | Medium
3. UV Quality                |  10 | Medium
4. Shader Correctness        |  10 | High
5. Animation Completeness    |  10 | High
6. Direction Coverage        |  10 | High
7. Performance               |  10 | Medium
8. Format Compliance         |  10 | Medium
9. Pipeline Documentation    |  10 | Low
10. RO Authenticity          |  10 | High
────────────────────────────┼─────┼────────
TOTAL                        | 100 |
MINIMUM PASS                 |  90 |
```

---

## 32.2 CATEGORY 1 — VISUAL FIDELITY (10 pts)

```
CRITERION: Does the character look like the original reference?

10 pts — PERFECT:
  → Proportion H:B ratio within ±2% of reference
  → Color palette matches reference within ΔE < 3 (perceptible)
  → Silhouette recognizable from all 8 directions
  → Equipment shape and position accurate

8 pts — GOOD:
  → Proportions within ±5%
  → Colors close but minor shifts acceptable
  → Silhouette clear from primary directions (S, SW, SE)

5 pts — ACCEPTABLE:
  → Character recognizable as the class/type
  → Proportions visibly off but not extreme
  → Color palette generally correct

0 pts — FAIL:
  → Character unrecognizable
  → Wrong proportions (non-chibi, wrong head-to-body ratio)
  → Wildly wrong colors

COMMON FAILURES:
  → Head too small (forgot 1:3 head-to-body RO ratio)
  → Wrong skin tone (#E8B88A is the standard warm tan)
  → Equipment missing or positioned incorrectly
```

---

## 32.3 CATEGORY 2 — POLYGON EFFICIENCY (10 pts)

```
CRITERION: Is the polygon count within budget and topology clean?

10 pts — PERFECT:
  → Polygon count within budget (see Module 10)
  → Zero N-gons
  → Zero non-manifold edges
  → Topology follows natural muscle/cloth flow

8 pts — GOOD:
  → Up to 20% over budget but acceptable
  → ≤ 2 non-critical N-gons (removed in export)

5 pts — ACCEPTABLE:
  → 20–50% over budget
  → Some N-gons present but no major deformation issues

0 pts — FAIL:
  → More than 2× polygon budget
  → N-gons causing shading artifacts
  → Non-manifold geometry causing export errors

POLYGON BUDGETS (from Module 10):
  Sprite-based character: 4 triangles (2 quad billboard)
  Simple NPC: 300–500 triangles
  Player character: 500–800 triangles
  Boss monster: 800–1500 triangles
```

---

## 32.4 CATEGORY 3 — UV QUALITY (10 pts)

```
CRITERION: Are UV coordinates clean, efficient, and artifact-free?

10 pts — PERFECT:
  → Pixel density ≥ 10 px/m throughout
  → No UV island outside 0–1 range
  → Atlas coverage ≥ 80% (little wasted space)
  → Zero texture seams visible in render
  → Lightmap UV2: no overlapping islands

8 pts — GOOD:
  → Minor seams in hidden areas only
  → Atlas coverage 60–80%
  → Pixel density slight drop in some areas

5 pts — ACCEPTABLE:
  → Visible seams on secondary faces (back of torso)
  → Atlas coverage 40–60%
  → Pixel density inconsistency

0 pts — FAIL:
  → Seams visible in primary view angles
  → UV islands overlapping (causes texture corruption)
  → Atlas coverage < 40% (wasteful)

MEASUREMENT:
  Run Module 12 pixel density formula:
  density = (texture_size / island_size_px) × (world_size_m / 1.0)
  Target: ≥ 10 px/m
```

---

## 32.5 CATEGORY 4 — SHADER CORRECTNESS (10 pts)

```
CRITERION: Does the shader produce correct visual output?

10 pts — PERFECT:
  → RO lighting formula correctly applied
  → Vertex colors multiply correctly
  → Alpha test threshold at 0.5 (no fringe)
  → No z-fighting artifacts
  → Lightmap UV2 (if applicable) blending correct

8 pts — GOOD:
  → Minor lighting difference from reference
  → Slight fringe at alpha edges

5 pts — ACCEPTABLE:
  → Lighting clearly different but functional
  → Some z-fighting at non-critical locations

0 pts — FAIL:
  → Shader compilation error
  → Character completely black or white
  → Alpha fringe visible at all edges
  → Z-fighting destroying readability

VERIFY: Run visual comparison against reference in same lighting conditions.
```

---

## 32.6 CATEGORY 5 — ANIMATION COMPLETENESS (10 pts)

```
CRITERION: Are all required animation states present and correct?

10 pts — PERFECT:
  → All 11 states present (idle, walk, run, attack_1h, attack_2h, 
    cast, skill, hurt, dead, sit, pickup)
  → Each state has correct frame count and FPS
  → All animation events fire at correct frames
  → No foot skating (foot drift < 0.005m/frame)
  → Smooth transitions between all states

8 pts — GOOD:
  → 9–10 of 11 states present
  → Minor timing differences (±1 frame)

5 pts — ACCEPTABLE:
  → 7–8 states present
  → Some events missing

0 pts — FAIL:
  → Fewer than 7 states
  → Dead animation loops (should not loop)
  → Hit event missing from attack animation
```

---

## 32.7 CATEGORY 6 — DIRECTION COVERAGE (10 pts)

```
CRITERION: Are all 8 facing directions available?

10 pts — PERFECT:
  → All 8 directions present (S, SW, W, NW, N, NE, E, SE)
  → OR: 5 unique directions + correct mirroring metadata
  → Direction quantization smooth at all camera angles
  → Mirrored directions flip correctly without artifacts

8 pts — GOOD:
  → 7 of 8 directions, or 5 unique + partial mirroring

5 pts — ACCEPTABLE:
  → 4 directions (front/back/left/right only)

0 pts — FAIL:
  → Fewer than 4 directions
  → Direction mapping inverted (character faces wrong way)
```

---

## 32.8 CATEGORY 7 — PERFORMANCE (10 pts)

```
CRITERION: Does the character render within performance budget?

10 pts — PERFECT:
  → Single draw call for billboard characters
  → Texture size ≤ 512×512 for standard characters
  → Atlas shared with other characters
  → Frame rate maintains 60fps with 50 characters in scene

8 pts — GOOD:
  → ≤ 2 draw calls per character
  → Texture ≤ 1024×1024

5 pts — ACCEPTABLE:
  → 3–4 draw calls
  → Texture up to 2048×2048

0 pts — FAIL:
  → > 5 draw calls per character
  → Texture > 2048×2048
  → Causes frame drops below 30fps
```

---

## 32.9 CATEGORY 8 — FORMAT COMPLIANCE (10 pts)

```
CRITERION: Is the exported file correctly formatted?

10 pts — PERFECT:
  → GLB passes gltf-validator with 0 errors, 0 warnings
  → Y-axis is up
  → Foot at Y=0
  → Coordinate system matches Three.js (Y-up, right-hand)
  → All textures embedded in GLB

8 pts — GOOD:
  → 0 errors, ≤ 3 warnings (cosmetic)

5 pts — ACCEPTABLE:
  → 1–2 warnings about mesh quality
  → Character may need minor positioning fix

0 pts — FAIL:
  → Validation errors
  → Character sideways or upside-down
  → Missing textures in GLB
```

---

## 32.10 CATEGORY 9 — PIPELINE DOCUMENTATION (10 pts)

```
CRITERION: Is the character properly documented for future maintenance?

10 pts — PERFECT:
  → metadata.json complete and valid
  → equipment_manifest.json present
  → Color palette documented (hex values)
  → Source files organized in standard directory
  → Reference images retained

8 pts — GOOD:
  → Metadata present, minor omissions

5 pts — Acceptable:
  → Basic documentation present

0 pts — FAIL:
  → No documentation whatsoever
```

---

## 32.11 CATEGORY 10 — RO AUTHENTICITY (10 pts)

```
CRITERION: Does the character pass the nostalgia test?

THE NOSTALGIA TEST:
  Show the character to someone familiar with classic RO.
  Can they identify: (1) the class, (2) the character, (3) the zone/context?
  
10 pts — PERFECT:
  → Instant recognition by any RO player
  → "Looks exactly like the game" response
  → Visual DNA (chibi proportion, flat diffuse, baked lighting) all correct

8 pts — GOOD:
  → Recognized as RO-style character
  → Minor deviations from original acceptable

5 pts — ACCEPTABLE:
  → Recognizable as the class/type
  → Clearly RO-inspired

0 pts — FAIL:
  → Generic character with no RO visual identity
  → Wrong visual language (too realistic, wrong proportions)
```

---

## 32.12 AUTOMATED QA SCORER (JavaScript)

```javascript
class ROCharQAScorer {
    constructor() {
        this.scores = {};
        this.total  = 0;
    }
    
    score(category, points, notes = '') {
        if (points < 0 || points > 10) {
            throw new Error(`Score must be 0–10, got ${points}`);
        }
        this.scores[category] = { points, notes };
        this.total = Object.values(this.scores).reduce((s, v) => s + v.points, 0);
    }
    
    isPassing() {
        return this.total >= 90;
    }
    
    report() {
        const lines = [
            '═══════════════════════════════════',
            '  ROCHAR QA SCORECARD',
            '═══════════════════════════════════',
        ];
        
        const categories = [
            'visual_fidelity', 'polygon_efficiency', 'uv_quality',
            'shader_correctness', 'animation_completeness', 'direction_coverage',
            'performance', 'format_compliance', 'documentation', 'ro_authenticity'
        ];
        
        for (const cat of categories) {
            const entry = this.scores[cat];
            const pts   = entry?.points ?? 0;
            const bar   = '█'.repeat(pts) + '░'.repeat(10 - pts);
            const name  = cat.replace(/_/g, ' ').padEnd(26);
            lines.push(`  ${name} ${bar} ${pts}/10`);
            if (entry?.notes) {
                lines.push(`    → ${entry.notes}`);
            }
        }
        
        lines.push('───────────────────────────────────');
        lines.push(`  TOTAL: ${this.total}/100  ${this.isPassing() ? '✓ PASS' : '✗ FAIL (need 90+)'}`);
        lines.push('═══════════════════════════════════');
        
        return lines.join('\n');
    }
}

// Usage:
// const scorer = new ROCharQAScorer();
// scorer.score('visual_fidelity', 9, 'Minor color shift in head shadow');
// scorer.score('polygon_efficiency', 10);
// ... etc
// console.log(scorer.report());
```
