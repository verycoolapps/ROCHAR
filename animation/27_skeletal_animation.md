# 27 — SKELETAL ANIMATION
## NLA Animation, Keyframing, and Export for RO Characters

> **Purpose:** Author all required animation states, organize in NLA, and export correctly.  
> **Related:** [26_character_rigging.md](26_character_rigging.md), [28_animation_validation.md](28_animation_validation.md)

---

## 27.1 REQUIRED ANIMATION STATES

```
ANIMATION   | FRAMES | FPS | LOOP  | PRIORITY | NOTES
────────────┼────────┼─────┼───────┼──────────┼──────────────────────
idle        |   4    |  4  | YES   |    0     | Breathing/subtle sway
walk        |   8    |  8  | YES   |    1     | 8-frame gait cycle
run         |   8    | 12  | YES   |    2     | Faster version of walk
attack_1h   |   7    | 15  | NO    |    5     | One-hand swing
attack_2h   |   8    | 15  | NO    |    5     | Two-hand swing (wider arc)
cast        |  10    | 10  | NO    |    5     | Magic casting raise
skill       |   8    | 12  | NO    |    6     | Skill use animation
hurt        |   3    | 20  | NO    |    8     | Flinch when hit
dead        |   5    |  8  | NO    | 10 (hold)| Fall and hold last frame
sit         |   2    |  4  | YES   |    0     | Sitting resting state
pickup      |   4    |  8  | NO    |    3     | Pick up item
```

---

## 27.2 KEYFRAME PLACEMENT GUIDE

### IDLE (4 frames @ 4fps = 1 second loop)
```
Frame 1: Neutral pose (rest position)
Frame 2: Slight chest raise (breathing in), weight shift 1-2px
Frame 3: Peak inhale — shoulders slightly raised
Frame 4: Exhale return toward frame 1

RULE: Max displacement — 0.5–1px equivalents in world space.
      Barely perceptible but adds life. Too much = jittery.
```

### WALK (8 frames @ 8fps = 1 second loop)
```
Frame 1: Contact — left foot forward, right foot back
Frame 2: Down — left foot on ground, body lowering
Frame 3: Passing — weight on left, right foot passes
Frame 4: Up — left foot pushes, body rising
Frame 5: Contact — right foot forward, left foot back (mirror of 1)
Frame 6: Down — right foot on ground
Frame 7: Passing — weight on right, left foot passes
Frame 8: Up — right foot pushes, returning to frame 1 position

BODY BOB: Y-axis oscillates by ~0.02 units (slight up/down each step)
ARM SWING: Opposite to legs — left arm forward when right leg forward
HEAD: Slight forward tilt during down phases
```

### ATTACK_1H (7 frames @ 15fps ≈ 0.47s)
```
Frame 1: Anticipation — shoulder pulled back (windup)
Frame 2: Begin swing — arm starts forward motion
Frame 3: HIT FRAME — ← EVENT: hit_connect fires here
         Arm at full extension, weapon at target position
Frame 4: Follow-through — arm continues past center
Frame 5: Deceleration — arm slowing
Frame 6: Return arc — beginning to return to rest
Frame 7: Return rest — back to idle-like position

HIT EVENT: frame 3 must trigger hit detection in game logic.
           See metadata events[] array in sprite JSON.
```

### CAST (10 frames @ 10fps = 1 second)
```
Frame 1:  Neutral
Frame 2:  Arms beginning to raise
Frame 3:  Arms mid-raise
Frame 4:  Arms fully raised, slight head tilt up
Frames 5–8: Hold pose, slight trembling (keyframe every other frame)
Frame 9:  Arms thrust forward — SPELL EVENT fires here
Frame 10: Return arc begins (leads back to idle)
```

### DEAD (5 frames @ 8fps = 0.625s, holds last frame)
```
Frame 1: Initial reaction — back arches, arms fly up slightly
Frame 2: Gravity pulls — body begins to fall backward
Frame 3: Mid-fall
Frame 4: Almost on ground
Frame 5: Final rest — lying flat (hold this frame indefinitely)

NOTE: Dead animation NEVER loops. Character stays on frame 5.
      Respawn triggers a full state reset, not an animation.
```

---

## 27.3 F-CURVE INTERPOLATION SETTINGS

```
INTERPOLATION TYPE GUIDE:

CONSTANT (Step):
  → Use for: pixel art sprite changes, equipment swap moments
  → NO in-between frames generated
  → Each keyframe holds its value until the next keyframe
  → Perfect for sprite-sheet style "snap" transitions
  
BEZIER (Smooth):
  → Use for: 3D model animation (smooth arcs)
  → Default Blender interpolation
  → Adjust handles for overshooting (ease-in/ease-out)
  
LINEAR:
  → Use for: simple position moves without ease
  → Uniform speed, no acceleration
  
RECOMMENDATION:
  → 3D rigged characters: Bezier with auto-handles
  → Sprite sheet anim: Constant (matches 2D feel)
  → Position of foot on ground: Linear → no sliding

BLENDER SETTING:
  F-Curve Editor → Key menu → Interpolation Mode → Bezier/Constant/Linear
  Select all keys first (A), then set mode
```

---

## 27.4 NLA EDITOR ORGANIZATION

```python
import bpy

def organize_nla_editor(arm_obj):
    """
    Properly organize all actions in the NLA editor for clean GLB export.
    Each action becomes a separate animation in the exported GLB file.
    """
    
    if not arm_obj.animation_data:
        arm_obj.animation_data_create()
    
    # List all actions belonging to this character
    char_prefix = arm_obj.name.split("_Armature")[0]
    char_actions = [a for a in bpy.data.actions if a.name.startswith(char_prefix)]
    
    nla_data = arm_obj.animation_data
    
    # Push all actions to NLA strips
    for action in char_actions:
        # Create NLA track for this action
        track = nla_data.nla_tracks.new()
        track.name = action.name
        
        # Add strip with the action
        strip = track.strips.new(action.name, 
                                  start=action.frame_range[0], 
                                  action=action)
        strip.action_frame_start = action.frame_range[0]
        strip.action_frame_end   = action.frame_range[1]
        strip.use_auto_blend     = False  # No automatic blending
        
        print(f"[ROCHAR] NLA: Added '{action.name}' "
              f"frames {action.frame_range[0]}–{action.frame_range[1]}")
    
    print(f"[ROCHAR] NLA editor organized with {len(char_actions)} actions.")

def list_character_actions():
    """List all actions with their frame ranges and keyframe counts."""
    print(f"\n{'Action Name':<25} {'Start':>6} {'End':>6} {'Keys':>6}")
    print("-" * 50)
    for action in bpy.data.actions:
        start, end = action.frame_range
        keys = sum(len(fc.keyframe_points) for fc in action.fcurves)
        print(f"{action.name:<25} {int(start):>6} {int(end):>6} {keys:>6}")
```

---

## 27.5 THREE.JS ANIMATION MIXER SETUP

```javascript
import * as THREE from 'three';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

class ROCharacterAnimator {
    constructor(gltf) {
        this.mixer    = new THREE.AnimationMixer(gltf.scene);
        this.actions  = {};
        this.current  = null;
        
        // Load all clips from GLB
        gltf.animations.forEach(clip => {
            const action = this.mixer.clipAction(clip);
            this.actions[clip.name.toLowerCase()] = action;
            
            // Configure looping per animation
            if (['idle', 'walk', 'run', 'sit'].includes(clip.name.toLowerCase())) {
                action.setLoop(THREE.LoopRepeat, Infinity);
            } else {
                action.setLoop(THREE.LoopOnce, 1);
                action.clampWhenFinished = true; // Hold last frame on dead
            }
        });
        
        // Listen for animation finish events
        this.mixer.addEventListener('finished', (e) => {
            this.onAnimationFinished(e.action);
        });
    }
    
    play(stateName, crossfadeDuration = 0.1) {
        const newAction = this.actions[stateName.toLowerCase()];
        if (!newAction) {
            console.warn(`[ROCHAR] Unknown animation: ${stateName}`);
            return;
        }
        
        if (this.current && this.current !== newAction) {
            // Smooth crossfade between animations
            newAction.reset();
            newAction.play();
            this.current.crossFadeTo(newAction, crossfadeDuration, true);
        } else if (!this.current) {
            newAction.play();
        }
        
        this.current = newAction;
    }
    
    onAnimationFinished(action) {
        // Find which state finished
        const stateName = Object.keys(this.actions)
            .find(k => this.actions[k] === action);
        
        // Auto-return to idle after non-looping animations
        const noAutoReturn = ['dead', 'sit'];
        if (stateName && !noAutoReturn.includes(stateName)) {
            this.play('idle');
        }
    }
    
    update(deltaTime) {
        this.mixer.update(deltaTime);
    }
    
    dispose() {
        this.mixer.stopAllAction();
        this.mixer.uncacheRoot(this.mixer.getRoot());
    }
}
```

---

## 27.6 ANIMATION EXPORT CONFIGURATION

```
PRE-EXPORT CHECKLIST:
□ All actions visible in NLA editor (not starred/solo)
□ Each action has correct frame range set (no extra frames)
□ Rest pose applied before export (Ctrl+A → Apply as Rest Pose)
□ No action has keyframes on hidden or non-deforming bones
□ Foot anchor verified at Y=0 in all standing animations

BLENDER GLB EXPORT SETTINGS:
  Animation: ✓ ENABLED
  NLA Strips: ✓ ENABLED  
  Always Sample Animations: ✓ ENABLED
  Group by NLA Track: ✓ ENABLED (each track = named animation in GLB)
  Optimize Animation: ✓ ENABLED (removes redundant keyframes)

VERIFY IN THREE.JS:
  gltf.animations.forEach(clip => console.log(clip.name, clip.duration + 's'));
  
EXPECTED OUTPUT:
  idle    0.25s
  walk    1.0s
  run     0.67s
  attack_1h 0.47s
  cast    1.0s
  hurt    0.15s
  dead    0.63s
  sit     0.5s
```
