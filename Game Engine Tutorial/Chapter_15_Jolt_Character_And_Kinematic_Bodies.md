# Chapter 15: Jolt Character Controller & Kinematic Bodies

## What You'll Learn
- Using Jolt's `CharacterVirtual` for player movement
- Wiring WASD/jump input into the character controller
- Automatic stair stepping and slope handling
- Converting movers (lifts, doors) to kinematic Jolt bodies
- Trigger volumes as Jolt sensor bodies
- The player riding lifts without any custom rider code

---

## The Goal

Chapter 14 gave us a Jolt-powered physics world with solid floors and falling cubes. Now we add the player. Rather than building a character controller from scratch (as we did with `playerMovementSystem` + `physicsSystem` + `groundDetectionSystem`), we'll use Jolt's built-in `CharacterVirtual` — a specialised class designed for FPS player movement.

We'll also convert movers to kinematic bodies so lifts and doors interact with the player physically. The "lift doesn't carry the player" problem from Chapter 13 is solved automatically — kinematic bodies push dynamic/virtual characters as they move.

---

## Step 1: CharacterVirtual vs Character

Jolt offers two character controller types:

| | `Character` | `CharacterVirtual` |
|---|---|---|
| **Physics** | Full rigid body in the simulation | No rigid body — uses collision queries |
| **Control** | Less direct — apply forces | Direct — set velocity each frame |
| **Interactions** | Other bodies see and push it | Invisible to other bodies (optional proxy) |
| **Best for** | AI characters | Player characters |

We'll use `CharacterVirtual` because:
- We want direct velocity control (Quake-style acceleration)
- We need to update it inside our fixed timestep loop with precise timing
- The player's movement rules are game-specific, not physics-derived

---

## Step 2: Player Character System

This system replaces both `playerMovementSystem` and the old `physicsSystem` (gravity/friction) for the player entity. It creates a `CharacterVirtual`, applies input-driven velocity, and steps the character each tick.

### New Component: `JoltCharacter`

Add to `components.h`:

```cpp
#include <Jolt/Physics/Character/CharacterVirtual.h>

// Links an ECS entity to a Jolt CharacterVirtual controller
struct JoltCharacter
{
    JPH::Ref<JPH::CharacterVirtual> character;
};
```

### New file: `src/engine/ecs/systems/player_character_system.h`

```cpp
#pragma once

#include <entt/entt.hpp>

void initPlayerCharacter(entt::registry& registry);
void playerCharacterSystem(entt::registry& registry, float dt);
```

### New file: `src/engine/ecs/systems/player_character_system.cpp`

```cpp
#include "engine/ecs/systems/player_character_system.h"
#include "engine/ecs/components.h"
#include "engine/physics/jolt_world.h"

#include <Jolt/Physics/Character/CharacterVirtual.h>

void initPlayerCharacter(entt::registry& registry)
{
    auto& jolt = registry.ctx().get<JoltWorld>();

    auto view = registry.view<Position, AABBCollider, TagPlayer>();
    for (auto [entity, pos, col] : view.each())
    {
        // Create a capsule shape for the player
        // Capsule height = total height minus the two hemisphere caps
        float radius = col.halfExtents.x;  // 0.3
        float halfHeight = col.halfExtents.y - radius;  // 0.85 - 0.3 = 0.55
        if (halfHeight < 0.01f) halfHeight = 0.01f;

        JPH::Ref<JPH::Shape> capsuleShape = new JPH::CapsuleShape(halfHeight, radius);

        // Configure the character
        JPH::Ref<JPH::CharacterVirtualSettings> settings = new JPH::CharacterVirtualSettings();
        settings->mShape = capsuleShape;
        settings->mMaxSlopeAngle = JPH::DegreesToRadians(50.0f);
        settings->mMaxStrength = 100.0f;
        settings->mMass = 70.0f;
        settings->mPredictiveContactDistance = 0.1f;

        // Create the character at the entity's current position
        JPH::Ref<JPH::CharacterVirtual> character = new JPH::CharacterVirtual(
            settings,
            JPH::RVec3(pos.value.x, pos.value.y, pos.value.z),
            JPH::Quat::sIdentity(),
            0,  // user data
            jolt.physicsSystem.get()
        );

        registry.emplace<JoltCharacter>(entity, character);
    }
}

void playerCharacterSystem(entt::registry& registry, float dt)
{
    auto& jolt = registry.ctx().get<JoltWorld>();

    auto view = registry.view<Position, JoltCharacter, PlayerInput, CharacterPhysics, OnGround>();
    for (auto [entity, pos, joltChar, input, physics, ground] : view.each())
    {
        auto& character = joltChar.character;

        // ─── Read ground state from Jolt ────────────────────────
        bool onGround = character->GetGroundState() == JPH::CharacterVirtual::EGroundState::OnGround;
        ground.value = onGround;

        // ─── Build velocity from input ──────────────────────────
        JPH::Vec3 currentVel = character->GetLinearVelocity();
        JPH::Vec3 desiredVel(0.0f, 0.0f, 0.0f);

        if (onGround)
        {
            // Ground movement — Quake-style acceleration
            JPH::Vec3 wishDir(input.wishDir.x, 0.0f, input.wishDir.z);
            float wishSpeed = physics.maxGroundSpeed;

            if (wishDir.LengthSq() > 0.0f)
            {
                wishDir = wishDir.Normalized();
                float currentSpeed = currentVel.Dot(wishDir);
                float addSpeed = wishSpeed - currentSpeed;
                if (addSpeed > 0.0f)
                {
                    float accelSpeed = physics.groundAcceleration * wishSpeed * dt;
                    if (accelSpeed > addSpeed) accelSpeed = addSpeed;
                    desiredVel = JPH::Vec3(currentVel.GetX(), 0.0f, currentVel.GetZ())
                                 + wishDir * accelSpeed;
                }
                else
                {
                    desiredVel = JPH::Vec3(currentVel.GetX(), 0.0f, currentVel.GetZ());
                }
            }
            else
            {
                // No input — apply ground friction
                JPH::Vec3 horizontalVel(currentVel.GetX(), 0.0f, currentVel.GetZ());
                float speed = horizontalVel.Length();
                if (speed > 0.1f)
                {
                    float drop = speed * physics.groundFriction * dt;
                    float newSpeed = std::max(speed - drop, 0.0f);
                    desiredVel = horizontalVel * (newSpeed / speed);
                }
            }

            // Jump
            if (input.jump)
            {
                desiredVel += JPH::Vec3(0.0f, physics.jumpForce, 0.0f);
            }
            else
            {
                // Keep ground velocity vertical component
                desiredVel += JPH::Vec3(0.0f, currentVel.GetY(), 0.0f);
            }
        }
        else
        {
            // Air movement — limited air control
            JPH::Vec3 wishDir(input.wishDir.x, 0.0f, input.wishDir.z);

            if (wishDir.LengthSq() > 0.0f)
            {
                wishDir = wishDir.Normalized();
                float currentSpeed = JPH::Vec3(currentVel.GetX(), 0.0f, currentVel.GetZ()).Dot(wishDir);
                float addSpeed = physics.maxAirSpeed - currentSpeed;
                if (addSpeed > 0.0f)
                {
                    float accelSpeed = physics.airAcceleration * physics.maxAirSpeed * dt;
                    if (accelSpeed > addSpeed) accelSpeed = addSpeed;
                    desiredVel = JPH::Vec3(currentVel.GetX(), currentVel.GetY(), currentVel.GetZ())
                                 + wishDir * accelSpeed;
                }
                else
                {
                    desiredVel = JPH::Vec3(currentVel.GetX(), currentVel.GetY(), currentVel.GetZ());
                }
            }
            else
            {
                desiredVel = JPH::Vec3(currentVel.GetX(), currentVel.GetY(), currentVel.GetZ());
            }

            // Apply gravity while in the air
            desiredVel += JPH::Vec3(0.0f, -20.0f * dt, 0.0f);
        }

        character->SetLinearVelocity(desiredVel);

        // ─── Step the character ─────────────────────────────────
        // ExtendedUpdate handles collision, stair stepping, and floor sticking
        JPH::CharacterVirtual::ExtendedUpdateSettings updateSettings;
        updateSettings.mWalkStairsStepUp = JPH::Vec3(0.0f, physics.stepHeight, 0.0f);
        updateSettings.mStickToFloorStepDown = JPH::Vec3(0.0f, -physics.stepHeight, 0.0f);

        character->ExtendedUpdate(
            dt,
            -character->GetUp() * jolt.physicsSystem->GetGravity().Length(),
            updateSettings,
            jolt.physicsSystem->GetDefaultBroadPhaseLayerFilter(Layers::MOVING),
            jolt.physicsSystem->GetDefaultLayerFilter(Layers::MOVING),
            {},  // body filter
            {},  // shape filter
            *jolt.tempAllocator
        );

        // ─── Write position back to ECS ─────────────────────────
        JPH::RVec3 charPos = character->GetPosition();
        pos.value = glm::vec3(charPos.GetX(), charPos.GetY(), charPos.GetZ());
    }
}
```

### Key Differences from the Old System

| Old Approach | Jolt Approach |
|---|---|
| Manual gravity in `physicsSystem` | Gravity applied to velocity before `ExtendedUpdate` |
| Manual friction with `horizontalVel *= factor` | Same Quake-style friction, but Jolt handles resting contact |
| `groundDetectionSystem` raycasting | `character->GetGroundState()` — built-in ground detection |
| Stair-stepping hack in `collisionSystem` | `ExtendedUpdate` with `mWalkStairsStepUp` — automatic |
| Player position via `movementSystem` | `character->GetPosition()` — Jolt resolves collisions internally |

---

## Step 3: Kinematic Movers

Lifts and doors should be kinematic Jolt bodies. When a kinematic body moves, it pushes dynamic bodies and virtual characters out of the way — solving the "lift doesn't carry the player" problem.

### Update `scene_setup.cpp`

For entities with a `Mover` component, create a kinematic body:

```cpp
void createKinematicBody(entt::registry& registry, entt::entity entity)
{
    auto& jolt = registry.ctx().get<JoltWorld>();
    auto& bodyInterface = jolt.getBodyInterface();
    auto& pos = registry.get<Position>(entity);
    auto& col = registry.get<AABBCollider>(entity);

    JPH::BoxShapeSettings shapeSettings(
        JPH::Vec3(col.halfExtents.x, col.halfExtents.y, col.halfExtents.z)
    );
    shapeSettings.SetEmbedded();

    auto shapeResult = shapeSettings.Create();

    JPH::BodyCreationSettings bodySettings(
        shapeResult.Get(),
        JPH::RVec3(pos.value.x, pos.value.y, pos.value.z),
        JPH::Quat::sIdentity(),
        JPH::EMotionType::Kinematic,
        Layers::MOVING
    );

    JPH::BodyID bodyId = bodyInterface.CreateAndAddBody(
        bodySettings, JPH::EActivation::Activate
    );

    registry.emplace<JoltBody>(entity, bodyId);
}
```

### Update `moverSystem`

After the mover system updates `pos.value` via `glm::mix`, we need to push that new position into the Jolt kinematic body.

Add a new system that runs right after `moverSystem`:

### New file: `src/engine/ecs/systems/mover_sync_system.h`

```cpp
#pragma once

#include <entt/entt.hpp>

void moverSyncSystem(entt::registry& registry);
```

### New file: `src/engine/ecs/systems/mover_sync_system.cpp`

```cpp
#include "engine/ecs/systems/mover_sync_system.h"
#include "engine/ecs/components.h"
#include "engine/physics/jolt_world.h"

void moverSyncSystem(entt::registry& registry)
{
    auto& jolt = registry.ctx().get<JoltWorld>();
    auto& bodyInterface = jolt.getBodyInterface();

    auto view = registry.view<Position, Mover, JoltBody>();
    for (auto [entity, pos, mover, joltBody] : view.each())
    {
        // Move the kinematic body to match the mover's current position
        bodyInterface.MoveKinematic(
            joltBody.id,
            JPH::RVec3(pos.value.x, pos.value.y, pos.value.z),
            JPH::Quat::sIdentity(),
            1.0f / 60.0f  // the time step — Jolt uses this to compute velocity
        );
    }
}
```

> **`MoveKinematic` vs `SetPosition`** — `MoveKinematic` tells Jolt where the body should be at the end of the next physics step. Jolt calculates the velocity needed to reach that position, which means it pushes anything in the way. `SetPosition` would teleport the body without pushing anything.

---

## Step 4: Trigger Volumes as Sensors

Our trigger volumes (lava, teleporter, lift activator) should be Jolt sensor bodies — they detect overlap but don't block movement.

### Update `scene_setup.cpp`

For entities with `AABBCollider.isTrigger == true`:

```cpp
void createSensorBody(entt::registry& registry, entt::entity entity)
{
    auto& jolt = registry.ctx().get<JoltWorld>();
    auto& bodyInterface = jolt.getBodyInterface();
    auto& pos = registry.get<Position>(entity);
    auto& col = registry.get<AABBCollider>(entity);

    JPH::BoxShapeSettings shapeSettings(
        JPH::Vec3(col.halfExtents.x, col.halfExtents.y, col.halfExtents.z)
    );
    shapeSettings.SetEmbedded();

    auto shapeResult = shapeSettings.Create();

    JPH::BodyCreationSettings bodySettings(
        shapeResult.Get(),
        JPH::RVec3(pos.value.x, pos.value.y, pos.value.z),
        JPH::Quat::sIdentity(),
        JPH::EMotionType::Static,
        Layers::SENSOR
    );
    bodySettings.mIsSensor = true;

    JPH::BodyID bodyId = bodyInterface.CreateAndAddBody(
        bodySettings, JPH::EActivation::DontActivate
    );

    registry.emplace<JoltBody>(entity, bodyId);
}
```

> **Note:** The `triggerSystem` still uses its own AABB overlap check against ECS positions. You can keep this working as-is since the positions are still in the ECS. Later, you could replace it with Jolt's contact listener for sensor overlap events, but that's an optimisation, not a requirement.

---

## Step 5: Wire It All Up in main.cpp

### Add new includes

```cpp
#include "engine/ecs/systems/player_character_system.h"
#include "engine/ecs/systems/mover_sync_system.h"
```

### Remove old player movement include

```cpp
// REMOVE:
#include "engine/ecs/systems/player_movement_system.h"
```

### Initialise the player character after scene setup

After `OptimizeBroadPhase()`:

```cpp
// Create Jolt bodies for movers (lifts, doors)
auto moverView = registry.view<Position, AABBCollider, Mover>();
for (auto entity : moverView)
{
    createKinematicBody(registry, entity);
}

// Create sensor bodies for triggers
auto triggerView = registry.view<Position, AABBCollider, TriggerVolume>();
for (auto entity : triggerView)
{
    auto& col = registry.get<AABBCollider>(entity);
    if (col.isTrigger)
    {
        createSensorBody(registry, entity);
    }
}

// Initialise the player's CharacterVirtual
initPlayerCharacter(registry);

// Re-optimise broad phase after adding more bodies
joltWorld.physicsSystem->OptimizeBroadPhase();
```

### Update the tick loop

```cpp
while (fixedTimestep.step())
{
    float dt = 1.0f / 60.0f;

    weaponSwitchSystem(registry);
    playerCharacterSystem(registry, dt);    // NEW: replaces playerMovementSystem
    moverSystem(registry);                   // animate doors/lifts
    moverSyncSystem(registry);               // NEW: push mover positions to Jolt
    joltWorld.step(dt);                      // step physics
    joltSyncSystem(registry);                // sync dynamic body transforms to ECS
    combatSystem(registry, level);
    lifetimeSystem(registry);
    triggerSystem(registry);
    demoResetSystem(registry);
}
```

### Tick Order Explained

```
1. weaponSwitchSystem      — handle weapon swap input
2. playerCharacterSystem   — apply input → Jolt CharacterVirtual
3. moverSystem             — animate door/lift positions
4. moverSyncSystem         — push mover positions to Jolt kinematic bodies
5. joltWorld.step()        — simulate physics, resolve collisions
6. joltSyncSystem          — read Jolt transforms → ECS Position/Velocity
7. combatSystem            — hitscan/projectile weapons
8. lifetimeSystem          — auto-destroy timed entities
9. triggerSystem            — detect trigger overlaps
10. demoResetSystem         — reset physics demos
```

The critical ordering: `playerCharacterSystem` runs first to set the character's desired velocity. `moverSyncSystem` runs before the physics step so kinematic bodies have their target positions. The physics step resolves everything. Then `joltSyncSystem` reads the results back to the ECS.

---

## Step 6: Update CMakeLists.txt

### Remove old sources

```cmake
# REMOVE (if not already removed in Ch14):
src/engine/ecs/systems/player_movement_system.cpp
```

### Add new sources

```cmake
src/engine/ecs/systems/player_character_system.cpp
src/engine/ecs/systems/mover_sync_system.cpp
```

---

## What Changed — Summary

| File | Change |
|------|--------|
| `components.h` | Added `JoltCharacter` component |
| `player_character_system.h/cpp` | **New** — Quake-style movement via Jolt CharacterVirtual |
| `mover_sync_system.h/cpp` | **New** — Pushes mover positions into Jolt kinematic bodies |
| `scene_setup.cpp` | Added `createKinematicBody()`, `createSensorBody()`, character init call |
| `main.cpp` | Updated tick loop, removed old player movement system |
| `CMakeLists.txt` | Swapped source files |

### Files removed from build

| File | Was |
|------|-----|
| `player_movement_system.cpp/h` | Custom Quake-style acceleration (now in `playerCharacterSystem`) |

---

## What You Should See

After building and running:

1. **WASD moves the player** — same Quake-style acceleration and friction as before
2. **Spacebar jumps** — gravity pulls you back down, no jitter on landing
3. **The lift carries you upward** — stand on it, trigger it, ride it up. No custom rider code needed — the kinematic body pushes the CharacterVirtual
4. **Doors push you out of the way** — walk into a closing door and it moves you
5. **Stair stepping works** — walk onto the lift's thin platform without being blocked. `ExtendedUpdate` handles this automatically
6. **Cubes land cleanly** — no jitter, no micro-bouncing
7. **Lava still damages you** — trigger system is unchanged
8. **Weapons still work** — combat system is unchanged

### Troubleshooting

**Player falls through the floor:**
- Check that `initPlayerCharacter` is called after level bodies are created
- Verify the capsule shape dimensions match the player's collider half-extents
- Ensure gravity direction is `(0, -20, 0)` not `(0, 20, 0)`

**Player slides on slopes they should stand on:**
- Increase `mMaxSlopeAngle` in the character settings (default 50 degrees)

**Lift doesn't push the player:**
- Verify the lift entity has a `JoltBody` component (kinematic)
- Check that `moverSyncSystem` runs before `joltWorld.step()`
- Use `MoveKinematic` not `SetPosition` — only `MoveKinematic` generates the velocity needed to push

**Player movement feels different:**
- The Quake-style acceleration is reimplemented in `playerCharacterSystem`. Tune `CharacterPhysics` values (groundAcceleration, maxGroundSpeed, etc.) to match your desired feel.
- `ExtendedUpdate` has its own floor-sticking behaviour that interacts with your velocity — if the player "sticks" going down slopes, reduce `mStickToFloorStepDown`

**Triggers don't fire:**
- The `triggerSystem` uses ECS-level AABB overlap, which still works since positions are synced. If triggers seem unreliable, check that the player's ECS `Position` is being updated each frame.

---

## New C++ Concept: CharacterVirtual

`CharacterVirtual` is Jolt's solution for player characters in games. Unlike a rigid body, it doesn't participate in the physics simulation directly — instead, it performs its own collision queries each frame.

The key method is `ExtendedUpdate`:

```cpp
character->ExtendedUpdate(
    deltaTime,        // how far forward in time to simulate
    gravity,          // gravity vector (for floor sticking)
    updateSettings,   // stair step and floor stick configuration
    broadPhaseFilter, // which broad-phase layers to collide with
    layerFilter,      // which object layers to collide with
    bodyFilter,       // per-body filtering (empty = all)
    shapeFilter,      // per-shape filtering (empty = all)
    allocator         // Jolt temp allocator
);
```

This single call:
1. Applies the character's velocity to move it forward
2. Detects collisions and slides along surfaces
3. Steps up stairs if the obstacle is short enough
4. Sticks to the floor when walking down slopes
5. Updates ground state (on ground, in air, on steep slope)

All the behaviour we hand-coded across four systems — in one function call.

---

## New C++ Concept: Kinematic Bodies

A kinematic body is controlled by code, not physics. It has infinite mass — nothing can push it. But it pushes everything else.

```cpp
// Create a kinematic body
bodySettings.mMotionType = EMotionType::Kinematic;

// Each frame, tell it where to be
bodyInterface.MoveKinematic(bodyId, targetPosition, targetRotation, deltaTime);
```

The `MoveKinematic` call is critical. It tells Jolt: "this body needs to be at `targetPosition` after `deltaTime` seconds." Jolt calculates the velocity needed, and during the physics step, the body moves at that velocity — pushing anything in its path.

This is exactly what our movers (lifts, doors) need. The `moverSystem` calculates the position via `glm::mix`, then `moverSyncSystem` tells Jolt's kinematic body to move there. During the physics step, the kinematic body sweeps to its target position and pushes the player's CharacterVirtual.

---

## Architecture Review

Let's compare the old and new physics architecture:

### Before (Custom Physics)

```
playerMovementSystem  → sets Velocity from input
physicsSystem         → applies gravity, friction to Velocity
moverSystem           → animates mover positions
collisionSystem       → sweeps AABBs, adjusts Velocity
movementSystem        → applies Velocity to Position
groundDetectionSystem → raycasts downward, sets OnGround
```

Six systems, all interdependent, full of edge cases.

### After (Jolt)

```
playerCharacterSystem → sets CharacterVirtual velocity, calls ExtendedUpdate
moverSystem           → animates mover positions (unchanged)
moverSyncSystem       → pushes mover positions to kinematic bodies
joltWorld.step()      → simulates everything
joltSyncSystem        → reads transforms back to ECS
```

Five calls, but the heavy lifting is inside Jolt. Ground detection, stair stepping, collision response, resting contact — all handled by Jolt's solver.

---

## What's Next

The physics are now solid and reliable. In **Chapter 16**, we'll start building test objects and gameplay scenarios — floating platforms, moving obstacles, physics puzzles — to exercise the new physics system and verify everything works end-to-end before integrating TrenchBroom.
