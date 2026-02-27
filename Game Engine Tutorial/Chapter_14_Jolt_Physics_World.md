# Chapter 14: Jolt Physics — World & Static Geometry

## What You'll Learn
- Why we're replacing custom physics with Jolt
- Installing and linking Jolt Physics via CMake FetchContent
- The Jolt boilerplate: layers, filters, allocators, job systems
- Creating a physics world and stepping it each tick
- Converting level surfaces into static rigid bodies
- Converting entity colliders into dynamic rigid bodies
- Syncing Jolt body transforms back into ECS `Position` components
- Removing the old custom physics systems

---

## Why Jolt?

Our custom physics — `physicsSystem`, `collisionSystem`, `movementSystem`, `groundDetectionSystem` — works for simple cases but breaks down when things interact. The lift can't carry the player. Objects jitter when resting on surfaces. Stair-stepping is a fragile hack. Every fix reveals the next edge case.

Real physics engines solve all of this with constraint solvers, persistent contact manifolds, and proper resting contact. Jolt Physics is a modern C++17 library used in Horizon Forbidden West and Death Stranding 2. It's multi-threaded, clean to integrate, and its API style fits naturally with our EnTT codebase.

The swap is contained: we replace four systems and their support code. Everything else — rendering, ECS, input, triggers, movers, combat, the debug HUD — stays untouched.

---

## Step 1: Add Jolt to the Project

We'll use CMake FetchContent to download and build Jolt automatically.

### Update `CMakeLists.txt`

Add these lines after the existing `stb` library block and before the `add_executable` line:

```cmake
# ──────────────────────────────────────────────
# Jolt Physics — fetched from GitHub
# ──────────────────────────────────────────────
include(FetchContent)

# Jolt configuration — set BEFORE FetchContent_MakeAvailable
set(PHYSICS_REPO_ROOT ${CMAKE_CURRENT_SOURCE_DIR})
set(USE_STATIC_MSVC_RUNTIME_LIBRARY OFF CACHE BOOL "" FORCE)

FetchContent_Declare(
    JoltPhysics
    GIT_REPOSITORY https://github.com/jrouwe/JoltPhysics.git
    GIT_TAG        v5.2.0
    SOURCE_SUBDIR  Build
)
FetchContent_MakeAvailable(JoltPhysics)
```

Then add `Jolt` to the `target_link_libraries`:

```cmake
target_link_libraries(QEngine PRIVATE
    glfw
    glad
    glm
    entt
    stb
    Jolt                # NEW
)
```

The first build will take a minute or two as CMake downloads and compiles Jolt. Subsequent builds are fast — CMake caches the result.

> **`SOURCE_SUBDIR Build`** — Jolt's CMakeLists.txt lives in a `Build/` subdirectory of the repo, not the root. This tells FetchContent where to find it.

---

## Step 2: The Jolt Boilerplate

Jolt requires several pieces of setup before you can create a physics world. This is the most code-heavy step, but it's all copy-paste boilerplate that you write once and rarely touch again.

### New file: `src/engine/physics/jolt_setup.h`

```cpp
#pragma once

// Jolt requires this macro before including any headers
#include <Jolt/Jolt.h>

// Jolt headers
#include <Jolt/RegisterTypes.h>
#include <Jolt/Core/Factory.h>
#include <Jolt/Core/TempAllocator.h>
#include <Jolt/Core/JobSystemThreadPool.h>
#include <Jolt/Physics/PhysicsSettings.h>
#include <Jolt/Physics/PhysicsSystem.h>
#include <Jolt/Physics/Collision/Shape/BoxShape.h>
#include <Jolt/Physics/Collision/Shape/CapsuleShape.h>
#include <Jolt/Physics/Body/BodyCreationSettings.h>
#include <Jolt/Physics/Body/BodyActivationListener.h>

#include <thread>
#include <cstdarg>
#include <iostream>

// All Jolt symbols are inside the JPH namespace
using namespace JPH;
using namespace JPH::literals;

// ─── Object Layers ──────────────────────────────────────────
// These map roughly to our existing CollisionLayers bitmask,
// but Jolt uses a different system: ObjectLayers for bodies,
// BroadPhaseLayers for the broad-phase acceleration structure.

namespace Layers
{
    static constexpr JPH::ObjectLayer NON_MOVING = 0;  // floors, walls
    static constexpr JPH::ObjectLayer MOVING     = 1;  // player, physics objects
    static constexpr JPH::ObjectLayer SENSOR     = 2;  // triggers (no collision response)
    static constexpr JPH::ObjectLayer NUM_LAYERS = 3;
};

namespace BroadPhaseLayers
{
    static constexpr JPH::BroadPhaseLayer NON_MOVING(0);
    static constexpr JPH::BroadPhaseLayer MOVING(1);
    static constexpr uint NUM_LAYERS(2);
};

// ─── Layer Mapping ──────────────────────────────────────────
// Maps each ObjectLayer to a BroadPhaseLayer.
// Non-moving and sensors share the static broad-phase layer.

class BPLayerInterfaceImpl final : public JPH::BroadPhaseLayerInterface
{
public:
    BPLayerInterfaceImpl()
    {
        mObjectToBroadPhase[Layers::NON_MOVING] = BroadPhaseLayers::NON_MOVING;
        mObjectToBroadPhase[Layers::MOVING]     = BroadPhaseLayers::MOVING;
        mObjectToBroadPhase[Layers::SENSOR]     = BroadPhaseLayers::NON_MOVING;
    }

    virtual uint GetNumBroadPhaseLayers() const override
    {
        return BroadPhaseLayers::NUM_LAYERS;
    }

    virtual JPH::BroadPhaseLayer GetBroadPhaseLayer(JPH::ObjectLayer inLayer) const override
    {
        return mObjectToBroadPhase[inLayer];
    }

private:
    JPH::BroadPhaseLayer mObjectToBroadPhase[Layers::NUM_LAYERS];
};

// ─── Collision Filters ──────────────────────────────────────
// Determines which object layers can collide with each other.
// Static objects don't collide with each other (no point).

class ObjectLayerPairFilterImpl : public JPH::ObjectLayerPairFilter
{
public:
    virtual bool ShouldCollide(JPH::ObjectLayer inLayer1,
                               JPH::ObjectLayer inLayer2) const override
    {
        switch (inLayer1)
        {
        case Layers::NON_MOVING:
            return inLayer2 == Layers::MOVING;
        case Layers::MOVING:
            return inLayer2 != Layers::SENSOR;
        case Layers::SENSOR:
            return inLayer2 == Layers::MOVING;
        default:
            return false;
        }
    }
};

class ObjectVsBroadPhaseLayerFilterImpl : public JPH::ObjectVsBroadPhaseLayerFilter
{
public:
    virtual bool ShouldCollide(JPH::ObjectLayer inLayer1,
                               JPH::BroadPhaseLayer inLayer2) const override
    {
        switch (inLayer1)
        {
        case Layers::NON_MOVING:
            return inLayer2 == BroadPhaseLayers::MOVING;
        case Layers::MOVING:
            return true;
        case Layers::SENSOR:
            return inLayer2 == BroadPhaseLayers::MOVING;
        default:
            return false;
        }
    }
};

// ─── Jolt Trace Callback (debug logging) ────────────────────

static void JoltTraceImpl(const char* inFMT, ...)
{
    va_list list;
    va_start(list, inFMT);
    char buffer[1024];
    vsnprintf(buffer, sizeof(buffer), inFMT, list);
    va_end(list);
    std::cout << "[Jolt] " << buffer << std::endl;
}

#ifdef JPH_ENABLE_ASSERTS
static bool JoltAssertFailedImpl(const char* inExpression, const char* inMessage,
                                  const char* inFile, uint inLine)
{
    std::cout << inFile << ":" << inLine << ": (" << inExpression << ") "
              << (inMessage != nullptr ? inMessage : "") << std::endl;
    return true;  // break into debugger
}
#endif
```

### Understanding the Layers

Jolt uses a two-tier filtering system:

1. **Object Layers** — every body has one. Our three layers are `NON_MOVING` (level geometry), `MOVING` (player, physics cubes), and `SENSOR` (trigger volumes).

2. **Broad-Phase Layers** — a coarser grouping used by Jolt's internal acceleration structure. We map `NON_MOVING` and `SENSOR` to the same broad-phase layer since they're both static. `MOVING` gets its own.

3. **Filters** — `ObjectLayerPairFilter` decides which object layer pairs can collide. `ObjectVsBroadPhaseLayerFilter` decides which object layers test against which broad-phase layers. The key rule: static doesn't collide with static.

This is more setup than our old bitmask system, but Jolt uses it for performance — the broad-phase layer structure enables very fast spatial queries.

---

## Step 3: The Physics World Wrapper

Rather than scattering Jolt setup through `main.cpp`, we'll wrap it in a class that the ECS can use.

### New file: `src/engine/physics/jolt_world.h`

```cpp
#pragma once

#include "engine/physics/jolt_setup.h"
#include <memory>

// Wraps Jolt's PhysicsSystem and its required infrastructure.
// Stored in registry context so systems can access it.
struct JoltWorld
{
    // Jolt infrastructure — must outlive the PhysicsSystem
    std::unique_ptr<JPH::TempAllocatorImpl> tempAllocator;
    std::unique_ptr<JPH::JobSystemThreadPool> jobSystem;

    // Layer interfaces
    BPLayerInterfaceImpl broadPhaseLayerInterface;
    ObjectVsBroadPhaseLayerFilterImpl objectVsBroadPhaseFilter;
    ObjectLayerPairFilterImpl objectLayerPairFilter;

    // The physics world itself
    std::unique_ptr<JPH::PhysicsSystem> physicsSystem;

    void init()
    {
        // Register Jolt allocator and install callbacks
        JPH::RegisterDefaultAllocator();
        JPH::Trace = JoltTraceImpl;
        JPH_IF_ENABLE_ASSERTS(JPH::AssertFailed = JoltAssertFailedImpl;)

        // Create the factory (needed for serialization/deserialization)
        JPH::Factory::sInstance = new JPH::Factory();
        JPH::RegisterTypes();

        // Pre-allocate 10 MB for physics temp data
        tempAllocator = std::make_unique<JPH::TempAllocatorImpl>(10 * 1024 * 1024);

        // Create a thread pool — use all available cores minus one
        jobSystem = std::make_unique<JPH::JobSystemThreadPool>(
            JPH::cMaxPhysicsJobs, JPH::cMaxPhysicsBarriers,
            (int)std::thread::hardware_concurrency() - 1
        );

        // Create the physics system
        const uint maxBodies = 1024;
        const uint numBodyMutexes = 0;    // auto
        const uint maxBodyPairs = 1024;
        const uint maxContactConstraints = 1024;

        physicsSystem = std::make_unique<JPH::PhysicsSystem>();
        physicsSystem->Init(
            maxBodies, numBodyMutexes, maxBodyPairs, maxContactConstraints,
            broadPhaseLayerInterface, objectVsBroadPhaseFilter,
            objectLayerPairFilter
        );

        // Set gravity (Quake-style: 20 units/s^2 downward)
        physicsSystem->SetGravity(JPH::Vec3(0.0f, -20.0f, 0.0f));
    }

    void step(float deltaTime)
    {
        // Step the physics simulation
        // 1 collision step per update is fine for our fixed timestep
        physicsSystem->Update(deltaTime, 1, tempAllocator.get(), jobSystem.get());
    }

    JPH::BodyInterface& getBodyInterface()
    {
        return physicsSystem->GetBodyInterface();
    }

    void shutdown()
    {
        physicsSystem.reset();
        jobSystem.reset();
        tempAllocator.reset();

        JPH::UnregisterTypes();
        delete JPH::Factory::sInstance;
        JPH::Factory::sInstance = nullptr;
    }
};
```

### New file: `src/engine/physics/jolt_world.cpp`

```cpp
#include "engine/physics/jolt_world.h"
```

This is intentionally minimal — the implementation is all in the header for now since everything is inline. The `.cpp` file exists so the linker has a translation unit for Jolt symbols.

---

## Step 4: Add a JoltBody Component

We need to link ECS entities to their Jolt physics bodies. A simple component stores the Jolt `BodyID`.

### Add to `components.h`

```cpp
// Links an ECS entity to a Jolt Physics body
struct JoltBody
{
    JPH::BodyID id;
};
```

You'll need to add this include at the top of `components.h`:

```cpp
#include <Jolt/Jolt.h>
#include <Jolt/Physics/Body/BodyID.h>
```

---

## Step 5: Create Static Bodies from Level Surfaces

Each surface in the level becomes a static box body in Jolt. This replaces the per-frame AABB construction that `collisionSystem` was doing.

### Update `scene_setup.cpp`

Add the include:

```cpp
#include "engine/physics/jolt_world.h"
```

After `setupScene` creates the level geometry, add a function to create the static bodies:

```cpp
void createLevelBodies(entt::registry& registry, const Level& level)
{
    auto& jolt = registry.ctx().get<JoltWorld>();
    auto& bodyInterface = jolt.getBodyInterface();

    for (const auto& sector : level.sectors)
    {
        for (const auto& surface : sector.surfaces)
        {
            // Compute AABB from the surface vertices
            glm::vec3 surfMin = glm::min(
                glm::min(surface.vertices[0], surface.vertices[1]),
                glm::min(surface.vertices[2], surface.vertices[3])
            );
            glm::vec3 surfMax = glm::max(
                glm::max(surface.vertices[0], surface.vertices[1]),
                glm::max(surface.vertices[2], surface.vertices[3])
            );

            // Fatten thin dimensions (same as old collision system)
            for (int i = 0; i < 3; i++)
            {
                if (surfMax[i] - surfMin[i] < 0.01f)
                {
                    surfMin[i] -= 0.05f;
                    surfMax[i] += 0.05f;
                }
            }

            // Jolt box shape takes half-extents
            glm::vec3 halfExtents = (surfMax - surfMin) * 0.5f;
            glm::vec3 centre = (surfMin + surfMax) * 0.5f;

            JPH::BoxShapeSettings shapeSettings(
                JPH::Vec3(halfExtents.x, halfExtents.y, halfExtents.z)
            );
            shapeSettings.SetEmbedded();

            auto shapeResult = shapeSettings.Create();
            if (!shapeResult.IsValid()) continue;

            JPH::BodyCreationSettings bodySettings(
                shapeResult.Get(),
                JPH::RVec3(centre.x, centre.y, centre.z),
                JPH::Quat::sIdentity(),
                JPH::EMotionType::Static,
                Layers::NON_MOVING
            );

            bodyInterface.CreateAndAddBody(bodySettings, JPH::EActivation::DontActivate);
        }
    }
}
```

Static bodies don't need an ECS entity — they're part of the world geometry and never move or get destroyed during gameplay.

---

## Step 6: Create Dynamic Bodies for Physics Entities

Entities that have `Position`, `Velocity`, and `AABBCollider` (our physics-driven cubes) need dynamic Jolt bodies.

### Update entity creation in `scene_setup.cpp`

For each physics entity (e.g., the test cubes), create a Jolt body and attach a `JoltBody` component:

```cpp
void createDynamicBody(entt::registry& registry, entt::entity entity)
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
        JPH::EMotionType::Dynamic,
        Layers::MOVING
    );

    // Match our gravity strength
    bodySettings.mGravityFactor = 1.0f;

    // Set initial velocity if the entity has one
    if (registry.all_of<Velocity>(entity))
    {
        auto& vel = registry.get<Velocity>(entity);
        bodySettings.mLinearVelocity = JPH::Vec3(vel.value.x, vel.value.y, vel.value.z);
    }

    JPH::BodyID bodyId = bodyInterface.CreateAndAddBody(
        bodySettings, JPH::EActivation::Activate
    );

    registry.emplace<JoltBody>(entity, bodyId);
}
```

Call `createDynamicBody` for each physics test cube after creating it. The falling cube, the sliding cube, etc.

---

## Step 7: The Jolt Sync System

This system reads Jolt body transforms and writes them back into ECS `Position` (and optionally `Velocity`) components. It replaces `movementSystem` and `groundDetectionSystem`.

### New file: `src/engine/ecs/systems/jolt_sync_system.h`

```cpp
#pragma once

#include <entt/entt.hpp>

void joltSyncSystem(entt::registry& registry);
```

### New file: `src/engine/ecs/systems/jolt_sync_system.cpp`

```cpp
#include "engine/ecs/systems/jolt_sync_system.h"
#include "engine/ecs/components.h"
#include "engine/physics/jolt_world.h"

void joltSyncSystem(entt::registry& registry)
{
    auto& jolt = registry.ctx().get<JoltWorld>();
    auto& bodyInterface = jolt.getBodyInterface();

    auto view = registry.view<Position, JoltBody>();
    for (auto [entity, pos, joltBody] : view.each())
    {
        // Read position from Jolt
        JPH::RVec3 joltPos = bodyInterface.GetCenterOfMassPosition(joltBody.id);
        pos.value = glm::vec3(joltPos.GetX(), joltPos.GetY(), joltPos.GetZ());

        // Sync velocity if the entity has one
        if (registry.all_of<Velocity>(entity))
        {
            JPH::Vec3 joltVel = bodyInterface.GetLinearVelocity(joltBody.id);
            auto& vel = registry.get<Velocity>(entity);
            vel.value = glm::vec3(joltVel.GetX(), joltVel.GetY(), joltVel.GetZ());
        }

        // Update ground state if the entity has one
        if (registry.all_of<OnGround>(entity))
        {
            auto& ground = registry.get<OnGround>(entity);
            // A body is "on ground" if it has very low vertical velocity
            // and is not in free-fall. This is a simple heuristic —
            // Chapter 15 replaces this with CharacterVirtual's ground detection.
            JPH::Vec3 joltVel = bodyInterface.GetLinearVelocity(joltBody.id);
            ground.value = std::abs(joltVel.GetY()) < 0.5f;
        }
    }
}
```

---

## Step 8: Wire It Up in main.cpp

### Remove old includes

Remove these includes (the systems are being replaced):

```cpp
// REMOVE these:
#include "engine/ecs/systems/collision_system.h"
#include "engine/ecs/systems/movement_system.h"
#include "engine/ecs/systems/physics_system.h"
#include "engine/physics/spatial_hash.h"
#include "engine/physics/physics_config.h"
```

### Add new includes

```cpp
#include "engine/physics/jolt_world.h"
#include "engine/ecs/systems/jolt_sync_system.h"
```

### Initialize Jolt before the game loop

Replace the old `PhysicsConfig` and `SpatialHash` setup:

```cpp
// REMOVE:
// auto& physicsConfig = registry.ctx().emplace<PhysicsConfig>();
// SpatialHash spatialHash(4.0f);

// ADD:
auto& joltWorld = registry.ctx().emplace<JoltWorld>();
joltWorld.init();
```

### Create bodies after scene setup

After `setupScene` returns:

```cpp
Level level = setupScene(registry, resources);

// Create Jolt bodies from the level geometry
createLevelBodies(registry, level);

// Create Jolt bodies for dynamic entities
auto dynamicView = registry.view<Position, AABBCollider, Velocity>();
for (auto entity : dynamicView)
{
    if (!registry.all_of<TagPlayer>(entity))  // Player is handled in Ch15
    {
        createDynamicBody(registry, entity);
    }
}

// Optimise the broad-phase after all bodies are added
joltWorld.physicsSystem->OptimizeBroadPhase();
```

### Update the tick loop

Replace the old system calls:

```cpp
while (fixedTimestep.step())
{
    weaponSwitchSystem(registry);
    playerMovementSystem(registry);

    // REMOVED: physicsSystem(registry);
    moverSystem(registry);
    // REMOVED: collisionSystem(registry, spatialHash, level);
    // REMOVED: movementSystem(registry);

    // NEW: Step Jolt physics and sync transforms back to ECS
    joltWorld.step(physicsConfig.fixedDeltaTime);
    joltSyncSystem(registry);

    // REMOVED: groundDetectionSystem(registry, level);
    combatSystem(registry, level);
    lifetimeSystem(registry);
    triggerSystem(registry);
    demoResetSystem(registry);
}
```

> **Note:** We still keep `PhysicsConfig` for the fixed timestep delta, or you can hardcode `1.0f / 60.0f`. Either way, the `fixedDeltaTime` value must match what you pass to `joltWorld.step()`.

### Shutdown Jolt before exit

Before `return 0;` in `main()`:

```cpp
joltWorld.shutdown();
resources.clear();
return 0;
```

---

## Step 9: Update CMakeLists.txt Source Files

### Remove old physics files

Remove these from the `add_executable` source list:

```cmake
# REMOVE:
src/engine/ecs/systems/collision_system.cpp
src/engine/ecs/systems/movement_system.cpp
src/engine/ecs/systems/physics_system.cpp
src/engine/physics/collision.cpp
src/engine/physics/spatial_hash.cpp
```

### Add new files

```cmake
# ADD:
src/engine/physics/jolt_world.cpp
src/engine/ecs/systems/jolt_sync_system.cpp
```

> **Don't delete** the old `.cpp` and `.h` files from disk yet — just remove them from the build. You might want to reference them later, and they serve as documentation of how the custom physics worked.

---

## What Changed — Summary

| File | Change |
|------|--------|
| `CMakeLists.txt` | Added Jolt FetchContent, removed old physics sources, added new sources |
| `jolt_setup.h` | **New** — Jolt includes, layer definitions, filter classes, callbacks |
| `jolt_world.h` | **New** — Physics world wrapper with init/step/shutdown |
| `jolt_world.cpp` | **New** — Translation unit for Jolt symbols |
| `jolt_sync_system.h/cpp` | **New** — Reads Jolt body transforms into ECS components |
| `components.h` | Added `JoltBody` component, Jolt includes |
| `scene_setup.cpp` | Added `createLevelBodies()`, `createDynamicBody()` |
| `main.cpp` | Replaced old physics setup and system calls with Jolt |

### Files removed from build (not deleted)

| File | Was |
|------|-----|
| `collision_system.cpp/h` | AABB sweep collision detection |
| `movement_system.cpp/h` | Velocity integration (`pos += vel * dt`) |
| `physics_system.cpp/h` | Gravity, friction, ground detection |
| `collision.cpp/h` | Sweep AABB helper (Minkowski method) |
| `spatial_hash.cpp/h` | Broad-phase spatial hashing |

---

## What You Should See

After building and running:

1. **The room is solid** — walls and floor block movement (via Jolt static bodies instead of sweep AABB)
2. **Test cubes fall and land cleanly** — no jitter, no micro-bouncing. Jolt's resting contact solver keeps them perfectly still once they settle
3. **The sliding cube slides and stops** — friction is handled by Jolt's material system
4. **The lift and door still work** — `moverSystem` is unchanged, it animates position independently of physics
5. **The player can't move yet** — that's expected! The player entity doesn't have a Jolt body; we deliberately skipped it. Chapter 15 adds a proper character controller

### Troubleshooting

**Build fails with "cannot find Jolt/Jolt.h":**
- FetchContent needs an internet connection on first build
- Check that `SOURCE_SUBDIR Build` is in your `FetchContent_Declare`
- Clean the build directory and re-run CMake

**Objects fall through the floor:**
- Check that `createLevelBodies` is called after `setupScene`
- Verify the surface fattening produces non-zero thickness bodies
- Call `OptimizeBroadPhase()` after adding all bodies

**Cubes explode or fly away:**
- Check initial positions aren't overlapping other bodies
- Jolt resolves interpenetration aggressively — make sure cubes spawn above surfaces

**Linker errors about Jolt symbols:**
- Ensure `Jolt` is in `target_link_libraries`
- Make sure `jolt_world.cpp` exists and is in the source list

---

## New C++ Concept: FetchContent

CMake's `FetchContent` module downloads and builds external dependencies automatically:

```cmake
FetchContent_Declare(
    JoltPhysics                                          # Name (your choice)
    GIT_REPOSITORY https://github.com/jrouwe/JoltPhysics.git  # Where
    GIT_TAG        v5.2.0                                # Which version
    SOURCE_SUBDIR  Build                                 # Where CMakeLists.txt lives
)
FetchContent_MakeAvailable(JoltPhysics)                  # Download & add_subdirectory
```

This is the modern alternative to git submodules. The download happens once at configure time and is cached in `_deps/` inside your build directory. You can pin a specific version with `GIT_TAG` — always do this in real projects to avoid surprises.

---

## New C++ Concept: EMotionType

Jolt bodies have three motion types:

| Type | Description | Use For |
|------|-------------|---------|
| `Static` | Never moves. Infinite mass. | Level geometry (floors, walls) |
| `Dynamic` | Fully simulated. Responds to forces. | Physics objects (cubes, barrels) |
| `Kinematic` | Moves via code, not forces. Pushes dynamic bodies. | Moving platforms (lifts, doors) |

A `Kinematic` body is perfect for our `Mover` entities — Chapter 15 converts them to kinematic Jolt bodies so lifts and doors push the player physically.

---

## What's Next

The physics world is running and objects behave correctly, but the player is frozen — they have no Jolt body. In **Chapter 15**, we'll add a `CharacterVirtual` controller for the player with proper ground detection, stair stepping, and jumping. We'll also convert movers to kinematic bodies so lifts carry the player automatically.
