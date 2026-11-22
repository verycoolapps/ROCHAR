# 19 — SPRITE ANIMATION SYSTEM
## State Machine, Direction Logic, and Frame Controller

> **Purpose:** Define animation state machine, directional logic, timing, and event system.  
> **Related:** [17_sprite_sheet_architecture.md](17_sprite_sheet_architecture.md), [18_billboard_rendering.md](18_billboard_rendering.md)

---

## 19.1 ANIMATION STATE MACHINE

```
                    ┌─────────────────────────────┐
                    │                             │
         ┌──────────▼──────────┐                 │
         │        IDLE          │◄────────────────┤
         │  (loop, low priority)│                 │
         └──────────┬──────────┘                 │
                    │                             │
          direction input                   animation end
                    │                             │
         ┌──────────▼──────────┐                 │
         │        WALK          │                 │
         │  (loop, mid priority)│                 │
         └──────────┬──────────┘                 │
                    │                             │
            attack input                         │
                    │                             │
         ┌──────────▼──────────┐                 │
         │       ATTACK         │─────────────────┘
         │(oneshot, high prior) │
         └──────────┬──────────┘
                    │
             damage received
                    │
         ┌──────────▼──────────┐
         │        HURT          │─────► IDLE (after hurt finishes)
         │(oneshot, very high)  │
         └──────────┬──────────┘
                    │
                HP = 0
                    │
         ┌──────────▼──────────┐
         │        DEAD          │
         │(oneshot, highest)    │
         │  holds last frame    │
         └────────────────────-─┘
```

---

## 19.2 ANIMATION STATE DEFINITIONS

```javascript
const ANIMATION_STATES = {
    IDLE: {
        id: 'idle',
        loop: true,
        priority: 0,
        canInterrupt: true,
        interruptedBy: ['walk', 'run', 'attack_1h', 'attack_2h', 'cast', 
                        'skill', 'hurt', 'dead', 'sit', 'pickup']
    },
    WALK: {
        id: 'walk',
        loop: true,
        priority: 1,
        canInterrupt: true,
        interruptedBy: ['attack_1h', 'attack_2h', 'cast', 'skill', 
                        'hurt', 'dead']
    },
    SIT: {
        id: 'sit',
        loop: true,
        priority: 0,
        canInterrupt: true,
        interruptedBy: ['walk', 'attack_1h', 'hurt', 'dead']
    },
    ATTACK_1H: {
        id: 'attack_1h',
        loop: false,
        priority: 5,
        canInterrupt: false,
        interruptedBy: ['hurt', 'dead'],
        onComplete: 'return_to_idle'
    },
    CAST: {
        id: 'cast',
        loop: false,
        priority: 5,
        canInterrupt: false,
        interruptedBy: ['hurt', 'dead'],
        onComplete: 'return_to_idle'
    },
    HURT: {
        id: 'hurt',
        loop: false,
        priority: 8,
        canInterrupt: false,
        interruptedBy: ['dead'],
        onComplete: 'return_to_previous'
    },
    DEAD: {
        id: 'dead',
        loop: false,
        holdLastFrame: true,
        priority: 10,
        canInterrupt: false,
        interruptedBy: []  // Nothing interrupts death
    }
};
```

---

## 19.3 8-DIRECTION FACING LOGIC

```javascript
/**
 * Convert a 2D movement vector to one of 8 sprite directions.
 * 
 * @param {number} dx - X component of movement (negative = left)
 * @param {number} dz - Z component of movement (negative = north/away)
 * @returns {string} Direction: 'S', 'SW', 'W', 'NW', 'N', 'NE', 'E', 'SE'
 */
function movementToDirection(dx, dz) {
    // No movement → maintain current direction
    if (dx === 0 && dz === 0) return null;
    
    // Calculate angle: 0° = East, 90° = South, 180° = West, 270° = North
    // We use atan2(dz, dx) but need to map to our compass
    // dz positive = moving toward camera (south), negative = away (north)
    
    let angle = Math.atan2(dz, dx) * (180 / Math.PI);
    // atan2 returns: East=0, North=90, West=180/-180, South=-90
    // Convert to compass: North=0, East=90, South=180, West=270
    
    // Map to 8 directions (each covers 45° sector):
    const sector = Math.round(angle / 45);  // 0-7
    
    const directions = ['E', 'SE', 'S', 'SW', 'W', 'NW', 'N', 'NE'];
    return directions[((sector % 8) + 8) % 8];
}

/**
 * Convert camera-relative facing direction to sprite direction.
 * Accounts for camera rotation in 3D world.
 * 
 * @param {THREE.Vector3} characterPos
 * @param {THREE.Camera} camera
 * @param {number} characterFacingAngle - radians in XZ plane
 * @returns {string} One of 8 compass directions
 */
function getSpriteFacingDirection(characterPos, camera, characterFacingAngle) {
    // Get camera forward in XZ plane (ignore Y)
    const camDir = new THREE.Vector3();
    camera.getWorldDirection(camDir);
    camDir.y = 0;
    camDir.normalize();
    
    // Character facing direction as vector
    const charDir = new THREE.Vector3(
        Math.sin(characterFacingAngle), 
        0, 
        Math.cos(characterFacingAngle)
    );
    
    // Angle between camera forward and character facing (in XZ plane)
    let angle = Math.atan2(
        charDir.x * camDir.z - charDir.z * camDir.x,  // cross product Y
        charDir.dot(camDir)                              // dot product
    );
    
    // Convert to 8-direction index
    angle = ((angle * 180 / Math.PI) + 360) % 360;
    const dirIndex = Math.round(angle / 45) % 8;
    
    // Direction map (from camera's perspective)
    const directions = ['S', 'SE', 'E', 'NE', 'N', 'NW', 'W', 'SW'];
    return directions[dirIndex];
}
```

---

## 19.4 FRAME TIMER AND PLAYBACK CONTROLLER

```javascript
class SpriteAnimationController {
    constructor(metadata) {
        this.metadata = metadata;
        this.currentState = 'idle';
        this.currentDirection = 'S';
        this.currentFrameIndex = 0;
        this.frameTimer = 0;
        this.frameCount = 0;
        this.onAnimationComplete = null;
        this.events = {};  // frame event callbacks
        this._stateStack = [];  // for return_to_previous
    }
    
    /**
     * Play animation state. Respects priority system.
     */
    play(stateName, direction = null, force = false) {
        const newState = ANIMATION_STATES[stateName.toUpperCase()];
        const curState = ANIMATION_STATES[this.currentState.toUpperCase()];
        
        if (!newState) {
            console.warn(`[ROCHAR] Unknown animation state: ${stateName}`);
            return false;
        }
        
        // Check priority
        if (!force && curState && newState.priority <= curState.priority) {
            // Check if current state CAN be interrupted by new state
            if (!curState.canInterrupt || 
                !curState.interruptedBy.includes(stateName.toLowerCase())) {
                return false;  // Cannot interrupt
            }
        }
        
        // Save state for 'return_to_previous'
        if (this.currentState !== stateName) {
            this._stateStack.push(this.currentState);
        }
        
        this.currentState = stateName.toLowerCase();
        if (direction) this.currentDirection = direction;
        this.currentFrameIndex = 0;
        this.frameTimer = 0;
        
        // Get frame data for this state/direction
        const animDef = this.metadata.animations[this.currentState];
        this.frameCount = animDef ? animDef.frame_count : 1;
        
        return true;
    }
    
    /**
     * Update animation timer. Call once per frame.
     * @param {number} deltaTime - Seconds since last update
     */
    update(deltaTime) {
        const animDef = this.metadata.animations[this.currentState];
        if (!animDef) return this.currentFrameIndex;
        
        const frameDuration = 1.0 / animDef.fps;
        this.frameTimer += deltaTime;
        
        while (this.frameTimer >= frameDuration) {
            this.frameTimer -= frameDuration;
            this._advanceFrame(animDef);
        }
        
        return this.getCurrentSpriteFrame();
    }
    
    _advanceFrame(animDef) {
        this.currentFrameIndex++;
        
        // Check for frame events
        if (animDef.events) {
            const event = animDef.events.find(e => e.frame === this.currentFrameIndex);
            if (event && this.events[event.event]) {
                this.events[event.event](event);
            }
        }
        
        // Handle end of animation
        if (this.currentFrameIndex >= animDef.frame_count) {
            if (animDef.loop) {
                this.currentFrameIndex = 0;
            } else if (animDef.holdLastFrame) {
                this.currentFrameIndex = animDef.frame_count - 1;
                // Stay on last frame forever
            } else {
                this.currentFrameIndex = 0;
                // Return to previous state
                const onComplete = ANIMATION_STATES[this.currentState.toUpperCase()]?.onComplete;
                if (onComplete === 'return_to_idle') {
                    this.play('idle');
                } else if (onComplete === 'return_to_previous') {
                    const prev = this._stateStack.pop() || 'idle';
                    this.play(prev);
                }
                if (this.onAnimationComplete) {
                    this.onAnimationComplete(this.currentState);
                }
            }
        }
    }
    
    getCurrentSpriteFrame() {
        const animDef = this.metadata.animations[this.currentState];
        if (!animDef) return 0;
        
        // Get direction data, falling back to 'S' if direction not in atlas
        const dirData = animDef.directions?.[this.currentDirection] 
                     || animDef.directions?.['S']
                     || { frames: [0] };
        
        const frameArrayIndex = Math.min(
            this.currentFrameIndex, 
            dirData.frames.length - 1
        );
        
        return dirData.frames[frameArrayIndex];
    }
    
    // Register frame event callbacks
    on(eventName, callback) {
        this.events[eventName] = callback;
    }
    
    setDirection(direction) {
        this.currentDirection = direction;
    }
}
```

---

## 19.5 ANIMATION TRANSITION RULES

```
GENERAL TRANSITION TABLE:
  IDLE    → WALK    : on movement input
  WALK    → IDLE    : on no movement (after current walk step complete)
  IDLE    → ATTACK  : on attack input (immediate)
  WALK    → ATTACK  : on attack input (immediate)
  ATTACK  → IDLE    : on animation complete
  IDLE    → HURT    : on damage received (immediate)
  WALK    → HURT    : on damage received (immediate)
  ATTACK  → HURT    : on damage received (immediate if HURT.priority > ATTACK.priority)
  HURT    → IDLE    : on animation complete
  ANY     → DEAD    : on HP reaches 0 (highest priority)
  DEAD    →  —      : terminal state (no exit)

TIMING CONSIDERATION:
  Walk cycle: frame advances trigger footstep sounds at mid-step frame
  Attack:     "hit frame" triggers hit detection (see events in metadata)
  Cast:       "spell frame" triggers effect spawn
```

---

## 19.6 WALK CYCLE FOOT TIMING

```
8-FRAME WALK CYCLE STRUCTURE:
  Frame 0: Both feet neutral (contact phase)
  Frame 1: Left foot forward (down contact)
  Frame 2: Left foot up slightly, right foot back
  Frame 3: Left foot at mid-stride
  Frame 4: Right foot forward (contact phase)
  Frame 5: Right foot down (contact)
  Frame 6: Right foot up slightly, left foot back
  Frame 7: Right foot at mid-stride

FOOTSTEP EVENTS:
  Frame 1: LEFT_FOOTSTEP event
  Frame 5: RIGHT_FOOTSTEP event

WALK SPEED RELATIONSHIP:
  Character moves: (worldSpeedMetersPerSecond) each second
  Animation FPS:   8 frames per second
  Stride length:   worldSpeed / (animFPS / 2) = worldSpeed / 4
  
  For natural-looking walk:
  → Character moves 0.08m per frame
  → At 8 FPS = 0.64 m/s walk speed
```
