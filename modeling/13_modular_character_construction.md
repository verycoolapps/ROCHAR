# 13 — MODULAR CHARACTER CONSTRUCTION
## Interchangeable Equipment System and Component Architecture

> **Purpose:** Define modular architecture for equipment swaps, attachment points, and component organization.  
> **Related:** [../references/05_costume_and_equipment_architecture.md](../references/05_costume_and_equipment_architecture.md), [14_blender_procedural_modeling.md](14_blender_procedural_modeling.md)

---

## 13.1 MODULAR SYSTEM PRINCIPLES

The modular character system allows:
- Swapping any equipment slot independently
- Reusing the same base body for different classes
- Combining equipment from different sources
- Dynamic equipment changes at runtime

**Rule:** The base body mesh must work visually with **zero equipment** equipped.  
Equipment adds to it; nothing should break when equipment is absent.

---

## 13.2 BASE MESH ARCHITECTURE

```
BASE MESH LAYERS (always rendered):
  Layer 0: Nude base body
    → Provides skin geometry under all equipment
    → Full body, full UV, full detail
    → Polygon budget: 500–800 tris

EQUIPMENT VISIBILITY STATES:
  When ARMOR equipped:
    → Base torso region: hidden (disable rendering)
    → Equipment torso: shown
    → Other base body regions: visible (arms, neck, etc.)
  
  When FOOTGEAR equipped:
    → Base feet: hidden
    → Equipment feet: shown
  
  Implementation: Use mesh vertex groups + shape keys, OR
  separate mesh objects with visibility toggling
```

---

## 13.3 ATTACHMENT POINT NAMING CONVENTION

All equipment attachment bones follow this convention:

```
FORMAT: [Category]_[SubType]_[Side?]_Socket

HEADGEAR:
  HeadgearTop_Socket
  HeadgearMid_Socket
  HeadgearBot_Socket

WEAPONS:
  Weapon_Main_Socket          (right hand, primary weapon)
  Weapon_Off_Socket           (left hand, off-hand/shield)
  Weapon_Back_Socket          (weapons worn on back)

BODY:
  Armor_Chest_Socket          (body armor anchor)
  Garment_Back_Socket         (cape/wing anchor)
  Garment_Collar_Socket       (collar attachment)

LEGS:
  Footgear_L_Socket           (left boot/shoe)
  Footgear_R_Socket           (right boot/shoe)

ACCESSORIES:
  Accessory_L_Socket          (left wrist/ring)
  Accessory_R_Socket          (right wrist/ring)
  
SPECIAL:
  Effect_L_Socket             (left-hand effects)
  Effect_R_Socket             (right-hand effects)
  Effect_Spine_Socket         (back effects, aura)
```

---

## 13.4 BLENDER COLLECTION ORGANIZATION

```
ROCHAR_Character/
├── BASE/
│   ├── [Character]_Body          ← Base mesh
│   ├── [Character]_Hair_Base     ← Base hair (always present)
│   └── [Character]_Armature      ← Skeleton
│
├── EQUIPMENT/
│   ├── HEAD_TOP/
│   │   ├── [Item]_HeadgearTop
│   │   └── [Item2]_HeadgearTop
│   ├── HEAD_MID/
│   │   └── [Item]_HeadgearMid
│   ├── ARMOR/
│   │   ├── [Item]_Armor
│   │   └── [Item2]_Armor
│   ├── GARMENT/
│   │   └── [Item]_Garment
│   ├── FOOTGEAR/
│   │   └── [Item]_Footgear
│   ├── WEAPON/
│   │   └── [Item]_Weapon
│   └── SHIELD/
│       └── [Item]_Shield
│
└── EFFECTS/
    ├── [Item]_Effect_L
    └── [Item]_Effect_R
```

---

## 13.5 EQUIPMENT VISIBILITY MANAGEMENT

### Blender Approach (Design Time)

```python
# Toggle equipment visibility in Blender viewport
import bpy

def set_equipment_slot(character_collection, slot_name, item_name):
    """Show only the specified item in a slot collection."""
    slot_col = character_collection.children.get(slot_name)
    if not slot_col:
        return
    
    for obj in slot_col.all_objects:
        # Hide all items in this slot
        obj.hide_viewport = True
        obj.hide_render = True
    
    # Show only the requested item
    target = slot_col.all_objects.get(item_name)
    if target:
        target.hide_viewport = False
        target.hide_render = False

# Example usage:
# set_equipment_slot(bpy.data.collections['EQUIPMENT'], 
#                   'HEAD_TOP', 'TopHat_HeadgearTop')
```

### Three.js Approach (Runtime)

```javascript
class ModularCharacter {
    constructor(baseMesh, armature) {
        this.baseMesh = baseMesh;
        this.armature = armature;
        this.slots = {
            HEAD_TOP: null,
            HEAD_MID: null,
            ARMOR: null,
            GARMENT: null,
            FOOTGEAR: null,
            WEAPON: null,
            SHIELD: null
        };
        this.hiddenBaseParts = new Set();
    }
    
    equip(slotName, equipmentMesh, hideBaseParts = []) {
        // Unequip current item in slot
        this.unequip(slotName);
        
        // Attach new equipment to armature socket bone
        const socketBone = this.getSocketBone(slotName);
        if (socketBone) {
            socketBone.add(equipmentMesh);
        }
        
        this.slots[slotName] = equipmentMesh;
        
        // Hide base mesh parts covered by equipment
        hideBaseParts.forEach(part => {
            this.hideBasePart(part);
            this.hiddenBaseParts.add(part);
        });
    }
    
    unequip(slotName) {
        const current = this.slots[slotName];
        if (current) {
            current.parent?.remove(current);
            this.slots[slotName] = null;
        }
        // Restore hidden base parts if no other equipment hides them
        this.restoreHiddenParts();
    }
    
    getSocketBone(slotName) {
        const socketMap = {
            'HEAD_TOP': 'HeadgearTop_Socket',
            'HEAD_MID': 'HeadgearMid_Socket',
            'ARMOR':    'Armor_Chest_Socket',
            'WEAPON':   'Weapon_Main_Socket',
            'SHIELD':   'Weapon_Off_Socket',
            'GARMENT':  'Garment_Back_Socket',
            'FOOTGEAR': null  // handled separately per foot
        };
        const boneName = socketMap[slotName];
        return boneName ? this.armature.getBoneByName(boneName) : null;
    }
    
    hideBasePart(partName) {
        const part = this.baseMesh.getObjectByName(partName);
        if (part) part.visible = false;
    }
    
    restoreHiddenParts() {
        this.hiddenBaseParts.forEach(part => {
            // Check if any current equipment still needs this hidden
            const stillNeeded = Object.values(this.slots).some(eq => 
                eq && eq.userData.hides?.includes(part)
            );
            if (!stillNeeded) {
                const meshPart = this.baseMesh.getObjectByName(part);
                if (meshPart) meshPart.visible = true;
                this.hiddenBaseParts.delete(part);
            }
        });
    }
}
```

---

## 13.6 JSON EQUIPMENT MANIFEST FORMAT

Complete format for defining a character equipment set:

```json
{
  "schema_version": "1.0",
  "character_id": "knight_male_01",
  "character_base": {
    "mesh": "meshes/base/human_male_base.glb",
    "armature": "meshes/base/human_male_armature.glb",
    "default_skin": "textures/skins/skin_fair.png",
    "hair_style": "hair_short_spiked"
  },
  "slots": {
    "HEAD_TOP": {
      "item_id": "helm_iron_full",
      "mesh": "equipment/helms/iron_full.glb",
      "attachment": "HeadgearTop_Socket",
      "offsets": {
        "position": [0, 0, 0],
        "rotation": [0, 0, 0],
        "scale": [1, 1, 1]
      },
      "hides_base": ["head_hair", "head_ears"],
      "material": "materials/iron_01.json"
    },
    "ARMOR": {
      "item_id": "armor_plate_iron_full",
      "mesh": "equipment/armor/plate_iron_full.glb",
      "attachment": "Armor_Chest_Socket",
      "offsets": {"position": [0, 0, 0], "rotation": [0, 0, 0], "scale": [1, 1, 1]},
      "hides_base": ["torso_front", "torso_back", "shoulder_l", "shoulder_r"],
      "material": "materials/iron_01.json"
    },
    "WEAPON": {
      "item_id": "sword_one_hand_iron",
      "mesh": "equipment/weapons/sword_1h_iron.glb",
      "attachment": "Weapon_Main_Socket",
      "offsets": {
        "position": [0, 0.05, 0],
        "rotation": [0, 0, 0],
        "scale": [1, 1, 1]
      },
      "hides_base": [],
      "material": "materials/iron_01.json"
    },
    "SHIELD": {
      "item_id": "shield_iron_round",
      "mesh": "equipment/shields/round_iron.glb",
      "attachment": "Weapon_Off_Socket",
      "offsets": {
        "position": [0, 0.3, 0.05],
        "rotation": [0, 0, 0],
        "scale": [1, 1, 1]
      },
      "hides_base": [],
      "material": "materials/iron_01.json"
    },
    "GARMENT": null,
    "FOOTGEAR": {
      "item_id": "boots_iron_ankle",
      "mesh": "equipment/footgear/boots_iron_ankle.glb",
      "attachment_L": "Footgear_L_Socket",
      "attachment_R": "Footgear_R_Socket",
      "offsets": {"position": [0, 0, 0], "rotation": [0, 0, 0], "scale": [1, 1, 1]},
      "hides_base": ["feet_l", "feet_r"],
      "material": "materials/iron_01.json"
    },
    "HEAD_MID": null,
    "HEAD_BOT": null,
    "ACCESSORY_L": null,
    "ACCESSORY_R": null
  }
}
```

---

## 13.7 BASE MESH VERTEX GROUP MAP

For modular hiding of base mesh parts, the base mesh uses vertex groups:

```
Vertex Groups (Blender naming):
  head_scalp          — top of head, hidden by HEAD_TOP full helms
  head_ears           — ear geometry
  head_face           — face — rarely hidden (only full mask)
  head_hair           — hair geometry — hidden by full helms
  
  torso_front         — chest/belly front faces
  torso_back          — back faces
  shoulder_l          — left shoulder area
  shoulder_r          — right shoulder area
  torso_full          — all torso (use for full plate armor)
  
  arm_upper_l         — left upper arm
  arm_upper_r         — right upper arm
  arm_lower_l         — left forearm
  arm_lower_r         — right forearm
  
  hand_l              — left hand
  hand_r              — right hand
  
  leg_upper_l         — left thigh
  leg_upper_r         — right thigh
  leg_lower_l         — left shin
  leg_lower_r         — right shin
  
  feet_l              — left foot
  feet_r              — right foot
```

Each equipment piece specifies which vertex groups to hide in its manifest.
