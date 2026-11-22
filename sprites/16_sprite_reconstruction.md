# 16 — SPRITE RECONSTRUCTION
## Authorized Sprite Recreation Workflow

> **Purpose:** Define workflow for recreating RO-style sprites using authorized references.  
> **Related:** [17_sprite_sheet_architecture.md](17_sprite_sheet_architecture.md), [../references/08_sprite_visual_identity.md](../references/08_sprite_visual_identity.md)

---

## 16.1 AUTHORIZED SOURCES ONLY

**ROCHAR sprite reconstruction works only with authorized materials:**
- Original character designs you created
- Licensed assets you have permission to recreate
- Fan art/original creations inspired by RO visual style
- Assets from open-source or public domain projects

Do NOT extract sprites directly from copyrighted game files for redistribution.  
If working with original game files for personal research, clearly label all results as derived work.

---

## 16.2 SPRITE ISOLATION WORKFLOW

When working from a screenshot or reference image:

```
STEP 1: IMPORT REFERENCE
  → Open reference in image editor (Krita, GIMP, Photoshop)
  → Identify sprite bounds (smallest rectangle containing character)

STEP 2: ISOLATE CHARACTER
  Option A (magenta background):
    → Use "Select by Color" with tolerance 30–40
    → Select magenta (#FF00FF) background
    → Delete selected region
    
  Option B (mixed background):
    → Manual selection with polygon lasso tool
    → Trace character outline carefully
    → Feather: 0 pixels (hard edge for pixel art)

STEP 3: CLEAN EDGES
  → Inspect character outline at 400% zoom
  → Remove fringe pixels (semi-transparent remnants of background)
  → For pixel art: each pixel should be either fully opaque or fully transparent
  → Use pencil tool (hard edges) not brush tool (anti-aliased)

STEP 4: VERIFY ISOLATION
  → Place on checkerboard background (indicates transparency)
  → Check all corners and complex areas (hair tips, weapon edges)
  → Character outline should be clean, pixel-perfect
```

---

## 16.3 COLOR PALETTE NORMALIZATION

After isolation, normalize the color palette:

```
STEP 1: IDENTIFY UNIQUE COLORS
  → Image → Color Count / Color Palette tool
  → For pixel art: typically 8–32 unique colors per sprite

STEP 2: EXAMINE PALETTE
  → Sort by frequency (most used to least)
  → Group similar colors (skin tones, clothing colors, metal tones)

STEP 3: NORMALIZE INCONSISTENCIES
  → Reference screenshots may have JPEG artifacts
  → Small color variations that should be one color: unify them
  → Example: skin tone #E8B88A and #E9B98B → both become #E8B88A

STEP 4: DOCUMENT PALETTE
  → Create a palette swatch (1px × 1px tiles of each color)
  → Save palette as GIMP/Krita palette file
  → Use for all subsequent frames of the same character
```

---

## 16.4 SCALE CALIBRATION

Determine the correct character scale for the sprite system:

```
REFERENCE METHOD:
  → Find a reference with known world dimensions
    (e.g., a tile that is 1 world unit = known visual size)
  → Measure tile size in pixels at game distance
  → Calculate pixels per world unit

DIRECT METHOD:
  → Measure character height in pixels in reference
  → Define: 1 head unit (H) = measured value in pixels
  → This becomes the scale reference for all proportions

RECONSTRUCTION TARGET:
  → Decide output resolution
  → Standard: 64×96px body sprite (matches classic RO feel)
  → HD: 128×192px body sprite (2× for modern displays)
  → Keep power-of-two dimensions

SCALING DOWN FROM HIGH-RES:
  → Always scale down using nearest-neighbor algorithm
  → NEVER use bilinear/bicubic (blurs pixel art)
  → In Photoshop: Image Size > Resample: Nearest Neighbor
  → In GIMP: Scale > Interpolation: None
```

---

## 16.5 ALPHA CHANNEL RECONSTRUCTION FROM MAGENTA

Converting magenta-transparent sprites to alpha-channel PNG:

```python
"""
ROCHAR — Magenta to Alpha Converter
Converts classic RO magenta transparency to PNG alpha channel.
"""
from PIL import Image
import numpy as np

def magenta_to_alpha(input_path, output_path, 
                      tolerance=30, 
                      edge_smooth=False):
    """
    Convert magenta (#FF00FF) background to alpha transparency.
    
    tolerance: color distance threshold for magenta detection (0-255)
    edge_smooth: attempt to smooth edges (may cause fringing)
    """
    img = Image.open(input_path).convert('RGBA')
    data = np.array(img, dtype=float)
    
    r, g, b, a = data[:,:,0], data[:,:,1], data[:,:,2], data[:,:,3]
    
    # Magenta detection: high R, low G, high B
    # Distance from pure magenta (255, 0, 255)
    dist_from_magenta = np.sqrt(
        (r - 255)**2 + 
        (g - 0)**2 + 
        (b - 255)**2
    )
    
    # Create mask: pixels within tolerance of magenta → transparent
    magenta_mask = dist_from_magenta < tolerance
    
    # Apply mask
    data[:,:,3] = np.where(magenta_mask, 0, 255).astype(np.uint8)
    
    result = Image.fromarray(data.astype(np.uint8), 'RGBA')
    result.save(output_path, 'PNG')
    
    # Count affected pixels
    transparent_count = np.sum(magenta_mask)
    total_pixels = img.width * img.height
    print(f"[ROCHAR] Converted: {transparent_count} pixels made transparent "
          f"({100*transparent_count/total_pixels:.1f}% of image)")
    
    return result

# USAGE:
# magenta_to_alpha('sprite_magenta.bmp', 'sprite_alpha.png', tolerance=25)
```

---

## 16.6 DIRECTION VARIANT EXTRACTION

Extracting individual direction sprites from a composite sprite sheet:

```python
"""
ROCHAR — Sprite Direction Extractor
Extracts individual directional frames from an RO-style sprite sheet.
"""
from PIL import Image
import json
import os

def extract_direction_sprites(
    sheet_path,
    frame_width, 
    frame_height,
    output_dir,
    metadata_path=None
):
    """
    Extract individual frames from a grid sprite sheet.
    
    sheet_path:   path to sprite sheet PNG
    frame_width:  width of each frame in pixels
    frame_height: height of each frame in pixels
    output_dir:   directory to save extracted frames
    metadata_path: optional JSON metadata file
    """
    sheet = Image.open(sheet_path).convert('RGBA')
    os.makedirs(output_dir, exist_ok=True)
    
    sheet_w, sheet_h = sheet.size
    cols = sheet_w // frame_width
    rows = sheet_h // frame_height
    
    frame_index = 0
    frames_info = []
    
    for row in range(rows):
        for col in range(cols):
            x = col * frame_width
            y = row * frame_height
            
            frame = sheet.crop((x, y, x + frame_width, y + frame_height))
            
            # Check if frame is empty (all transparent or all magenta)
            bbox = frame.getbbox()
            if bbox is None:
                continue  # Skip empty frames
            
            frame_name = f"frame_{frame_index:04d}.png"
            frame_path = os.path.join(output_dir, frame_name)
            frame.save(frame_path)
            
            frames_info.append({
                "index": frame_index,
                "file": frame_name,
                "row": row,
                "col": col,
                "content_bbox": list(bbox)
            })
            
            frame_index += 1
    
    # Save metadata
    if metadata_path:
        with open(metadata_path, 'w') as f:
            json.dump({
                "sheet": sheet_path,
                "frame_width": frame_width,
                "frame_height": frame_height,
                "total_frames": frame_index,
                "frames": frames_info
            }, f, indent=2)
    
    print(f"[ROCHAR] Extracted {frame_index} frames to {output_dir}")
    return frames_info
```

---

## 16.7 CONSISTENCY VALIDATION ACROSS FRAMES

After extracting/creating sprite frames, validate consistency:

```python
def validate_sprite_consistency(frames_dir, expected_color_tolerance=15):
    """
    Check color and scale consistency across sprite frames.
    Reports potential inconsistencies for manual review.
    """
    from PIL import Image
    import os
    
    frames = sorted([
        f for f in os.listdir(frames_dir) 
        if f.endswith('.png')
    ])
    
    if not frames:
        print("[ROCHAR] No PNG frames found.")
        return
    
    # Sample skin color from first frame
    first = Image.open(os.path.join(frames_dir, frames[0])).convert('RGBA')
    # Manual: define expected skin color from reference
    expected_skin = (232, 184, 138)  # #E8B88A warm tan
    
    issues = []
    
    for frame_name in frames:
        img = Image.open(os.path.join(frames_dir, frame_name)).convert('RGBA')
        
        # Check dimensions consistency
        if img.size != first.size:
            issues.append(f"{frame_name}: Size mismatch {img.size} vs {first.size}")
        
        # Check if frame has content
        bbox = img.getbbox()
        if bbox is None:
            issues.append(f"{frame_name}: Empty frame (all transparent)")
        
    if issues:
        print("[ROCHAR] Consistency issues found:")
        for issue in issues:
            print(f"  → {issue}")
    else:
        print(f"[ROCHAR] All {len(frames)} frames passed consistency check.")
    
    return issues
```

---

## 16.8 EDGE CLEANUP WORKFLOW

Cleaning edges on reconstructed sprites:

```
TOOL: Any pixel art editor (Aseprite recommended, or GIMP)

STEP 1: ZOOM TO 400–800%
  → Work at high zoom so individual pixels are visible

STEP 2: SCAN OUTLINE
  → Use pencil tool (1px, hard edge)
  → Trace the character outline
  → Remove any stray pixels outside the main outline

STEP 3: CHECK HAIR TIPS
  → Hair tips often have aliased pixels from JPEG compression
  → Replace graduated pixels with clean on/off

STEP 4: CHECK WEAPON EDGES
  → Thin weapons (swords, arrows) are most prone to edge issues
  → Validate that the weapon is fully connected, no floating pixels

STEP 5: VALIDATE AGAINST CHECKERBOARD
  → Always view result against transparency checkerboard
  → Any unexpected transparent areas inside the character → repaint
  → Any unexpected opaque pixels outside → erase
```
