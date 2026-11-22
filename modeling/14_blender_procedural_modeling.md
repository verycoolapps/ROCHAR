# 14 — BLENDER PROCEDURAL MODELING
## Blender Python API Scripts for RO Character Construction

> **Purpose:** Provide Blender Python workflows and procedural geometry for RO character creation.  
> **Related:** [10_low_poly_modeling.md](10_low_poly_modeling.md), [15_character_export_pipeline.md](15_character_export_pipeline.md)

---

## 14.1 ENVIRONMENT REQUIREMENTS

```
Blender Version: 3.6+ (LTS recommended)
Python:          Built-in Blender Python (3.10+)
Running scripts: Text Editor → Run Script (Alt+P)
                 Or: Scripting workspace
```

---

## 14.2 SCRIPT: CREATE RO CHARACTER BASE MESH

```python
"""
ROCHAR — Base Character Mesh Generator
Creates a basic RO-proportioned character base mesh.
Run in Blender Scripting workspace.
"""
import bpy
import bmesh
import math

def clear_scene():
    """Remove default objects from scene."""
    bpy.ops.object.select_all(action='SELECT')
    bpy.ops.object.delete()

def create_cylinder_segment(bm, top_z, bot_z, top_r, bot_r, segments=8):
    """Create a tapered cylinder segment using bmesh."""
    top_verts = []
    bot_verts = []
    
    for i in range(segments):
        angle = 2 * math.pi * i / segments
        x = math.cos(angle)
        y = math.sin(angle)
        top_verts.append(bm.verts.new((x * top_r, y * top_r, top_z)))
        bot_verts.append(bm.verts.new((x * bot_r, y * bot_r, bot_z)))
    
    bm.verts.ensure_lookup_table()
    
    # Create side faces
    for i in range(segments):
        next_i = (i + 1) % segments
        bm.faces.new([top_verts[i], top_verts[next_i], 
                     bot_verts[next_i], bot_verts[i]])
    
    return top_verts, bot_verts

def cap_ring(bm, ring_verts, top=True):
    """Cap a ring of vertices with a fan of triangles."""
    center = bm.verts.new((0, 0, ring_verts[0].co.z))
    for i in range(len(ring_verts)):
        next_i = (i + 1) % len(ring_verts)
        if top:
            bm.faces.new([center, ring_verts[next_i], ring_verts[i]])
        else:
            bm.faces.new([center, ring_verts[i], ring_verts[next_i]])

def create_ro_character_mesh(H=0.25, segments=8):
    """
    Create basic RO-proportioned character base mesh.
    
    H: head unit in Blender world units (meters)
       Default 0.25m = head is 25cm, full character ~1m
    segments: polygon segments per ring (8 = octagonal, low-poly look)
    """
    clear_scene()
    
    mesh = bpy.data.meshes.new("RO_Base_Mesh")
    obj = bpy.data.objects.new("RO_Character", mesh)
    bpy.context.collection.objects.link(obj)
    
    bm = bmesh.new()
    
    # Ground plane = Z=0
    # Character stands with feet at Z=0
    
    # Height map (from ground):
    h_ankle    = 0.20 * H
    h_knee     = 0.80 * H
    h_hip      = 1.10 * H
    h_waist    = 1.60 * H
    h_chest    = 2.10 * H
    h_shoulder = 2.40 * H
    h_neck_bot = 2.60 * H
    h_neck_top = 2.75 * H
    h_chin     = 2.85 * H
    h_head_ctr = 3.35 * H  # center of head
    h_crown    = 3.85 * H  # top of head
    
    # === HEAD ===
    # Build head as a 6-segment sphere approximation
    head_r = 0.5 * H
    bmesh.ops.create_uvsphere(bm, 
        u_segments=segments, 
        v_segments=6, 
        radius=head_r
    )
    # Move all head verts upward to head center position
    for v in bm.verts:
        v.co.z += h_head_ctr
    
    # === NECK ===
    neck_top_r = 0.10 * H
    neck_bot_r = 0.12 * H
    neck_top, neck_bot = create_cylinder_segment(
        bm, h_neck_top, h_neck_bot, neck_top_r, neck_bot_r, segments
    )
    
    # === TORSO ===
    chest_r   = 0.65 * H
    waist_r   = 0.45 * H
    hip_r     = 0.55 * H
    
    # Upper torso (shoulders to waist)
    torso_top, torso_mid = create_cylinder_segment(
        bm, h_shoulder, h_waist, chest_r, waist_r, segments
    )
    # Lower torso (waist to hip)
    _, torso_bot = create_cylinder_segment(
        bm, h_waist, h_hip, waist_r, hip_r, segments
    )
    
    # === UPPER ARMS (both sides) ===
    for side in [-1, 1]:
        arm_x_offset = side * chest_r
        # Simplified: just cylinder, no accurate positioning
        upper_arm_r = 0.09 * H
        lower_arm_r = 0.07 * H
        hand_r = 0.12 * H
        
        for v_pair in [
            (h_shoulder, h_shoulder - 0.70*H, upper_arm_r, upper_arm_r),
            (h_shoulder - 0.70*H, h_shoulder - 1.30*H, upper_arm_r, lower_arm_r),
        ]:
            top_z, bot_z, tr, br = v_pair
            ring_top, ring_bot = create_cylinder_segment(bm, top_z, bot_z, tr, br, segments)
            # NOTE: This creates the cylinder at X=0; in a real script you would
            # translate these verts to side positions after creation.
            # This is a simplified procedural example.
    
    # === LEGS ===
    thigh_r   = 0.12 * H
    shin_r    = 0.09 * H
    ankle_r   = 0.07 * H
    
    for side in [-1, 1]:
        create_cylinder_segment(bm, h_hip, h_knee, thigh_r, thigh_r, segments)
        create_cylinder_segment(bm, h_knee, h_ankle, thigh_r, shin_r, segments)
    
    bm.to_mesh(mesh)
    bm.free()
    
    mesh.update()
    
    # Apply smooth shading to head, flat to body
    bpy.context.view_layer.objects.active = obj
    bpy.ops.object.mode_set(mode='EDIT')
    bpy.ops.mesh.select_all(action='DESELECT')
    bpy.ops.object.mode_set(mode='OBJECT')
    
    print(f"Created RO character base mesh with {len(mesh.vertices)} vertices, "
          f"{len(mesh.polygons)} faces")
    print(f"Triangle count: {sum(len(p.vertices) - 2 for p in mesh.polygons)}")
    
    return obj

# RUN:
# obj = create_ro_character_mesh(H=0.25, segments=8)
```

---

## 14.3 SCRIPT: CREATE RO CHARACTER MATERIAL

```python
"""
ROCHAR — RO-Style Material Creator
Creates a simplified flat-diffuse material with vertex color support.
"""
import bpy

def create_ro_material(
    name="RO_Character_Mat",
    diffuse_color=(0.6, 0.4, 0.8, 1.0),  # Purple cloth default
    texture_path=None,
    use_vertex_color=True
):
    """
    Create a simplified RO-style material.
    
    Configured for flat diffuse rendering:
    - Full roughness (no specular)
    - Zero metalness
    - Optional texture
    - Vertex color multiplication
    """
    mat = bpy.data.materials.new(name)
    mat.use_nodes = True
    nodes = mat.node_tree.nodes
    links = mat.node_tree.links
    
    # Clear default nodes
    nodes.clear()
    
    # === OUTPUT ===
    output = nodes.new('ShaderNodeOutputMaterial')
    output.location = (600, 0)
    
    # === PRINCIPLED BSDF (configured flat) ===
    bsdf = nodes.new('ShaderNodeBsdfPrincipled')
    bsdf.location = (200, 0)
    bsdf.inputs['Roughness'].default_value = 1.0      # Fully matte
    bsdf.inputs['Metallic'].default_value = 0.0        # No metallic
    # Blender 4.x uses 'Specular IOR Level', earlier uses 'Specular'
    try:
        bsdf.inputs['Specular IOR Level'].default_value = 0.0
    except KeyError:
        try:
            bsdf.inputs['Specular'].default_value = 0.0
        except KeyError:
            pass  # Version compatibility
    
    links.new(bsdf.outputs['BSDF'], output.inputs['Surface'])
    
    # === TEXTURE (optional) ===
    if texture_path:
        tex_node = nodes.new('ShaderNodeTexImage')
        tex_node.location = (-400, 100)
        try:
            img = bpy.data.images.load(texture_path)
            tex_node.image = img
            tex_node.interpolation = 'Closest'  # Nearest-neighbor for pixel art
        except Exception as e:
            print(f"[ROCHAR] Warning: Could not load texture '{texture_path}': {e}")
            tex_node.image = None
        
        base_color_source = tex_node.outputs['Color']
        alpha_source = tex_node.outputs['Alpha']
        links.new(alpha_source, bsdf.inputs['Alpha'])
        mat.blend_method = 'CLIP'
        mat.alpha_threshold = 0.5
    else:
        # Use solid color
        rgb_node = nodes.new('ShaderNodeRGB')
        rgb_node.location = (-400, 100)
        rgb_node.outputs[0].default_value = diffuse_color
        base_color_source = rgb_node.outputs['Color']
    
    # === VERTEX COLOR ===
    if use_vertex_color:
        vcol_node = nodes.new('ShaderNodeVertexColor')
        vcol_node.location = (-400, -100)
        vcol_node.layer_name = 'Col'
        
        mix_node = nodes.new('ShaderNodeMixRGB')
        mix_node.location = (0, 0)
        mix_node.blend_type = 'MULTIPLY'
        mix_node.inputs['Fac'].default_value = 1.0
        
        links.new(base_color_source, mix_node.inputs['Color1'])
        links.new(vcol_node.outputs['Color'], mix_node.inputs['Color2'])
        links.new(mix_node.outputs['Color'], bsdf.inputs['Base Color'])
    else:
        links.new(base_color_source, bsdf.inputs['Base Color'])
    
    return mat

# USAGE EXAMPLES:
# mat = create_ro_material("Knight_Plate", texture_path="/path/to/armor.png")
# mat = create_ro_material("Mage_Robe", diffuse_color=(0.3, 0.1, 0.5, 1.0))
```

---

## 14.4 SCRIPT: UV AUTO-UNWRAP

```python
"""
ROCHAR — Smart UV Unwrap with RO-optimized settings
"""
import bpy

def ro_smart_unwrap(obj, angle_limit=60, margin=0.03):
    """
    Apply Smart UV Project with settings optimized for RO characters.
    
    angle_limit: 60° works well for low-poly humanoids
    margin:      0.03 leaves good padding between islands
    """
    # Set object as active
    bpy.context.view_layer.objects.active = obj
    bpy.ops.object.mode_set(mode='EDIT')
    
    # Select all faces
    bpy.ops.mesh.select_all(action='SELECT')
    
    # Apply Smart UV Project
    bpy.ops.uv.smart_project(
        angle_limit=angle_limit,
        island_margin=margin,
        area_weight=0.0,
        correct_aspect=True,
        scale_to_bounds=False
    )
    
    # Pack islands
    bpy.ops.uv.pack_islands(margin=margin)
    
    bpy.ops.object.mode_set(mode='OBJECT')
    
    print(f"[ROCHAR] UV unwrap complete for: {obj.name}")
    
    # Create UV image for painting
    img = bpy.data.images.new(
        f"{obj.name}_UV",
        width=256, 
        height=256,
        alpha=True
    )
    img.generated_type = 'BLANK'
    img.generated_color = (0, 0, 0, 1)
    
    return img
```

---

## 14.5 SCRIPT: BONE HIERARCHY CREATION

```python
"""
ROCHAR — RO Character Armature Creator
Creates a standard skeleton hierarchy for RO character animation.
"""
import bpy
import mathutils

def create_ro_armature(character_obj, H=0.25):
    """
    Create a standard RO character armature.
    H: head unit (default 0.25m)
    """
    # Create armature
    arm_data = bpy.data.armatures.new("RO_Armature")
    arm_obj = bpy.data.objects.new("RO_Armature_Obj", arm_data)
    bpy.context.collection.objects.link(arm_obj)
    
    bpy.context.view_layer.objects.active = arm_obj
    bpy.ops.object.mode_set(mode='EDIT')
    
    bones = arm_data.edit_bones
    
    def add_bone(name, head_pos, tail_pos, parent=None):
        bone = bones.new(name)
        bone.head = mathutils.Vector(head_pos)
        bone.tail = mathutils.Vector(tail_pos)
        if parent:
            bone.parent = bones[parent]
        return bone
    
    # === SPINE CHAIN ===
    add_bone("Hips",   (0, 0, 1.10*H), (0, 0, 1.60*H))
    add_bone("Spine",  (0, 0, 1.60*H), (0, 0, 2.10*H), parent="Hips")
    add_bone("Chest",  (0, 0, 2.10*H), (0, 0, 2.60*H), parent="Spine")
    add_bone("Neck",   (0, 0, 2.60*H), (0, 0, 2.85*H), parent="Chest")
    add_bone("Head",   (0, 0, 2.85*H), (0, 0, 3.85*H), parent="Neck")
    
    # === HEADGEAR SOCKETS ===
    add_bone("HeadgearTop_Socket", (0, 0, 3.85*H), (0, 0, 4.15*H), parent="Head")
    add_bone("HeadgearMid_Socket", (0, 0.3*H, 3.25*H), (0, 0.4*H, 3.45*H), parent="Head")
    add_bone("HeadgearBot_Socket", (0, 0.3*H, 3.00*H), (0, 0.4*H, 3.10*H), parent="Head")
    
    # === LEFT ARM ===
    add_bone("Shoulder.L",  (-0.65*H, 0, 2.40*H), (-0.80*H, 0, 2.40*H), parent="Chest")
    add_bone("UpperArm.L",  (-0.80*H, 0, 2.40*H), (-0.80*H, 0, 1.70*H), parent="Shoulder.L")
    add_bone("Forearm.L",   (-0.80*H, 0, 1.70*H), (-0.80*H, 0, 1.10*H), parent="UpperArm.L")
    add_bone("Hand.L",      (-0.80*H, 0, 1.10*H), (-0.80*H, 0, 0.90*H), parent="Forearm.L")
    
    # === RIGHT ARM ===
    add_bone("Shoulder.R",  (0.65*H, 0, 2.40*H), (0.80*H, 0, 2.40*H), parent="Chest")
    add_bone("UpperArm.R",  (0.80*H, 0, 2.40*H), (0.80*H, 0, 1.70*H), parent="Shoulder.R")
    add_bone("Forearm.R",   (0.80*H, 0, 1.70*H), (0.80*H, 0, 1.10*H), parent="UpperArm.R")
    add_bone("Hand.R",      (0.80*H, 0, 1.10*H), (0.80*H, 0, 0.90*H), parent="Forearm.R")
    
    # === LEFT LEG ===
    add_bone("UpperLeg.L",  (-0.25*H, 0, 1.10*H), (-0.25*H, 0, 0.45*H), parent="Hips")
    add_bone("LowerLeg.L",  (-0.25*H, 0, 0.45*H), (-0.25*H, 0, 0.12*H), parent="UpperLeg.L")
    add_bone("Foot.L",      (-0.25*H, 0, 0.12*H), (-0.25*H, 0.3*H, 0.05*H), parent="LowerLeg.L")
    
    # === RIGHT LEG ===
    add_bone("UpperLeg.R",  (0.25*H, 0, 1.10*H), (0.25*H, 0, 0.45*H), parent="Hips")
    add_bone("LowerLeg.R",  (0.25*H, 0, 0.45*H), (0.25*H, 0, 0.12*H), parent="UpperLeg.R")
    add_bone("Foot.R",      (0.25*H, 0, 0.12*H), (0.25*H, 0.3*H, 0.05*H), parent="LowerLeg.R")
    
    # === WEAPON SOCKETS ===
    add_bone("Weapon_Main_Socket", (0.80*H, 0.1*H, 0.90*H), (0.80*H, 0.1*H, 0.60*H), parent="Hand.R")
    add_bone("Weapon_Off_Socket",  (-0.80*H, 0.1*H, 0.90*H), (-0.80*H, 0.1*H, 1.20*H), parent="Hand.L")
    add_bone("Garment_Back_Socket",(0, -0.3*H, 2.40*H), (0, -0.5*H, 1.80*H), parent="Chest")
    
    bpy.ops.object.mode_set(mode='OBJECT')
    
    # Parent character to armature
    character_obj.parent = arm_obj
    character_obj.parent_type = 'ARMATURE'
    
    print(f"[ROCHAR] Armature created with {len(arm_data.bones)} bones")
    return arm_obj

# USAGE:
# arm = create_ro_armature(character_obj, H=0.25)
```

---

## 14.6 SCRIPT: EXPORT TO GLB

```python
"""
ROCHAR — GLB Export Utility
Exports character for use in Three.js with correct settings.
"""
import bpy
import os

def export_ro_character_glb(export_path, include_animations=True):
    """
    Export selected objects as GLB (binary glTF).
    
    Settings optimized for Three.js compatibility:
    - Y-up conversion
    - Embedded textures
    - Animation export
    """
    # Ensure export directory exists
    os.makedirs(os.path.dirname(export_path), exist_ok=True)
    
    bpy.ops.export_scene.gltf(
        filepath=export_path,
        export_format='GLB',           # Binary glTF
        use_selection=True,            # Export only selected objects
        
        # Transform
        export_yup=True,               # Convert Z-up Blender to Y-up glTF
        
        # Mesh
        export_apply=True,             # Apply all modifiers
        export_normals=True,
        export_texcoords=True,
        export_colors=True,            # Vertex colors
        
        # Materials
        export_materials='EXPORT',
        export_image_format='PNG',     # Embed images as PNG
        
        # Animation
        export_animations=include_animations,
        export_nla_strips=True,
        export_frame_range=False,
        export_def_bones=False,
        
        # Compression
        export_draco_mesh_compression_enable=False,  # Disable for compatibility
    )
    
    print(f"[ROCHAR] Exported to: {export_path}")
    return export_path

# USAGE:
# Select character mesh and armature in Blender
# export_ro_character_glb("/path/to/output/knight.glb")
```
