# 28 — ANIMATION VALIDATION
## QA Procedures, Automated Checks, and Visual Regression

> **Purpose:** Systematically verify animation quality before shipping.  
> **Related:** [27_skeletal_animation.md](27_skeletal_animation.md), [../integration/32_quality_assurance.md](../integration/32_quality_assurance.md)

---

## 28.1 VALIDATION CHECKLIST

```
COMPLETENESS:
□ All 11 required animation states present
□ Each animation has correct frame count
□ Each animation has correct FPS setting
□ Dead animation clamps on last frame
□ Loop flag set correctly for each state

TIMING:
□ Idle: 4 frames, natural breathing pace (1 second loop)
□ Walk: 8 frames, feet contact ground at frames 1 and 5
□ Attack: hit frame fires at correct frame (see events)
□ Cast: spell frame at correct position
□ Dead: doesn't loop, holds frame 5

DEFORMATION:
□ No mesh clipping through itself during any pose
□ No mesh clipping through equipment during animation
□ Smooth weight transitions at all major joints
□ No foot skating (foot slides on ground during walk)
□ Pivot point stays at foot anchor (Y=0) throughout all standing animations

EVENTS:
□ attack_1h: hit_connect event at frame 3
□ attack_2h: hit_connect event at frame 4
□ cast: spell_fire event at frame 9
□ walk: left_footstep at frame 1, right_footstep at frame 5
□ All event frame indices match metadata JSON

EXPORT:
□ GLB loads without warnings in Three.js
□ All animations accessible via gltf.animations array
□ Animation names match ANIMATION_STATES keys (lowercase)
□ No duplicate animation names
```

---

## 28.2 FOOT SKATING DETECTION

```python
import bpy
import mathutils

def detect_foot_skating(arm_obj, action_name, 
                         threshold_m_per_frame=0.005):
    """
    Check if the foot bone moves horizontally while in contact with ground.
    'Skating' = foot slides when it should be planted.
    
    threshold_m_per_frame: Max allowed horizontal movement per frame (meters).
    """
    action = bpy.data.actions.get(action_name)
    if not action:
        print(f"[ROCHAR] Action '{action_name}' not found.")
        return
    
    bpy.context.view_layer.objects.active = arm_obj
    bpy.ops.object.mode_set(mode='POSE')
    
    # Get foot bone positions at each frame
    foot_bones = ["Foot_L", "Foot_R"]
    
    issues = []
    
    arm_obj.animation_data.action = action
    frame_start = int(action.frame_range[0])
    frame_end   = int(action.frame_range[1])
    
    for bone_name in foot_bones:
        bone = arm_obj.pose.bones.get(bone_name)
        if not bone:
            continue
        
        prev_pos = None
        
        for frame in range(frame_start, frame_end + 1):
            bpy.context.scene.frame_set(frame)
            
            # Get world position of foot bone head
            world_pos = arm_obj.matrix_world @ bone.head
            
            if prev_pos is not None:
                # Horizontal movement = XZ plane distance
                horiz_move = mathutils.Vector([
                    world_pos.x - prev_pos.x,
                    0,
                    world_pos.z - prev_pos.z
                ]).length
                
                # If foot is near ground (Y < 0.05m) and moving too much:
                if world_pos.y < 0.05 and horiz_move > threshold_m_per_frame:
                    issues.append({
                        'bone':  bone_name,
                        'frame': frame,
                        'movement_m': horiz_move
                    })
            
            prev_pos = world_pos.copy()
    
    bpy.ops.object.mode_set(mode='OBJECT')
    
    if issues:
        print(f"[ROCHAR] FOOT SKATING DETECTED in '{action_name}':")
        for issue in issues:
            print(f"  → {issue['bone']} at frame {issue['frame']}: "
                  f"{issue['movement_m']*100:.1f}cm per frame")
    else:
        print(f"[ROCHAR] '{action_name}': No foot skating detected. ✓")
    
    return issues
```

---

## 28.3 MESH INTERSECTION DETECTION

```python
def detect_mesh_clipping(mesh_obj_a, mesh_obj_b, action_name, arm_obj,
                           check_every_n_frames=2):
    """
    Approximate check: if bounding boxes overlap at any animation frame.
    For exact check, use BVH tree per frame (slower).
    """
    import bpy
    from mathutils.bvhtree import BVHTree
    
    action = bpy.data.actions.get(action_name)
    if not action:
        return []
    
    arm_obj.animation_data.action = action
    frame_start = int(action.frame_range[0])
    frame_end   = int(action.frame_range[1])
    
    clip_frames = []
    
    for frame in range(frame_start, frame_end + 1, check_every_n_frames):
        bpy.context.scene.frame_set(frame)
        bpy.context.evaluated_depsgraph_get()  # Update armature deformation
        
        # Get evaluated meshes (with armature deformation applied)
        depsgraph = bpy.context.evaluated_depsgraph_get()
        eval_a = mesh_obj_a.evaluated_get(depsgraph)
        eval_b = mesh_obj_b.evaluated_get(depsgraph)
        
        mesh_a = eval_a.to_mesh()
        mesh_b = eval_b.to_mesh()
        
        # Build BVH trees
        bvh_a = BVHTree.FromObject(eval_a, depsgraph)
        bvh_b = BVHTree.FromObject(eval_b, depsgraph)
        
        # Check overlap
        overlaps = bvh_a.overlap(bvh_b)
        
        if overlaps:
            clip_frames.append({
                'frame': frame,
                'overlap_count': len(overlaps)
            })
        
        eval_a.to_mesh_clear()
        eval_b.to_mesh_clear()
    
    if clip_frames:
        print(f"[ROCHAR] MESH CLIPPING detected in '{action_name}':")
        for cf in clip_frames:
            print(f"  → Frame {cf['frame']}: {cf['overlap_count']} overlapping polygon pairs")
    else:
        print(f"[ROCHAR] '{action_name}': No mesh clipping detected. ✓")
    
    return clip_frames
```

---

## 28.4 ANIMATION TIMING VALIDATION

```python
def validate_animation_timing(arm_obj):
    """
    Check that all animations match expected frame counts and FPS.
    """
    EXPECTED = {
        'idle':       {'frames': 4,  'fps': 4},
        'walk':       {'frames': 8,  'fps': 8},
        'run':        {'frames': 8,  'fps': 12},
        'attack_1h':  {'frames': 7,  'fps': 15},
        'attack_2h':  {'frames': 8,  'fps': 15},
        'cast':       {'frames': 10, 'fps': 10},
        'skill':      {'frames': 8,  'fps': 12},
        'hurt':       {'frames': 3,  'fps': 20},
        'dead':       {'frames': 5,  'fps': 8},
        'sit':        {'frames': 2,  'fps': 4},
        'pickup':     {'frames': 4,  'fps': 8}
    }
    
    errors = []
    warnings = []
    
    for name, expected in EXPECTED.items():
        # Find matching action (case-insensitive)
        action = next((a for a in bpy.data.actions 
                       if name in a.name.lower()), None)
        
        if not action:
            errors.append(f"MISSING: '{name}' animation not found")
            continue
        
        actual_frames = int(action.frame_range[1] - action.frame_range[0]) + 1
        
        if actual_frames != expected['frames']:
            errors.append(
                f"FRAME COUNT: '{name}' has {actual_frames} frames, "
                f"expected {expected['frames']}"
            )
        
        # Check scene FPS matches (can't directly read action FPS, 
        # but check scene FPS was set correctly when action was made)
        scene_fps = bpy.context.scene.render.fps
        if scene_fps != expected['fps']:
            warnings.append(
                f"FPS MISMATCH: Scene is {scene_fps}fps but '{name}' "
                f"expects {expected['fps']}fps"
            )
    
    # Print results
    if errors:
        print("[ROCHAR] ANIMATION ERRORS:")
        for e in errors: print(f"  ✗ {e}")
    if warnings:
        print("[ROCHAR] ANIMATION WARNINGS:")
        for w in warnings: print(f"  ⚠ {w}")
    if not errors and not warnings:
        print("[ROCHAR] All animation timing checks PASSED. ✓")
    
    return {'errors': errors, 'warnings': warnings}
```

---

## 28.5 THREE.JS VISUAL REGRESSION TEST

```javascript
/**
 * Automated visual frame capture for regression testing.
 * Renders each animation state at frame 0 and frame N/2 (mid-animation).
 * Saves PNG screenshots for manual or automated comparison.
 */
async function captureAnimationFrames(character, renderer, scene, camera) {
    const animator = character.animator;
    const states   = ['idle', 'walk', 'attack_1h', 'cast', 'hurt', 'dead'];
    
    const frames = {};
    
    for (const state of states) {
        animator.play(state);
        
        // Capture frame at time 0
        animator.update(0);
        renderer.render(scene, camera);
        frames[`${state}_start`] = captureCanvas(renderer.domElement);
        
        // Advance to middle of animation
        const clip = animator.actions[state]?._clip;
        if (clip) {
            animator.update(clip.duration / 2);
        }
        renderer.render(scene, camera);
        frames[`${state}_mid`] = captureCanvas(renderer.domElement);
    }
    
    return frames;
}

function captureCanvas(canvas) {
    return canvas.toDataURL('image/png');
}

// Compare against baseline screenshots:
function compareFrames(baseline, current, threshold = 0.02) {
    // Simple pixel comparison (requires canvas or Jimp library)
    // Returns % difference
    // If > threshold, flag as regression
}
```

---

## 28.6 PIVOT CONSISTENCY CHECK

```python
def check_foot_pivot(arm_obj, action_names):
    """
    Verify that the Root bone stays at Y=0 (ground level) throughout
    all standing animations. If Root lifts off ground, the character
    will float/sink in-game.
    """
    root_bone_name = "Root"
    
    for action_name in action_names:
        action = bpy.data.actions.get(action_name)
        if not action:
            continue
        
        arm_obj.animation_data.action = action
        
        max_y_offset = 0
        worst_frame  = 0
        
        for frame in range(int(action.frame_range[0]), int(action.frame_range[1]) + 1):
            bpy.context.scene.frame_set(frame)
            
            root_pose = arm_obj.pose.bones.get(root_bone_name)
            if root_pose:
                y_world = (arm_obj.matrix_world @ root_pose.head).y
                if abs(y_world) > max_y_offset:
                    max_y_offset = abs(y_world)
                    worst_frame  = frame
        
        if max_y_offset > 0.001:
            print(f"[ROCHAR] PIVOT DRIFT '{action_name}': "
                  f"Max Y offset {max_y_offset:.4f}m at frame {worst_frame}")
        else:
            print(f"[ROCHAR] '{action_name}': Pivot clean (max drift {max_y_offset*1000:.2f}mm). ✓")
```
