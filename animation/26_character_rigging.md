# 26 — CHARACTER RIGGING
## Armature Setup, Bone Hierarchy, and Weight Painting

> **Purpose:** Build a complete, animation-ready RO character armature with proper IK setup.  
> **Related:** [../modeling/14_blender_procedural_modeling.md](../modeling/14_blender_procedural_modeling.md), [27_skeletal_animation.md](27_skeletal_animation.md)

---

## 26.1 BONE HIERARCHY

```
ROCHAR STANDARD ARMATURE:
Root (at origin)
└── Hips (pelvis)
    ├── Spine
    │   └── Chest
    │       ├── Neck
    │       │   └── Head
    │       ├── Shoulder_L
    │       │   └── UpperArm_L
    │       │       └── ForeArm_L
    │       │           └── Hand_L
    │       │               ├── Weapon_Socket_L   (empty bone, weapon attach point)
    │       │               └── Shield_Socket_L   (empty bone, shield attach point)
    │       └── Shoulder_R
    │           └── UpperArm_R
    │               └── ForeArm_R
    │                   └── Hand_R
    │                       └── Weapon_Socket_R   (empty bone)
    ├── UpperLeg_L (thigh)
    │   └── LowerLeg_L (shin)
    │       └── Foot_L
    │           └── Toe_L
    └── UpperLeg_R
        └── LowerLeg_R
            └── Foot_R
                └── Toe_R
```

---

## 26.2 BLENDER PYTHON: BUILD FULL ARMATURE

```python
import bpy
import mathutils

def create_ro_armature(name="ROChar_Armature"):
    """
    Build a complete RO character armature via Blender Python API.
    Call from Scripting workspace or Blender headless.
    """
    # Create armature object
    arm_data = bpy.data.armatures.new(name)
    arm_obj  = bpy.data.objects.new(name, arm_data)
    bpy.context.scene.collection.objects.link(arm_obj)
    
    # Must be in Edit Mode to add bones
    bpy.context.view_layer.objects.active = arm_obj
    bpy.ops.object.mode_set(mode='EDIT')
    
    edit_bones = arm_data.edit_bones
    
    def add_bone(name, head, tail, parent_name=None, roll=0.0):
        """Add a bone with optional parent."""
        bone = edit_bones.new(name)
        bone.head   = mathutils.Vector(head)
        bone.tail   = mathutils.Vector(tail)
        bone.roll   = roll
        if parent_name:
            bone.parent = edit_bones[parent_name]
            bone.use_connect = False  # Always disconnected for RO characters
        return bone
    
    # ── ROOT AND SPINE ─────────────────────────────────────────────────────────
    # Coordinate: Y=forward, Z=up (Blender world space)
    # Character height: 1.0 unit (1 meter)
    
    add_bone("Root",    head=(0, 0, 0),    tail=(0, 0, 0.05))
    add_bone("Hips",    head=(0, 0, 0.42), tail=(0, 0, 0.50), parent_name="Root")
    add_bone("Spine",   head=(0, 0, 0.50), tail=(0, 0, 0.62), parent_name="Hips")
    add_bone("Chest",   head=(0, 0, 0.62), tail=(0, 0, 0.75), parent_name="Spine")
    add_bone("Neck",    head=(0, 0, 0.75), tail=(0, 0, 0.82), parent_name="Chest")
    add_bone("Head",    head=(0, 0, 0.82), tail=(0, 0, 0.98), parent_name="Neck")
    
    # ── LEFT ARM ───────────────────────────────────────────────────────────────
    add_bone("Shoulder_L",   head=(0.05, 0, 0.73),  tail=(0.10, 0, 0.73),   parent_name="Chest")
    add_bone("UpperArm_L",   head=(0.10, 0, 0.73),  tail=(0.22, 0, 0.73),   parent_name="Shoulder_L")
    add_bone("ForeArm_L",    head=(0.22, 0, 0.73),  tail=(0.32, 0, 0.65),   parent_name="UpperArm_L")
    add_bone("Hand_L",       head=(0.32, 0, 0.65),  tail=(0.38, 0, 0.62),   parent_name="ForeArm_L")
    add_bone("Weapon_Socket_L", head=(0.38, 0, 0.62), tail=(0.38, 0.08, 0.62), parent_name="Hand_L")
    add_bone("Shield_Socket_L", head=(0.38, 0, 0.62), tail=(0.38, -0.06, 0.62), parent_name="Hand_L")
    
    # ── RIGHT ARM ──────────────────────────────────────────────────────────────
    add_bone("Shoulder_R",   head=(-0.05, 0, 0.73), tail=(-0.10, 0, 0.73),  parent_name="Chest")
    add_bone("UpperArm_R",   head=(-0.10, 0, 0.73), tail=(-0.22, 0, 0.73),  parent_name="Shoulder_R")
    add_bone("ForeArm_R",    head=(-0.22, 0, 0.73), tail=(-0.32, 0, 0.65),  parent_name="UpperArm_R")
    add_bone("Hand_R",       head=(-0.32, 0, 0.65), tail=(-0.38, 0, 0.62),  parent_name="ForeArm_R")
    add_bone("Weapon_Socket_R", head=(-0.38, 0, 0.62), tail=(-0.38, 0.08, 0.62), parent_name="Hand_R")
    
    # ── LEFT LEG ───────────────────────────────────────────────────────────────
    add_bone("UpperLeg_L",  head=( 0.08, 0, 0.42), tail=( 0.08, 0, 0.23),  parent_name="Hips")
    add_bone("LowerLeg_L",  head=( 0.08, 0, 0.23), tail=( 0.08, 0, 0.07),  parent_name="UpperLeg_L")
    add_bone("Foot_L",      head=( 0.08, 0, 0.07), tail=( 0.08, 0.10, 0.03), parent_name="LowerLeg_L")
    add_bone("Toe_L",       head=( 0.08, 0.10, 0.03), tail=( 0.08, 0.16, 0.02), parent_name="Foot_L")
    
    # ── RIGHT LEG ──────────────────────────────────────────────────────────────
    add_bone("UpperLeg_R",  head=(-0.08, 0, 0.42), tail=(-0.08, 0, 0.23),  parent_name="Hips")
    add_bone("LowerLeg_R",  head=(-0.08, 0, 0.23), tail=(-0.08, 0, 0.07),  parent_name="UpperLeg_R")
    add_bone("Foot_R",      head=(-0.08, 0, 0.07), tail=(-0.08, 0.10, 0.03), parent_name="LowerLeg_R")
    add_bone("Toe_R",       head=(-0.08, 0.10, 0.03), tail=(-0.08, 0.16, 0.02), parent_name="Foot_R")
    
    # Return to Object Mode
    bpy.ops.object.mode_set(mode='OBJECT')
    
    print(f"[ROCHAR] Armature '{name}' created with {len(arm_data.bones)} bones.")
    return arm_obj
```

---

## 26.3 IK CHAIN SETUP (LEGS)

```python
def setup_leg_ik(arm_obj, side):
    """
    Set up 2-bone IK chain for a leg.
    side: 'L' or 'R'
    """
    bpy.context.view_layer.objects.active = arm_obj
    bpy.ops.object.mode_set(mode='POSE')
    
    pose_bones = arm_obj.pose.bones
    
    # Create IK target empty
    ik_target = bpy.data.objects.new(f"IK_Foot_{side}", None)
    bpy.context.scene.collection.objects.link(ik_target)
    # Position at foot level
    foot_pos = arm_obj.data.bones[f"Foot_{side}"].head_local
    ik_target.location = arm_obj.matrix_world @ foot_pos
    
    # Add IK constraint to LowerLeg bone
    lower_leg = pose_bones[f"LowerLeg_{side}"]
    ik_constraint = lower_leg.constraints.new('IK')
    ik_constraint.target       = ik_target
    ik_constraint.chain_count  = 2  # Affects LowerLeg + UpperLeg
    ik_constraint.iterations   = 10 # Solver iterations (10 is enough for 2-bone)
    ik_constraint.use_tail     = True
    
    # Pole target (prevents knee from folding backward)
    pole_target = bpy.data.objects.new(f"IK_Pole_{side}", None)
    bpy.context.scene.collection.objects.link(pole_target)
    knee_offset = 1 if side == 'L' else -1
    pole_target.location = ik_target.location + mathutils.Vector((knee_offset * 0.3, -0.3, 0.3))
    
    ik_constraint.pole_target  = pole_target
    ik_constraint.pole_angle   = 0.0  # Adjust if knee points wrong direction
    
    bpy.ops.object.mode_set(mode='OBJECT')
    print(f"[ROCHAR] IK chain set up for {side} leg.")
```

---

## 26.4 WEIGHT PAINTING STRATEGY

```
WEIGHT PAINTING RULES FOR RO CHARACTERS:

1. AUTOMATIC WEIGHTS FIRST:
   → Select mesh, then Shift+Click armature
   → Ctrl+P → "With Automatic Weights"
   → Blender assigns weights based on bone proximity
   → Quality: 70–80% correct for simple characters

2. MANUAL CORRECTIONS NEEDED:
   → Hip/Spine boundary: blend smoothly across 3 vertices
   → Shoulder junction: prevent chest deforming when arm raises
   → Knee: ensure LowerLeg weight terminates before mid-thigh
   → Fingers: clean separation between adjacent finger bones

3. VERTEX GROUP NAMING:
   → Each bone creates a vertex group with matching name
   → "Hips", "Spine", "Head", "UpperArm_L", etc.
   → Equipment socket bones DON'T need vertex groups (empty bones)

4. WEIGHT PAINTING TOOLS:
   → Add (blue brush): increase weight toward 1.0
   → Subtract (red brush): decrease weight toward 0.0
   → Smooth: blend weight gradients
   → Normalize All: ensure sum of all weights for each vertex = 1.0

5. VALIDATION:
   → Pose arm to 90°: check shoulder area doesn't crease
   → Bend knee to 90°: check smooth deformation at knee
   → Open/close hand: check finger independence
```

---

## 26.5 BONE CONSTRAINTS FOR EQUIPMENT

```python
def attach_equipment_to_socket(arm_obj, equip_mesh, socket_name):
    """
    Parent equipment mesh to a socket bone so it follows the bone transform.
    """
    # Set equipment mesh parent to armature
    equip_mesh.parent = arm_obj
    
    # Set parent type to BONE
    equip_mesh.parent_bone = socket_name
    equip_mesh.parent_type = 'BONE'
    
    # Clear parent inverse (so equipment uses bone's world transform)
    bpy.context.view_layer.objects.active = equip_mesh
    bpy.ops.object.parent_no_inverse_set(keep_transform=False)
    
    print(f"[ROCHAR] '{equip_mesh.name}' attached to socket '{socket_name}'")

# Usage:
# attach_equipment_to_socket(char_armature, sword_mesh, "Weapon_Socket_R")
# attach_equipment_to_socket(char_armature, shield_mesh, "Shield_Socket_L")
```

---

## 26.6 RIGGING VALIDATION CHECKLIST

```
GEOMETRY:
□ Armature modifier applied to character mesh
□ All vertices have at least one bone weight > 0
□ No vertex has total weight != 1.0 (normalize all)
□ Equipment socket bones have no vertex weights (empty)

DEFORMATION:
□ Rotate each major joint: smooth deformation, no pinching
□ No mesh intersections during extreme poses
□ Head rotates naturally without neck stretching
□ Arm raises without clipping through chest

NAMING:
□ All bones follow ROCHAR naming convention (UpperArm_L, etc.)
□ Vertex groups match bone names exactly (case-sensitive)
□ Socket bones named with _Socket_ suffix

EXPORT:
□ All armature modifiers applied before GLB export
□ Pose cleared to Rest Pose before export
□ Root bone at world origin (0, 0, 0)
□ Character standing straight in rest pose (T-pose preferred)
```
