# 29 — RAGNAROK ONLINE ASSET FORMATS
## RSM, GND, RSW, SPR, ACT, STR — Format Reference

> **Purpose:** Reference documentation for proprietary RO file formats relevant to character reconstruction.  
> **Related:** [../references/01_ro_visual_dna.md](../references/01_ro_visual_dna.md), [30_threejs_and_opengl_integration.md](30_threejs_and_opengl_integration.md)

---

> **LEGAL NOTE:** This documentation describes file format structures for research and interoperability purposes only. It does not include copyrighted game data. Only use this knowledge with assets you have legitimate rights to.

---

## 29.1 RSM — 3D MODEL FORMAT

**Used for:** World objects, NPCs, some monster models.  
**Extension:** `.rsm`

```
RSM STRUCTURE:
  Header:
    magic[4]        : "GRSM"
    major_version   : uint8
    minor_version   : uint8
    animation_length: int32  (frames in main animation)
    shade_type      : int32  (lighting mode)
    alpha           : uint8  (global alpha 0–255)
    reserved[16]    : uint8

  Textures:
    texture_count : int32
    textures[]    : char[40] × texture_count  (texture filenames)

  Root Node Name:
    main_node_name : char[40]

  Nodes (recursive tree):
    node_count : int32
    nodes[]:
      name        : char[40]
      parent_name : char[40]
      texture_count : int32
      texture_ids   : int32[] (indices into texture list)
      offset_matrix : float[9] (3×3 rotation matrix)
      pos_offset    : float[3] (translation from parent)
      rot_angle     : float    (rotation keyframe angle in radians)
      rot_axis      : float[3]
      scale         : float[3]
      
      vertex_count  : int32
      vertices      : float[3] × vertex_count  (XYZ positions)
      
      texture_vertex_count : int32
      texture_vertices     : (float[2] color + float[2] uv) × count
      
      face_count : int32
      faces[]:
        vertex_indices[3]      : uint16 × 3
        texture_vertex_ids[3]  : uint16 × 3
        texture_id             : uint16
        padding                : uint16
        two_sided              : int32
        smoothGroup            : int32

RELEVANT FOR ROCHAR:
  → Vertex positions → rebuild mesh topology
  → Texture names → identify which textures the model uses
  → Face list → rebuild UV mapping
  → Node tree → understand articulation points
```

---

## 29.2 GND — GROUND/TERRAIN FORMAT

**Used for:** Map terrain, tile layout, baked lightmaps.  
**Extension:** `.gnd`

```
GND STRUCTURE:
  Header:
    magic[4]       : "GRGN"
    major_version  : uint8
    minor_version  : uint8
    width          : int32  (map width in tiles)
    height         : int32  (map height in tiles)
    zoom           : float  (tile world scale, usually 10.0)
    texture_count  : int32
    texture_len    : int32  (always 80)
    textures[]     : char[80] × texture_count

  Lightmap data:
    lightmap_count : int32
    lightmap_width : int32  (always 8)
    lightmap_height: int32  (always 8)
    grid_size      : int32  (always 1 = 8×8 per tile)
    lightmaps[]    : {
      brightness[64] : uint8    (8×8 grid of intensity values)
      color[192]     : uint8    (8×8 grid of RGB)
    } × lightmap_count

  Surface data (tiles):
    tiles[width × height]:
      height[4]       : float[4]  (height at each corner: SW, NW, NE, SE)
      top_surf_id     : int32    (surface texture index, -1 = no top surface)
      front_surf_id   : int32    (front face texture, -1 = none)
      right_surf_id   : int32    (right face texture, -1 = none)

RELEVANT FOR ROCHAR:
  → Terrain height data → place characters correctly on ground
  → Lightmap data → reproduce zone lighting atmosphere
  → Texture names → identify zone tiles for color palette reference
```

---

## 29.3 RSW — WORLD FORMAT (MAP SCENE)

**Used for:** Complete map scene: objects, lights, fog, effects.  
**Extension:** `.rsw`

```
RSW STRUCTURE:
  Header:
    magic[4]       : "GRSW"
    major_version  : uint8
    minor_version  : uint8
    ini_file[40]   : char (map config .ini reference)
    gnd_file[40]   : char (ground file name)
    gat_file[40]   : char (walkability map)
    src_file[40]   : char (source file reference)
    
  Water settings:
    water_level    : float
    water_type     : int32
    wave_height    : float
    wave_speed     : float
    wave_pitch     : float
    anim_speed     : int32
    
  Light settings:
    light_longitude: int32  (sun angle: 0–360)
    light_latitude : int32  (sun elevation: 0–90)
    diffuse_color  : float[3]  (RGB, 0–1)
    ambient_color  : float[3]  (RGB, 0–1)
    
  Fog settings (version ≥ 2.6):
    fog_start      : float  (normalized 0–1)
    fog_end        : float
    fog_color      : float[3]
    fog_near_plane : float
    
  Objects (models placed in world):
    object_count : int32
    objects[]:
      type         : int32  (1=Model, 2=Light, 3=Sound, 4=Effect)
      name[40]     : char
      anim_type    : int32
      anim_speed   : float
      block_type   : int32
      filename[80] : char   (RSM file reference)
      position     : float[3]
      rotation     : float[3]  (degrees)
      scale        : float[3]

RELEVANT FOR ROCHAR:
  → Light longitude/latitude → replicate directional light in GLSL
  → Diffuse/Ambient colors → zone atmosphere presets
  → Fog settings → replicate fog in shader
```

---

## 29.4 SPR — SPRITE FORMAT

**Used for:** Character/NPC/monster sprite sheets.  
**Extension:** `.spr`

```
SPR STRUCTURE (version 2.0):
  Header:
    magic[2]       : "SP"
    major_version  : uint8
    minor_version  : uint8
    indexed_count  : uint16  (number of indexed/palette frames)
    rgba_count     : uint16  (number of RGBA frames, v2.1+)
    
  Palette (version < 2.1):
    palette[256×4] : uint8  (RGBA palette, 1024 bytes total)
    
  Indexed Frames (8-bit indexed color):
    frames[indexed_count]:
      width  : uint16
      height : uint16
      data   : uint8[width × height]  (palette indices, row-major)
      
  RGBA Frames (version 2.1+, 32-bit direct color):
    frames[rgba_count]:
      width  : uint16
      height : uint16
      data   : uint8[width × height × 4]  (RGBA, row-major)
      
  Palette (version 2.1+, appended at end):
    palette[256×4] : uint8

MAGENTA TRANSPARENCY:
  → Palette index 0 is always magenta (255, 0, 255, 255)
  → Any pixel referencing palette[0] = transparent
  → When converting: map palette[0] pixels to alpha=0

RELEVANT FOR ROCHAR:
  → Frame dimensions → sprite canvas size reference
  → Palette → extract exact character colors
  → Frame count → verify animation completeness
  → Layout → understand direction/animation ordering
```

---

## 29.5 ACT — ACTION/ANIMATION FORMAT

**Used for:** Animation metadata linking to SPR frames.  
**Extension:** `.act`

```
ACT STRUCTURE:
  Header:
    magic[2]       : "AC"
    major_version  : uint8
    minor_version  : uint8
    action_count   : uint16
    reserved[10]   : uint8
    
  Actions[action_count]:
    frame_count  : uint32
    frames[frame_count]:
      range[4]     : int32[4]  (bounding box: l, t, r, b)
      layer_count  : uint32
      
      layers[layer_count]:
        pos_x    : int32    (sprite offset X from anchor)
        pos_y    : int32    (sprite offset Y from anchor)
        spr_num  : int32    (sprite frame index from SPR file)
        mirror   : uint32   (0=normal, 1=flip horizontally)
        color    : uint8[4] (RGBA tint for this layer, v2.0+)
        x_scale  : float    (scale X, v2.0+, default 1.0)
        y_scale  : float    (scale Y, v2.0+, default 1.0)
        rot_deg  : int32    (rotation in degrees, v2.0+)
        type     : int32    (0=sprite, 1=hidden)
        
      event_id : int32  (index into events, -1=none)
      
  Events (v2.0+):
    event_count : int32
    events[event_count]:
      name[40]  : char  (event name, e.g. "atk", "atk2", "hit")
      
  Intervals (v2.0+, one per action):
    interval[action_count] : float  (ms per frame for this action)

ACTION INDEX → ANIMATION STATE:
  ACT actions are numbered. Mapping varies by character type.
  Common mapping for player characters:
    0–7:   Walk (8 directions: SW, W, NW, N, NE, E, SE, S)
    8–15:  Idle (8 directions)
    16–23: Sit (8 directions)
    24–31: Pickup (8 directions)
    32–39: Hurt (8 directions)
    40–47: Dead (8 directions)
    48–55: Attack 1H (8 directions)
    ... varies by class

RELEVANT FOR ROCHAR:
  → pos_x/pos_y per layer → precise anchor offsets for compositing
  → mirror flag → confirms which directions are mirrored
  → event names ("atk") → map to hit_connect events
  → interval values → actual animation FPS per action
```

---

## 29.6 STR — EFFECT STREAMING FORMAT

**Used for:** Particle effects, skill visual effects (fire, lightning, etc.)  
**Extension:** `.str`

```
STR STRUCTURE:
  Header:
    magic[4]       : "STRM"
    version        : int32  (always 148)
    fps            : int32  (effect playback FPS)
    max_key        : int32  (total frames in effect)
    layer_count    : int32
    reserved[16]   : uint8
    
  Layers[layer_count]:
    keyframe_count : int32
    keyframes[]:
      frame_num    : int32
      type         : int32  (0=general, 1=extra tag)
      pos_x, pos_y : float
      anim_start   : float
      anim_type    : int32  (0=still, 1=linear anim, 2=random)
      delay        : float
      angle        : float
      color[4]     : float[4]  (RGBA, 0–1)
      src_alpha    : int32
      dst_alpha    : int32
      mtx_1..10    : int32  (texture animation frame indices)
      
RELEVANT FOR ROCHAR:
  → Layer color → replicate effect colors in additive blend shader
  → FPS + max_key → drive Three.js effect animation timing
  → Blend factors → match src_alpha/dst_alpha with WebGL blend modes
```
