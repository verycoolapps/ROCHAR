# ROCHAR
## Ragnarok Online Character Architect & Reconstruction Engine

**A Claude Code Skill — SUPER DUPER SAKTI Edition**  
*Sistem pengetahuan 33 modul untuk merekonstruksi karakter Ragnarok Online klasik menggunakan teknik grafis 3D modern.*

---

```
██████╗  ██████╗  ██████╗██╗  ██╗ █████╗ ██████╗ 
██╔══██╗██╔═══██╗██╔════╝██║  ██║██╔══██╗██╔══██╗
██████╔╝██║   ██║██║     ███████║███████║██████╔╝
██╔══██╗██║   ██║██║     ██╔══██║██╔══██║██╔══██╗
██║  ██║╚██████╔╝╚██████╗██║  ██║██║  ██║██║  ██║
╚═╝  ╚═╝ ╚═════╝  ╚═════╝╚═╝  ╚═╝╚═╝  ╚═╝╚═╝  ╚═╝
```

---

## ENGLISH

### What is ROCHAR?

ROCHAR is a comprehensive Claude Code skill that transforms AI assistance into a **master-level 3D character engineering consultant** for Ragnarok Online reconstruction. It is not a simple prompt template — it is a complete knowledge system with 33 technical modules covering every aspect of RO character creation, from visual DNA analysis to production-ready GLSL shaders.

ROCHAR knows:
- The exact pixel-art visual language of classic RO characters
- Mathematically correct billboard rendering (foot-anchored, Y-locked cylindrical)
- Complete GLSL 330 Core shader library for character, world, transparency, and pixel-style rendering
- Blender Python automation scripts for modeling, UV, rigging, and animation export
- Sprite sheet architecture with 8-direction facing logic and animation state machines
- RO proprietary file format internals (RSM, GND, RSW, SPR, ACT, STR)
- 100-point quality assurance rubric with automated scoring

### Key Features

- **33 Technical Modules** — no filler, every module has working code
- **Two Reconstruction Pathways** — Pathway A (sprite-based) and Pathway B (3D model)
- **Complete GLSL Shaders** — vertex + fragment shaders ready to paste into Three.js
- **Blender Python Scripts** — procedural armature, UV mapping, animation baking
- **Animation State Machine** — 11 required states, 8-direction facing, event system
- **Quality Gate** — 100-point QA rubric, minimum 90 to pass
- **Troubleshooting Guide** — symptom → cause → fix for every common issue

---

## BAHASA INDONESIA

### Apa itu ROCHAR?

ROCHAR adalah Claude Code skill komprehensif yang mengubah AI menjadi **konsultan rekonstruksi karakter Ragnarok Online kelas master**. Bukan template sederhana — ini adalah sistem pengetahuan lengkap dengan 33 modul teknis yang mencakup seluruh aspek pembuatan karakter RO, dari analisis visual DNA hingga GLSL shader siap produksi.

ROCHAR menguasai:
- Bahasa visual pixel-art karakter RO klasik yang otentik
- Matematika billboard rendering yang benar secara akurat (foot-anchored, cylindrical Y-locked)
- Library GLSL 330 Core lengkap untuk karakter, dunia, transparansi, dan rendering pixel-style
- Script Python Blender untuk modeling, UV, rigging, dan ekspor animasi otomatis
- Arsitektur sprite sheet dengan logika facing 8 arah dan state machine animasi
- Internal format file proprietary RO (RSM, GND, RSW, SPR, ACT, STR)
- Rubrik quality assurance 100 poin dengan scoring otomatis

### Fitur Utama

- **33 Modul Teknis** — tidak ada filler, setiap modul berisi kode yang bekerja
- **Dua Pathway Rekonstruksi** — Pathway A (berbasis sprite) dan Pathway B (model 3D)
- **GLSL Shader Lengkap** — vertex + fragment shader siap dipaste ke Three.js
- **Script Python Blender** — armature prosedural, UV mapping, baking animasi
- **State Machine Animasi** — 11 state wajib, facing 8 arah, sistem event
- **Quality Gate** — rubrik QA 100 poin, minimum 90 untuk lulus
- **Panduan Troubleshooting** — simtom → penyebab → solusi untuk setiap masalah umum

---

## Module Index

### References (09 modules)
| # | Module | Description |
|---|--------|-------------|
| 01 | [RO Visual DNA](references/01_ro_visual_dna.md) | Core visual language: lighting formula, texture characteristics, zone palettes |
| 02 | [Character Proportion System](references/02_character_proportion_system.md) | H:B ratios, measurement methodology, proportion grids |
| 03 | [RO Character Anatomy](references/03_ro_character_anatomy.md) | Body segment construction, polygon allocation, anatomy validation |
| 04 | [Class and Character Archetypes](references/04_class_and_character_archetypes.md) | All major classes, color palettes, silhouette profiles, decision trees |
| 05 | [Costume and Equipment Architecture](references/05_costume_and_equipment_architecture.md) | 10-slot equipment system, transform hierarchy, JSON manifest |
| 06 | [Hair and Accessory Engine](references/06_hair_and_accessory_engine.md) | 3-layer hair mesh, GLSL hair material, socket positions |
| 07 | [Color and Material Language](references/07_color_and_material_language.md) | Skin tone palette, class color associations, baked light rules |
| 08 | [Sprite Visual Identity](references/08_sprite_visual_identity.md) | 8-direction system, animation state library, pivot system |
| 09 | [Character Reference Analysis](references/09_character_reference_analysis.md) | 8-phase analysis workflow, specification template |

### Modeling (06 modules)
| # | Module | Description |
|---|--------|-------------|
| 10 | [Low Poly Modeling](modeling/10_low_poly_modeling.md) | Polygon budgets, box modeling stages, LOD strategy |
| 11 | [Topology and Mesh Engineering](modeling/11_topology_and_mesh_engineering.md) | Edge flow, N-gon elimination, non-manifold detection |
| 12 | [UV Mapping and Texture Pipeline](modeling/12_uv_mapping_and_texture_pipeline.md) | Atlas organization, pixel density formula, lightmap UV2 |
| 13 | [Modular Character Construction](modeling/13_modular_character_construction.md) | Attachment points, ModularCharacter JS class, equipment manifest |
| 14 | [Blender Procedural Modeling](modeling/14_blender_procedural_modeling.md) | Full Python scripts for mesh, material, UV, armature, export |
| 15 | [Character Export Pipeline](modeling/15_character_export_pipeline.md) | GLB/FBX/OBJ export settings, coordinate system reference |

### Sprites (04 modules)
| # | Module | Description |
|---|--------|-------------|
| 16 | [Sprite Reconstruction](sprites/16_sprite_reconstruction.md) | Authorized sources, isolation workflow, magenta_to_alpha Python |
| 17 | [Sprite Sheet Architecture](sprites/17_sprite_sheet_architecture.md) | Atlas layouts, metadata JSON format, UV calculation |
| 18 | [Billboard Rendering](sprites/18_billboard_rendering.md) | Billboard mathematics, foot anchor, Three.js implementation |
| 19 | [Sprite Animation System](sprites/19_sprite_animation_system.md) | State machine, 8-direction logic, frame controller, events |

### Shaders (06 modules)
| # | Module | Description |
|---|--------|-------------|
| 20 | [GLSL Character Shader](shaders/20_glsl_character_shader.md) | Complete vertex+fragment shader, RO lighting formula |
| 21 | [GLSL World and Lightmap](shaders/21_glsl_world_and_lightmap.md) | Dual-UV terrain, lightmap compositing, water shader |
| 22 | [GLSL Billboard Shader](shaders/22_glsl_billboard_shader.md) | Cylindrical billboard, frame UV, shadow variant |
| 23 | [GLSL Transparency and Chroma Key](shaders/23_glsl_transparency_and_chroma_key.md) | Alpha test, blend, chroma key, additive blend modes |
| 24 | [GLSL Pixel Style Rendering](shaders/24_glsl_pixel_style_rendering.md) | Nearest-neighbor, palette quantization, scanlines |
| 25 | [Shader Optimization](shaders/25_shader_optimization.md) | Draw call reduction, instancing, precision, benchmarks |

### Animation (03 modules)
| # | Module | Description |
|---|--------|-------------|
| 26 | [Character Rigging](animation/26_character_rigging.md) | Full armature hierarchy, Blender Python, IK chains, weight painting |
| 27 | [Skeletal Animation](animation/27_skeletal_animation.md) | NLA organization, all 11 states, Three.js AnimationMixer |
| 28 | [Animation Validation](animation/28_animation_validation.md) | Foot skating detection, timing validation, visual regression |

### Integration (04 modules)
| # | Module | Description |
|---|--------|-------------|
| 29 | [Ragnarok Asset Formats](integration/29_ragnarok_asset_formats.md) | RSM, GND, RSW, SPR, ACT, STR format internals |
| 30 | [Three.js and OpenGL Integration](integration/30_threejs_and_opengl_integration.md) | ROCharacterScene class, texture config, render loop |
| 31 | [Character Pipeline Orchestration](integration/31_character_pipeline_orchestration.md) | A/B pathways, state machine, batch processing, time estimates |
| 32 | [Quality Assurance](integration/32_quality_assurance.md) | 100-point rubric, automated scorer, all 10 categories |

### Troubleshooting
| # | Module | Description |
|---|--------|-------------|
| 33 | [Troubleshooting and Diagnostics](33_troubleshooting_and_diagnostics.md) | Symptom/cause/fix table, debug shaders, WebGL debug setup |

---

## Quick Start

### Install ROCHAR Skill (Claude Code)

```bash
# Option A: Install from directory
# Place this ROCHAR/ folder in your Claude Code skills directory

# Option B: Reference in conversation
# Tell Claude: "Use the ROCHAR skill at ~/Downloads/ROCHAR/SKILL.md"
```

### Usage Examples

```
"ROCHAR: Analyze Knight class character proportions"
"ROCHAR: Write a GLSL character shader for a Mage character"
"ROCHAR: Set up sprite atlas for 8-direction walk animation"
"ROCHAR: Validate my character against QA rubric"
"ROCHAR: Debug my billboard shader — character is floating"
```

---

## Requirements

| Tool | Purpose | Required |
|------|---------|----------|
| Three.js r150+ | 3D scene rendering | Yes (Pathway B + Sprites) |
| Blender 3.0+ | 3D modeling and animation | Yes (Pathway B) |
| Python 3.8+ | Sprite processing scripts | Yes (Pathway A) |
| PIL/Pillow | Image processing | Yes (Pathway A) |
| NumPy | Array operations in sprite tools | Yes (Pathway A) |
| gltf-validator | GLB format validation | Recommended |

---

## License

Copyright © 2024 VeryCoolApps (PT. Agra Karya Digital)  
All Rights Reserved.

See [LICENSE](LICENSE) for full terms.

---

## Credits

**ROCHAR** adalah karya orisinal VeryCoolApps (PT. Agra Karya Digital).

Dibuat dengan ❤️ untuk komunitas kreator yang mencintai Ragnarok Online klasik  
dan ingin menghidupkannya kembali menggunakan teknologi grafis modern.

*"Prontera masih ada di hati kita."*

---

*ROCHAR v1.0 — Claude Code Skill*  
*33 Technical Modules | Bilingual EN/ID*
