# Chapter 15a: Cleanup

## What You'll Learn
- Fixing typos and inconsistencies that accumulated across chapters
- Documenting the system tick order and why it matters
- Auditing collision layers for correctness
- Organising the scene setup for clarity
- Preparing the codebase for TrenchBroom integration

---

## Why a Cleanup Chapter?

Over the last four chapters (12-15) we've added weapons, a player body, platforming, HUD, death/respawn, and damage feedback. Each chapter focused on getting features working, not on code hygiene. Before we start TrenchBroom integration (which restructures the level system), let's clean house.

This chapter has no new features — it's all about making the existing code correct, consistent, and well-documented.

---

## Fix 1: The LighteningGun Typo

The `WeaponType` enum has `LighteningGun` instead of `LightningGun`. This typo is in multiple files.

### `components.h`

```cpp
enum class WeaponType
{
    Shotgun,
    SuperShotgun,
    Nailgun,
    RocketLauncher,
    GrenadeLauncher,
    LightningGun,       // FIXED: was LighteningGun
    Railgun
};
```

### `weapon_definitions.h`

Update the switch case:

```cpp
case WeaponType::LightningGun:    // FIXED: was LighteningGun
    w.fireMode = FireMode::Hitscan;
    w.damage = 30.0f;
    w.fireRate = 0.05f;
    w.range = 15.0f;
    w.spread = 0.0f;
    w.pelletCount = 1;
    w.ammoPerShot = 1;
    break;
```

### `debug_hud_system.cpp`

Update the weapon name display:

```cpp
case WeaponType::LightningGun:    // FIXED: was LighteningGun
    weaponName = "Lightning Gun"; break;
```

Use your editor's find-and-replace to catch any other occurrences: search for `LighteningGun` and replace with `LightningGun`.

---

## Fix 2: The Nailgun Fire Mode

In `weapon_definitions.h`, the Nailgun has `fireMode = FireMode::Hitscan` but sets `projectileSpeed = 30.0f`. Quake's Nailgun fires physical nails — it should be a projectile weapon.

```cpp
case WeaponType::Nailgun:
    w.fireMode = FireMode::Projectile;  // FIXED: was Hitscan
    w.damage = 9.0f;
    w.fireRate = 0.1f;
    w.projectileSpeed = 30.0f;
    w.ammoPerShot = 1;
    break;
```

---

## Fix 3: Remove Debug Output

The combat system has commented-out debug prints. Remove them:

In `combat_system.cpp`, remove:

```cpp
// std::cout << "Hitscan fired" << std::endl;
```

Also remove the `#include <iostream>` if no other debug output remains in the file.

---

## Fix 4: Comment Typos

The codebase has several comment typos. Fix these while you're at it:

In `combat_system.cpp`:
- `// apply spreadh to a direction vector` → `// apply spread to a direction vector`
- `// fidn the closest entity hit by a ray` → `// find the closest entity hit by a ray`

In `collision_system.cpp`:
- `// 2 for each entity with velocity, sweep against nerby entities and level` → `// For each entity with velocity, sweep against nearby entities and level`
- `// trigger detected - capter 11 will handle this` → `// trigger detected — handled by trigger_system`

In `components.h`:
- `// attacked to projectile entities` → `// attached to projectile entities`

---

## Document: System Tick Order

The system tick order in `main.cpp` is critical and easy to break. Here's the documented version with explanations:

```cpp
while (fixedTimestep.step())
{
    // 1. Read weapon switch input (must happen before combat)
    weaponSwitchSystem(registry);

    // 2. Apply player input to velocity (jump, acceleration)
    //    Must be before physics so gravity/friction act on
    //    the velocity we just set
    playerMovementSystem(registry);

    // 3. Apply gravity and friction to all velocities
    //    Must be before collision so we check the velocity
    //    we'll actually move with
    physicsSystem(registry);

    // 4. Update doors, lifts, and other movers
    //    Must be before collision so moving platforms are
    //    in their new positions when we check
    moverSystem(registry);

    // 5. Resolve collisions: adjust velocities to prevent
    //    penetration. Uses swept AABB against level geometry
    //    and other entities
    collisionSystem(registry, spatialHash, level);

    // 6. Apply final velocities to positions
    //    This is where entities actually move
    movementSystem(registry);

    // 7. Check what's under each entity's feet
    //    Must be after movement so we check the final position
    groundDetectionSystem(registry, level);

    // 8. Fire weapons (hitscan and projectile spawning)
    combatSystem(registry, level);

    // 9. Destroy expired entities (projectiles, tracers)
    lifetimeSystem(registry);

    // 10. Detect trigger overlaps (damage zones, teleporters)
    //     Must be after movement so positions are final
    triggerSystem(registry);

    // 11. Detect damage, handle death and respawn
    //     Must be after all damage sources (combat + triggers)
    playerStateSystem(registry);

    // 12. Reset demo cubes to their starting positions
    demoResetSystem(registry);
}
```

### Why the order matters

Common bugs from wrong tick order:

| Bug | Cause |
|-----|-------|
| Player falls through floor | `groundDetectionSystem` before `movementSystem` |
| Player clips into walls | `movementSystem` before `collisionSystem` |
| Lift doesn't carry player | `moverSystem` after `collisionSystem` |
| Jump doesn't work | `playerMovementSystem` after `physicsSystem` (gravity cancels the jump) |
| Damage flash one frame late | `playerStateSystem` before `triggerSystem` |
| Projectiles spawn inside shooter | `combatSystem` before `collisionSystem` (no, this is fine — projectiles spawn offset) |

---

## Document: Collision Layers

The collision layer system from Chapter 9 uses bitmasks. Here's the current setup:

```cpp
// collision_layers.h
namespace CollisionLayers
{
    constexpr uint32_t Player     = 0x01;
    constexpr uint32_t World      = 0x02;
    constexpr uint32_t Enemy      = 0x04;
    constexpr uint32_t Projectile = 0x08;
    constexpr uint32_t Trigger    = 0x10;
    constexpr uint32_t All        = 0xFFFFFFFF;
}
```

### Layer assignments

| Entity | Layer | Mask | Collides with |
|--------|-------|------|---------------|
| Player | Player | All | Everything |
| Walls/floor (level) | World | All | Everything |
| Scene cubes | World | All | Everything |
| Platforms | World | All | Everything |
| Projectiles | Projectile | World \| Enemy | Walls, enemies (not player who fired) |
| Triggers | Trigger | Player | Only the player |

### Current issue

The player entity currently uses the default `AABBCollider` layer (`World`) and mask (`All`). This works but is semantically wrong — the player should be on the `Player` layer.

### Fix in `scene_setup.cpp`

```cpp
auto& playerCol = registry.emplace<AABBCollider>(player,
    glm::vec3(0.3f, 0.85f, 0.3f), false);
playerCol.layer = CollisionLayers::Player;
playerCol.mask = CollisionLayers::World | CollisionLayers::Enemy;
```

The player collides with the world (walls, floors, platforms) and enemies, but not with triggers (trigger detection is handled separately by `triggerSystem` which checks overlap directly).

---

## Organise: Scene Setup Sections

The `scene_setup.cpp` file has grown organically. Group it into clearly labelled sections:

```cpp
Level setupScene(entt::registry& registry, const ResourceManager& resources)
{
    // Resource lookups
    auto litShader   = resources.getShader("lit");
    auto gridGrey    = resources.getTexture("grid_grey");
    // ... etc

    // ═══════════════════════════════════════════════════════════
    // LEVEL GEOMETRY
    // ═══════════════════════════════════════════════════════════
    Level level = createShowcaseLevel();
    // ... sector entities

    // ═══════════════════════════════════════════════════════════
    // PLAYER
    // ═══════════════════════════════════════════════════════════
    auto player = registry.create();
    // ... all player components

    // ═══════════════════════════════════════════════════════════
    // LIGHTING
    // ═══════════════════════════════════════════════════════════
    // ... lights

    // ═══════════════════════════════════════════════════════════
    // PHYSICS DEMOS
    // ═══════════════════════════════════════════════════════════
    // ... demo cubes

    // ═══════════════════════════════════════════════════════════
    // INTERACTIVE OBJECTS (doors, lifts, teleporters)
    // ═══════════════════════════════════════════════════════════
    // ... movers and triggers

    // ═══════════════════════════════════════════════════════════
    // PLATFORMING ARENA
    // ═══════════════════════════════════════════════════════════
    // ... platforms from Chapter 14

    // ═══════════════════════════════════════════════════════════
    // HAZARDS
    // ═══════════════════════════════════════════════════════════
    // ... lava pool

    // ═══════════════════════════════════════════════════════════
    // COMBAT RESOURCES
    // ═══════════════════════════════════════════════════════════
    auto& combatRes = registry.ctx().emplace<CombatResources>();
    // ...

    return level;
}
```

This doesn't change any behaviour — it just makes the file easier to navigate. When TrenchBroom replaces this file in Chapter 16, each section maps to a type of entity you'll place in the editor.

---

## Audit: Component Completeness

Here's a checklist of all components and where they're used. Verify your codebase matches:

### Components defined in `components.h`

| Component | Used by system(s) | Attached to |
|-----------|-------------------|-------------|
| `Position` | All spatial systems | All visible/physical entities |
| `Rotation` | `renderSystem` | Entities that need rotation (tracers) |
| `Scale` | `renderSystem` | Entities with non-default scale |
| `Velocity` | `physicsSystem`, `movementSystem`, `collisionSystem` | Moving entities (player, cubes, projectiles) |
| `Vertex` | (data struct for mesh loading) | Not an ECS component |
| `AABBCollider` | `collisionSystem`, `groundDetectionSystem`, `combatSystem` | Physical entities |
| `Gravity` | `physicsSystem` | Entities that fall |
| `OnGround` | `physicsSystem`, `groundDetectionSystem`, `playerMovementSystem` | Entities that can land |
| `CharacterPhysics` | `physicsSystem` (friction), `playerMovementSystem` | Player, demo cubes with friction |
| `Weapon` | (data struct inside WeaponInventory) | Not directly attached |
| `WeaponInventory` | `combatSystem`, `weaponSwitchSystem` | Player |
| `Ammo` | (display only currently) | Player |
| `Projectile` | `combatSystem` (collision) | Spawned projectile entities |
| `CombatResources` | `combatSystem` (context) | Registry context |
| `Health` | `combatSystem`, `triggerSystem`, `playerStateSystem` | Player, destructible entities |
| `Mover` | `moverSystem` | Doors, lifts, moving platforms |
| `TriggerVolume` | `triggerSystem` | Trigger zones |
| `Lifetime` | `lifetimeSystem` | Projectiles, tracers |
| `PlayerInput` | `playerMovementSystem`, `combatSystem`, `weaponSwitchSystem` | Player |
| `PlayerState` | `playerStateSystem`, `debugHudSystem` | Player |
| `HudConfig` | `debugHudSystem` (context) | Registry context |
| `DirectionalLight` | `renderSystem` | Sun light |
| `PointLight` | `renderSystem` | Point lights |
| `MeshRenderer` | `renderSystem` | All visible entities |
| `Colour` | (unused — defined but never attached) | None |
| `DemoReset` | `demoResetSystem` | Demo cubes |
| `TagPlayer` | Multiple systems for player identification | Player |
| `TagDebugWireframe` | `renderSystem` (wireframe + green override) | Debug volumes, tracers |

### Items to note

- **`Colour`** is defined but unused. Keep it — it'll be useful when TrenchBroom entities need custom colours.
- **`Ammo`** is attached and displayed but never consumed by firing. This is a known gap — weapons fire regardless of ammo count. To fix, add an ammo check in `combatSystem` before firing and decrement the appropriate ammo type. This is left as an exercise.

---

## Final State: All Files

After completing chapters 12-15a, here's the complete list of source files:

```
src/
├── main.cpp
└── engine/
    ├── core/
    │   ├── fixed_timestep.h
    │   ├── input_manager.h / .cpp
    │   ├── resource_manager.h / .cpp
    │   └── window.h / .cpp
    ├── ecs/
    │   ├── components.h
    │   ├── scene_setup.h / .cpp
    │   ├── weapon_definitions.h
    │   └── systems/
    │       ├── collision_system.h / .cpp
    │       ├── combat_system.h / .cpp
    │       ├── debug_hud_system.h / .cpp        (Ch 13-15)
    │       ├── demo_reset_system.h / .cpp
    │       ├── lifetime_system.h / .cpp
    │       ├── movement_system.h / .cpp
    │       ├── mover_system.h / .cpp
    │       ├── physics_system.h / .cpp
    │       ├── player_movement_system.h / .cpp  (Ch 13)
    │       ├── player_state_system.h / .cpp     (Ch 15)
    │       ├── render_system.h / .cpp
    │       ├── trigger_system.h / .cpp
    │       └── weapon_switch_system.h
    ├── level/
    │   ├── level.h
    │   └── level_loader.h / .cpp
    ├── physics/
    │   ├── aabb.h
    │   ├── collision.h / .cpp
    │   ├── collision_layers.h
    │   ├── physics_config.h
    │   ├── raycast.h / .cpp
    │   └── spatial_hash.h / .cpp
    └── renderer/
        ├── camera.h / .cpp
        ├── mesh.h / .cpp
        ├── obj_loader.h / .cpp
        ├── shader.h / .cpp
        ├── stb_image_impl.cpp
        └── texture.h / .cpp

assets/
├── models/
│   └── cube.obj
├── shaders/
│   ├── basic.vert / .frag
│   ├── textured.vert / .frag
│   ├── lit.vert / .frag
│   └── hud.vert / .frag                        (Ch 13)
└── textures/
    ├── wall.png
    ├── grid_grey.png
    ├── grid_orange.png
    ├── grid_blue.png
    ├── grid_green.png
    └── grid_red.png
```

---

## What Changed — Summary

| File | Change |
|------|--------|
| `components.h` | Fixed `LighteningGun` → `LightningGun` typo, fixed `attacked` → `attached` comment |
| `weapon_definitions.h` | Fixed `LighteningGun` → `LightningGun`, fixed Nailgun to `FireMode::Projectile` |
| `combat_system.cpp` | Removed debug cout, fixed comment typos |
| `collision_system.cpp` | Fixed comment typos |
| `debug_hud_system.cpp` | Fixed `LighteningGun` reference |
| `scene_setup.cpp` | Set player collision layer to `Player`, organised sections |

---

## Checkpoint

At this point, the engine has:

- A **physical player body** with gravity, jumping, and collision
- **Moon-like platforming** with variable jump height and coyote time
- **Weapons** that fire hitscan and projectile attacks
- **Interactive objects**: doors, lifts, teleporters, lava damage zones
- **A gameplay loop**: take damage, die, respawn
- **Visual feedback**: crosshair, HUD, damage flash, hitscan tracers, projectile cubes
- **Debug info**: FPS, health, ammo, weapon name

All built on a clean ECS architecture with 12 systems running in a documented tick order.

The one remaining limitation is that the level is a **hardcoded C++ function** (`createShowcaseLevel` in `scene_setup.cpp`). Every room, platform, light, and trigger is placed by writing code and recompiling. In Chapter 16, we'll replace this with **TrenchBroom** — a proper 3D level editor where you can build rooms visually, place entities, and export to a `.map` file that the engine loads at runtime.

---

## What's Next

In **Chapter 16**, we begin TrenchBroom integration. We'll write a parser for the `.map` file format, convert brush geometry into renderable meshes, and load our first TrenchBroom-authored level.
