# 31 — CHARACTER PIPELINE ORCHESTRATION
## End-to-End Workflow from Reference to Production Asset

> **Purpose:** Orchestrate the complete ROCHAR pipeline from reference analysis to deployed character.  
> **Related:** [../references/09_character_reference_analysis.md](../references/09_character_reference_analysis.md), [32_quality_assurance.md](32_quality_assurance.md)

---

## 31.1 PATHWAY DECISION TREE

```
START: Character reconstruction request
│
├── Q1: Do you have the original game files (RSM/SPR)?
│   ├── YES → Q2
│   └── NO  → Original creation / fan art only
│
├── Q2: Is the character used in a 3D scene or 2D scene?
│   ├── 3D environment (perspective view, rotation) → PATHWAY B (3D Model)
│   └── 2D/isometric with fixed camera → PATHWAY A (Sprite-Based)
│
├── Q3 (Pathway A): Do you have high-quality reference sprites?
│   ├── YES → Sprite Reconstruction workflow (Module 16)
│   └── NO  → Must recreate manually with art tools
│
└── Q4 (Pathway B): Target polygon budget?
    ├── Low (< 300 polys) → Minimal box model
    ├── Medium (300–800)  → Standard ROCHAR model
    └── High (800–2000)   → Detailed model with normal maps
```

---

## 31.2 PATHWAY A — SPRITE-BASED PIPELINE

```
STAGE 1: REFERENCE ANALYSIS (Module 09)
  Tasks:
    □ Collect reference sprites (all 8 directions × all animations)
    □ Measure proportions (H units from Module 02)
    □ Document color palette (hex values, Module 07)
    □ Map animation states to ACT action indices (Module 29)
  Output: Character specification document
  Duration: 1–2 hours

STAGE 2: SPRITE ISOLATION (Module 16)
  Tasks:
    □ Run magenta_to_alpha() on all source frames
    □ Verify edge quality at 400% zoom
    □ Run validate_sprite_consistency() across frames
    □ Normalize color palette
  Output: Clean PNG frames with alpha channel
  Duration: 30–60 minutes

STAGE 3: ATLAS CONSTRUCTION (Module 17)
  Tasks:
    □ Determine atlas size (Module 17.5)
    □ Pack frames using Grid or Packed layout
    □ Add 2px padding between frames
    □ Generate metadata JSON
    □ Verify UV calculations with getFrameUV()
  Output: atlas.png + metadata.json
  Duration: 1–2 hours

STAGE 4: ANIMATION SYSTEM (Module 19)
  Tasks:
    □ Define ANIMATION_STATES for this character
    □ Map ACT action indices to state definitions
    □ Implement SpriteAnimationController
    □ Wire frame events (hit_connect, spell_fire, footstep)
    □ Test all state transitions
  Output: Configured animation controller
  Duration: 1–2 hours

STAGE 5: BILLBOARD INTEGRATION (Modules 18, 22)
  Tasks:
    □ Set up billboard mesh with correct world scale
    □ Configure GLSL billboard shader uniforms
    □ Wire update loop (frame UV, direction, scale)
    □ Test all 8 directions
    □ Add shadow sprite
  Output: Functional billboard in Three.js scene
  Duration: 1–2 hours

STAGE 6: QA (Module 32)
  Tasks:
    □ Run QA scoring rubric
    □ Score ≥ 90/100 required to pass
    □ Fix any failing categories
  Duration: 30–60 minutes

TOTAL PATHWAY A: 5–9 hours per character
```

---

## 31.3 PATHWAY B — 3D MODEL PIPELINE

```
STAGE 1: REFERENCE ANALYSIS (Module 09)
  Same as Pathway A.
  Duration: 1–2 hours

STAGE 2: LOW-POLY MODELING (Modules 10–11)
  Tasks:
    □ Build base mesh using box modeling
    □ Check polygon budget (Module 10 budget tables)
    □ Fix topology: no N-gons, no non-manifold
    □ Validate with Blender Statistics overlay
  Output: Watertight base mesh
  Duration: 2–4 hours

STAGE 3: UV MAPPING (Module 12)
  Tasks:
    □ Smart UV Project with 85° angle limit
    □ Organize UV islands by body part
    □ Check pixel density ≥ 10 px/m
    □ Create UV2 for lightmap (if needed)
    □ Verify no UV overlap in UV2
  Output: UV-mapped mesh ready for texturing
  Duration: 1–2 hours

STAGE 4: MODULAR EQUIPMENT (Module 13)
  Tasks:
    □ Set up attachment point socket bones
    □ Build equipment meshes
    □ Create JSON equipment manifest
    □ Test equip/unequip via ModularCharacter class
  Output: Equipped character with swappable gear
  Duration: 2–4 hours

STAGE 5: RIGGING (Module 26)
  Tasks:
    □ Run create_ro_armature() Python script
    □ Apply armature to mesh (Ctrl+P → Auto Weights)
    □ Paint vertex weight corrections
    □ Set up IK for legs
    □ Test deformation in all directions
  Output: Fully rigged character
  Duration: 2–4 hours

STAGE 6: ANIMATION (Module 27)
  Tasks:
    □ Keyframe all 11 required animation states
    □ Set correct FPS per action
    □ Add animation events
    □ Organize NLA editor
    □ Run validate_animation_timing()
  Output: All animations authored and validated
  Duration: 4–8 hours

STAGE 7: EXPORT (Module 15)
  Tasks:
    □ Run pre-export checklist
    □ Export GLB with correct settings
    □ Validate with gltf-validator
    □ Load in Three.js for visual check
  Output: character.glb
  Duration: 30–60 minutes

STAGE 8: INTEGRATION (Module 30)
  Tasks:
    □ Load in ROCharacterScene
    □ Configure textures (colorspace, filtering)
    □ Set up render order
    □ Wire animation controller
  Output: Character live in scene
  Duration: 1–2 hours

STAGE 9: QA (Module 32)
  Duration: 1–2 hours

TOTAL PATHWAY B: 15–28 hours per character
```

---

## 31.4 PIPELINE STATE MACHINE (JavaScript)

```javascript
const PIPELINE_STATES = {
    IDLE:              'idle',
    REFERENCE_ANALYSIS:'reference_analysis',
    SPRITE_ISOLATION:  'sprite_isolation',
    ATLAS_BUILD:       'atlas_build',
    ANIMATION_SYSTEM:  'animation_system',
    BILLBOARD_SETUP:   'billboard_setup',
    QA_REVIEW:         'qa_review',
    COMPLETE:          'complete',
    FAILED:            'failed'
};

class CharacterPipeline {
    constructor(characterId, pathway = 'A') {
        this.characterId = characterId;
        this.pathway     = pathway;
        this.state       = PIPELINE_STATES.IDLE;
        this.stages      = this._buildStageList(pathway);
        this.currentStage = 0;
        this.artifacts   = {};  // Outputs accumulated across stages
        this.errors      = [];
    }
    
    _buildStageList(pathway) {
        if (pathway === 'A') {
            return [
                'reference_analysis',
                'sprite_isolation',
                'atlas_build',
                'animation_system',
                'billboard_setup',
                'qa_review'
            ];
        }
        return [
            'reference_analysis',
            'low_poly_modeling',
            'uv_mapping',
            'modular_equipment',
            'rigging',
            'animation',
            'export',
            'integration',
            'qa_review'
        ];
    }
    
    advance(artifact) {
        // Store output artifact
        const stageName = this.stages[this.currentStage];
        this.artifacts[stageName] = artifact;
        
        // Move to next stage
        this.currentStage++;
        
        if (this.currentStage >= this.stages.length) {
            this.state = PIPELINE_STATES.COMPLETE;
            console.log(`[ROCHAR] Pipeline COMPLETE for '${this.characterId}'`);
        } else {
            this.state = this.stages[this.currentStage];
            console.log(`[ROCHAR] '${this.characterId}' → ${this.state}`);
        }
    }
    
    fail(error) {
        this.errors.push({
            stage: this.stages[this.currentStage],
            error: error
        });
        this.state = PIPELINE_STATES.FAILED;
        console.error(`[ROCHAR] Pipeline FAILED at '${this.stages[this.currentStage]}': ${error}`);
    }
    
    getProgress() {
        return {
            percent:  Math.round((this.currentStage / this.stages.length) * 100),
            current:  this.stages[this.currentStage],
            remaining: this.stages.length - this.currentStage,
            complete:  this.state === PIPELINE_STATES.COMPLETE
        };
    }
}
```

---

## 31.5 BATCH PROCESSING MULTIPLE CHARACTERS

```javascript
async function processCharacterBatch(characterList, pathway = 'A') {
    const pipelines = characterList.map(char => ({
        id:       char.id,
        pipeline: new CharacterPipeline(char.id, pathway),
        config:   char
    }));
    
    console.log(`[ROCHAR] Starting batch: ${pipelines.length} characters, Pathway ${pathway}`);
    
    // Process in parallel (up to 3 at once to avoid overwhelming memory)
    const CONCURRENCY = 3;
    
    for (let i = 0; i < pipelines.length; i += CONCURRENCY) {
        const batch = pipelines.slice(i, i + CONCURRENCY);
        
        await Promise.allSettled(
            batch.map(item => runPipeline(item.pipeline, item.config))
        );
    }
    
    // Report results
    const complete = pipelines.filter(p => p.pipeline.state === 'complete');
    const failed   = pipelines.filter(p => p.pipeline.state === 'failed');
    
    console.log(`[ROCHAR] Batch complete: ${complete.length} OK, ${failed.length} failed`);
    
    if (failed.length > 0) {
        failed.forEach(f => {
            console.error(`  FAILED: ${f.id}`, f.pipeline.errors);
        });
    }
    
    return { complete, failed };
}
```

---

## 31.6 PIPELINE TIME ESTIMATES

```
CHARACTER TYPE     | PATHWAY A | PATHWAY B
───────────────────┼───────────┼──────────
Simple NPC         |   3–5h    |  12–18h
Player character   |   5–9h    |  18–28h
Monster (small)    |   2–4h    |   8–14h
Monster (large)    |   4–8h    |  15–24h
Boss character     |   8–12h   |  24–40h

SPEEDUP FACTORS:
  Familiar class (same as previous char): -20% time
  Reference quality HIGH:                 -15% time
  Using existing modular parts:           -30% time for Pathway B
  First time (learning):                  +50% time
```
