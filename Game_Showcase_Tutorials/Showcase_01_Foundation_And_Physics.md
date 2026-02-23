# Showcase 1: Foundation & Physics

> **Prerequisites:** Complete through Chapter 10a (Game Loop & Physics Cleanup). Your project should have the `FixedTimestep` class, `PhysicsConfig`, `CollisionLayers`, **multi-point-light rendering**, and no test cubes remaining.

---

## What We Are Building

A multi-room dev showcase — a playable test level that demonstrates every engine feature built so far. This is not a game level; it is a **museum for your code**. Each room isolates a feature so you can verify it works correctly.

After this tutorial you will have:

- A three-room level with grid-textured walls, floors, and ceilings
- A dedicated light room with minimal ambient, demonstrating coloured point lights and blending
- Physics demos that loop on a timer so you can watch them repeatedly
- A clear foundation to expand in future showcases

---

## Resources Needed

Before starting, you need a few grid textures. These are standard dev textures used across the industry.

| Resource | File Name | Size | Purpose |
|----------|-----------|------|---------|
| Orange grid | `grid_orange.png` | 512x512 | Main wall texture — high visibility with scale markings |
| Grey grid | `grid_grey.png` | 512x512 | Floors and ceilings — neutral, non-distracting |
| Blue grid | `grid_blue.png` | 512x512 | Secondary room differentiation |

**Where to get them:**

- **Kenney.nl Prototype Textures** — free pack, CC0 licence (no attribution required). Download from [kenney.nl/assets/prototype-textures](https://kenney.nl/assets/prototype-textures). Pick any orange, grey, and blue variants.
- **Make your own** — create a 512x512 image, draw a grid with lines every 64px (representing 1 metre at standard UV scale). Fill alternating cells with two shades of your chosen colour.

Place them in your project at:
```
assets/textures/grid_orange.png
assets/textures/grid_grey.png
assets/textures/grid_blue.png
```

You will also reuse `cube.obj` (already in `assets/models/`) and your existing shaders.

---

## Step 1: Load the New Textures

In `main.cpp`, add texture loads alongside your existing ones:

```cpp
auto gridOrange = resources.getTexture("grid_orange", "assets/textures/grid_orange.png");
auto gridGrey   = resources.getTexture("grid_grey",   "assets/textures/grid_grey.png");
auto gridBlue   = resources.getTexture("grid_blue",   "assets/textures/grid_blue.png");
```

These will also need to be available in `setupScene`. You can either pass them as parameters or use `resources.getTexture("grid_orange")` inside `setupScene` (since `ResourceManager` caches loaded textures and returns them by name).

---

## Step 2: The DemoReset Component

The physics demos (falling cubes, sliding cubes) are one-shot events — they happen at startup, then everything sits still. A showcase should loop so you can watch each demo repeatedly.

Add this component to `src/engine/ecs/components.h`:

```cpp
// ─── Showcase Components ─────────────────────────────────────────

struct DemoReset
{
    glm::vec3 startPosition;
    glm::vec3 startVelocity = glm::vec3(0.0f);
    float interval = 5.0f;    // seconds between resets
    float timer    = 0.0f;    // counts up each tick
};
```

The idea is simple: each demo entity remembers its starting position and velocity. A system counts up each tick, and when the timer exceeds the interval, it snaps the entity back to its starting state.

### The DemoReset System

Create `src/engine/ecs/systems/demo_reset_system.h`:

```cpp
// engine/ecs/systems/demo_reset_system.h
#pragma once

#include <entt/entt.hpp>

void demoResetSystem(entt::registry& registry, float dt);
```

Create `src/engine/ecs/systems/demo_reset_system.cpp`:

```cpp
#include "engine/ecs/systems/demo_reset_system.h"
#include "engine/ecs/components.h"

void demoResetSystem(entt::registry& registry, float dt)
{
    auto view = registry.view<Position, Velocity, DemoReset>();

    for (auto [entity, pos, vel, demo] : view.each())
    {
        demo.timer += dt;

        if (demo.timer >= demo.interval)
        {
            pos.value = demo.startPosition;
            vel.value = demo.startVelocity;
            demo.timer = 0.0f;

            // Reset ground state so gravity applies immediately
            if (registry.all_of<OnGround>(entity))
            {
                registry.get<OnGround>(entity).value = false;
            }
        }
    }
}
```

### Wiring It Into the Game Loop

Add `demoResetSystem` to your fixed timestep loop in `main.cpp`, after `groundDetectionSystem`:

```cpp
#include "engine/ecs/systems/demo_reset_system.h"

// ...

while (fixedTimestep.step())
{
    physicsSystem(registry);
    collisionSystem(registry, spatialHash, level);
    movementSystem(registry);
    groundDetectionSystem(registry, level);
    demoResetSystem(registry, fixedTimestep.getTimestep());  // reset demo entities
}
```

It runs last because it needs to override whatever physics just did to the entity. The reset should be the final word each tick.

---

## Step 3: Design the Showcase Level

We are building three connected rooms. Here is the layout:

```
               20m
    ┌──────────────────────────┐
    │                          │
    │       MAIN HALL          │  12m
    │       (20 x 12 x 4)     │
    │                          │
    │   Grid-textured walls    │
    │   Sunlight               │
    │   2 point light exhibits │
    │                          │
    ├───────────┐   ┌──────────┤
    │           │   │          │
    │           │   │  PHYSICS │
    │  LIGHT    │   │   LAB    │  10m
    │  ROOM     │   │(10x10x4)│
    │ (8x8x4)  │   │          │
    │           │   │ Looping  │
    │ Coloured  │   │ cubes,   │
    │ lights,   │   │ friction │
    │ low       │   │ demo     │
    │ ambient   │   │          │
    └───────────┘   └──────────┘
        8m              10m
```

The rooms are connected by open doorways (gaps in the walls). All measurements are in world units (approximately metres).

### Light Room Design

The Light Room needs special attention. The directional sun illuminates everything equally — we do not have shadow mapping yet, so there is no way to make the sun "not reach" an enclosed room. The light is calculated per-fragment against the surface normal regardless of geometry between the light and the surface.

To make the point lights the dominant visual feature in the Light Room, we use two tricks:

1. **Low directional ambient** — the sun's ambient strength is already 0.1f, which is quite dim. The directional light's diffuse contribution depends on surface normals; walls facing away from the sun direction will naturally be darker.
2. **Bright point lights** — we make the RGB point lights significantly brighter than the ambient, so they visually dominate even where the sun contributes some light.

This is not perfect — true darkness requires shadow mapping (a future chapter). But the coloured pools should be clearly visible and their blending should be obvious where radii overlap.

---

## Step 4: Build the Level Geometry

Replace your `createTestLevel()` function in `src/engine/ecs/scene_setup.cpp` with the showcase level. This is a larger level but follows the exact same pattern as the original box room — just more surfaces.

We will build each room as a separate sector.

```cpp
static Level createShowcaseLevel()
{
    Level level;

    // ═══════════════════════════════════════════════════════════
    // Sector 0: MAIN HALL (20 x 12 x 4)
    //   Origin: (0, 0, 0) to (20, 4, 12)
    //   Doorways:
    //     - Left wall (x=0): opening from z=4 to z=8 (to Light Room)
    //     - Front wall (z=12): opening from x=12 to x=16 (to Physics Lab)
    // ═══════════════════════════════════════════════════════════
    {
        Sector hall;
        hall.id = 0;
        hall.boundsMin = glm::vec3(0.0f, 0.0f, 0.0f);
        hall.boundsMax = glm::vec3(20.0f, 4.0f, 12.0f);

        // Floor
        hall.surfaces.push_back({
            {glm::vec3(0,0,12), glm::vec3(20,0,12), glm::vec3(20,0,0), glm::vec3(0,0,0)},
            glm::vec3(0, 1, 0),
            "grid_grey.png", 0
        });

        // Ceiling
        hall.surfaces.push_back({
            {glm::vec3(0,4,0), glm::vec3(20,4,0), glm::vec3(20,4,12), glm::vec3(0,4,12)},
            glm::vec3(0, -1, 0),
            "grid_grey.png", 0
        });

        // Back wall (z=0, full width, normal +z)
        hall.surfaces.push_back({
            {glm::vec3(20,4,0), glm::vec3(0,4,0), glm::vec3(0,0,0), glm::vec3(20,0,0)},
            glm::vec3(0, 0, 1),
            "grid_orange.png", 0
        });

        // Front wall (z=12, normal -z)
        // Split into two sections with a doorway gap at x=12 to x=16
        // Left section: x=0 to x=12
        hall.surfaces.push_back({
            {glm::vec3(0,4,12), glm::vec3(12,4,12), glm::vec3(12,0,12), glm::vec3(0,0,12)},
            glm::vec3(0, 0, -1),
            "grid_orange.png", 0
        });
        // Right section: x=16 to x=20
        hall.surfaces.push_back({
            {glm::vec3(16,4,12), glm::vec3(20,4,12), glm::vec3(20,0,12), glm::vec3(16,0,12)},
            glm::vec3(0, 0, -1),
            "grid_orange.png", 0
        });
        // Above doorway: x=12 to x=16, y=3 to y=4
        hall.surfaces.push_back({
            {glm::vec3(12,4,12), glm::vec3(16,4,12), glm::vec3(16,3,12), glm::vec3(12,3,12)},
            glm::vec3(0, 0, -1),
            "grid_orange.png", 0
        });

        // Left wall (x=0, normal +x)
        // Split with doorway from z=4 to z=8
        // Bottom section: z=0 to z=4
        hall.surfaces.push_back({
            {glm::vec3(0,4,0), glm::vec3(0,4,4), glm::vec3(0,0,4), glm::vec3(0,0,0)},
            glm::vec3(1, 0, 0),
            "grid_orange.png", 0
        });
        // Top section: z=8 to z=12
        hall.surfaces.push_back({
            {glm::vec3(0,4,8), glm::vec3(0,4,12), glm::vec3(0,0,12), glm::vec3(0,0,8)},
            glm::vec3(1, 0, 0),
            "grid_orange.png", 0
        });
        // Above doorway: z=4 to z=8, y=3 to y=4
        hall.surfaces.push_back({
            {glm::vec3(0,4,4), glm::vec3(0,4,8), glm::vec3(0,3,8), glm::vec3(0,3,4)},
            glm::vec3(1, 0, 0),
            "grid_orange.png", 0
        });

        // Right wall (x=20, full, normal -x)
        hall.surfaces.push_back({
            {glm::vec3(20,4,12), glm::vec3(20,4,0), glm::vec3(20,0,0), glm::vec3(20,0,12)},
            glm::vec3(-1, 0, 0),
            "grid_orange.png", 0
        });

        level.sectors.push_back(std::move(hall));
    }

    // ═══════════════════════════════════════════════════════════
    // Sector 1: LIGHT ROOM (8 x 8 x 4)
    //   Position: x = -8 to 0, z = 2 to 10, y = 0 to 4
    //   Doorway: right wall (x=0) from z=4 to z=8, matching hall
    // ═══════════════════════════════════════════════════════════
    {
        Sector lightRoom;
        lightRoom.id = 1;
        lightRoom.boundsMin = glm::vec3(-8.0f, 0.0f, 2.0f);
        lightRoom.boundsMax = glm::vec3(0.0f, 4.0f, 10.0f);

        // Floor
        lightRoom.surfaces.push_back({
            {glm::vec3(-8,0,10), glm::vec3(0,0,10), glm::vec3(0,0,2), glm::vec3(-8,0,2)},
            glm::vec3(0, 1, 0),
            "grid_grey.png", 1
        });

        // Ceiling
        lightRoom.surfaces.push_back({
            {glm::vec3(-8,4,2), glm::vec3(0,4,2), glm::vec3(0,4,10), glm::vec3(-8,4,10)},
            glm::vec3(0, -1, 0),
            "grid_grey.png", 1
        });

        // Back wall (z=2, normal +z)
        lightRoom.surfaces.push_back({
            {glm::vec3(0,4,2), glm::vec3(-8,4,2), glm::vec3(-8,0,2), glm::vec3(0,0,2)},
            glm::vec3(0, 0, 1),
            "grid_blue.png", 1
        });

        // Front wall (z=10, normal -z)
        lightRoom.surfaces.push_back({
            {glm::vec3(-8,4,10), glm::vec3(0,4,10), glm::vec3(0,0,10), glm::vec3(-8,0,10)},
            glm::vec3(0, 0, -1),
            "grid_blue.png", 1
        });

        // Left wall (x=-8, normal +x)
        lightRoom.surfaces.push_back({
            {glm::vec3(-8,4,2), glm::vec3(-8,4,10), glm::vec3(-8,0,10), glm::vec3(-8,0,2)},
            glm::vec3(1, 0, 0),
            "grid_blue.png", 1
        });

        // Right wall (x=0, normal -x)
        // Doorway from z=4 to z=8 (matching the hall's left wall opening)
        // Bottom section: z=2 to z=4
        lightRoom.surfaces.push_back({
            {glm::vec3(0,4,2), glm::vec3(0,4,4), glm::vec3(0,0,4), glm::vec3(0,0,2)},
            glm::vec3(-1, 0, 0),
            "grid_blue.png", 1
        });
        // Top section: z=8 to z=10
        lightRoom.surfaces.push_back({
            {glm::vec3(0,4,8), glm::vec3(0,4,10), glm::vec3(0,0,10), glm::vec3(0,0,8)},
            glm::vec3(-1, 0, 0),
            "grid_blue.png", 1
        });
        // Above doorway: z=4 to z=8, y=3 to y=4
        lightRoom.surfaces.push_back({
            {glm::vec3(0,4,4), glm::vec3(0,4,8), glm::vec3(0,3,8), glm::vec3(0,3,4)},
            glm::vec3(-1, 0, 0),
            "grid_blue.png", 1
        });

        level.sectors.push_back(std::move(lightRoom));
    }

    // ═══════════════════════════════════════════════════════════
    // Sector 2: PHYSICS LAB (10 x 10 x 4)
    //   Position: x = 12 to 22, z = 12 to 22, y = 0 to 4
    //   Doorway: back wall (z=12) from x=12 to x=16, matching hall
    // ═══════════════════════════════════════════════════════════
    {
        Sector physLab;
        physLab.id = 2;
        physLab.boundsMin = glm::vec3(12.0f, 0.0f, 12.0f);
        physLab.boundsMax = glm::vec3(22.0f, 4.0f, 22.0f);

        // Floor
        physLab.surfaces.push_back({
            {glm::vec3(12,0,22), glm::vec3(22,0,22), glm::vec3(22,0,12), glm::vec3(12,0,12)},
            glm::vec3(0, 1, 0),
            "grid_grey.png", 2
        });

        // Ceiling
        physLab.surfaces.push_back({
            {glm::vec3(12,4,12), glm::vec3(22,4,12), glm::vec3(22,4,22), glm::vec3(12,4,22)},
            glm::vec3(0, -1, 0),
            "grid_grey.png", 2
        });

        // Back wall (z=12, normal +z)
        // Doorway from x=12 to x=16 (matching hall)
        // Right section only: x=16 to x=22
        physLab.surfaces.push_back({
            {glm::vec3(22,4,12), glm::vec3(16,4,12), glm::vec3(16,0,12), glm::vec3(22,0,12)},
            glm::vec3(0, 0, 1),
            "grid_orange.png", 2
        });
        // Above doorway: x=12 to x=16, y=3 to y=4
        physLab.surfaces.push_back({
            {glm::vec3(16,4,12), glm::vec3(12,4,12), glm::vec3(12,3,12), glm::vec3(16,3,12)},
            glm::vec3(0, 0, 1),
            "grid_orange.png", 2
        });

        // Front wall (z=22, normal -z)
        physLab.surfaces.push_back({
            {glm::vec3(12,4,22), glm::vec3(22,4,22), glm::vec3(22,0,22), glm::vec3(12,0,22)},
            glm::vec3(0, 0, -1),
            "grid_orange.png", 2
        });

        // Left wall (x=12, normal +x)
        physLab.surfaces.push_back({
            {glm::vec3(12,4,12), glm::vec3(12,4,22), glm::vec3(12,0,22), glm::vec3(12,0,12)},
            glm::vec3(1, 0, 0),
            "grid_orange.png", 2
        });

        // Right wall (x=22, normal -x)
        physLab.surfaces.push_back({
            {glm::vec3(22,4,22), glm::vec3(22,4,12), glm::vec3(22,0,12), glm::vec3(22,0,22)},
            glm::vec3(-1, 0, 0),
            "grid_orange.png", 2
        });

        // ── Shelf: a raised platform to drop cubes from ──
        // A 4m x 4m platform at y=2 in the back-right corner
        physLab.surfaces.push_back({
            {glm::vec3(18,2,22), glm::vec3(22,2,22), glm::vec3(22,2,18), glm::vec3(18,2,18)},
            glm::vec3(0, 1, 0),
            "grid_grey.png", 2
        });

        // Shelf front face (z=18, y=0 to y=2, x=18 to x=22)
        physLab.surfaces.push_back({
            {glm::vec3(18,2,18), glm::vec3(22,2,18), glm::vec3(22,0,18), glm::vec3(18,0,18)},
            glm::vec3(0, 0, -1),
            "grid_orange.png", 2
        });

        // Shelf left face (x=18, y=0 to y=2, z=18 to z=22)
        physLab.surfaces.push_back({
            {glm::vec3(18,2,18), glm::vec3(18,2,22), glm::vec3(18,0,22), glm::vec3(18,0,18)},
            glm::vec3(-1, 0, 0),
            "grid_orange.png", 2
        });

        level.sectors.push_back(std::move(physLab));
    }

    buildSectorMeshes(level);
    return level;
}
```

### Important Notes

- The winding order follows the same convention as your existing `createTestLevel()`: vertices wound **counter-clockwise when viewed from inside** the room.
- Doorways are created by splitting a wall surface into sections with a gap. The gap has no surface, so nothing renders there and nothing blocks collision.
- The shelf in the Physics Lab is a horizontal surface at y=2 with front and side faces. Cubes placed above it will fall and land on it.
- The `sectorId` field (last integer in each surface) matches the sector's `id` for potential future BSP/PVS use.

---

## Step 5: Set Up the Showcase Entities

Update `setupScene()` to populate the showcase with demonstration entities. Note the `DemoReset` components on physics demo entities — these make the demos loop automatically.

```cpp
Level setupScene(entt::registry& registry, const ResourceManager& resources)
{
    auto litShader   = resources.getShader("lit");
    auto gridOrange  = resources.getTexture("grid_orange");
    auto gridGrey    = resources.getTexture("grid_grey");
    auto gridBlue    = resources.getTexture("grid_blue");
    auto cubeMesh    = resources.getMesh("cube");

    // ─── Create the showcase level ──────────────────────────────
    Level level = createShowcaseLevel();

    for (const auto& sector : level.sectors)
    {
        if (!sector.mesh) continue;

        auto sectorEntity = registry.create();
        registry.emplace<Position>(sectorEntity, glm::vec3(0.0f));

        // Pick texture based on sector id
        unsigned int texId = gridOrange->getId(); // default
        if (sector.id == 1) texId = gridBlue->getId();  // light room

        registry.emplace<MeshRenderer>
        (
            sectorEntity,
            sector.mesh->getVAO(),
            0u,
            litShader->getId(),
            texId,
            true,
            sector.mesh->getIndexCount()
        );
    }

    // ═══════════════════════════════════════════════════════════
    // MAIN HALL — Sunlight + point light contrast
    // ═══════════════════════════════════════════════════════════

    // Sun light (directional) — low ambient so it doesn't wash out point lights
    auto sun = registry.create();
    registry.emplace<DirectionalLight>
    (
        sun,
        glm::vec3(-0.2f, -1.0f, -0.3f),   // direction
        glm::vec3(1.0f, 0.95f, 0.8f),      // warm white
        0.08f                                // low ambient
    );

    // Point light 1: warm torch near left wall
    auto hallLight1 = registry.create();
    registry.emplace<Position>(hallLight1, glm::vec3(4.0f, 2.5f, 6.0f));
    registry.emplace<PointLight>
    (
        hallLight1,
        glm::vec3(2.0f, 1.4f, 0.6f),   // warm orange
        0.05f, 0.045f, 0.0075f
    );

    // Point light 2: cool blue near right side
    auto hallLight2 = registry.create();
    registry.emplace<Position>(hallLight2, glm::vec3(16.0f, 2.5f, 6.0f));
    registry.emplace<PointLight>
    (
        hallLight2,
        glm::vec3(0.4f, 0.6f, 2.0f),   // cool blue
        0.05f, 0.045f, 0.0075f
    );

    // A static cube in the main hall (reference object for scale/collision)
    auto hallCube = registry.create();
    registry.emplace<Position>(hallCube, glm::vec3(10.0f, 0.5f, 6.0f));
    registry.emplace<AABBCollider>(hallCube, glm::vec3(0.5f), false);
    registry.emplace<MeshRenderer>
    (
        hallCube,
        cubeMesh->getVAO(), 0u,
        litShader->getId(), gridOrange->getId(),
        true, cubeMesh->getIndexCount()
    );

    // ═══════════════════════════════════════════════════════════
    // LIGHT ROOM — Coloured point lights in low ambient
    //
    // Without shadow mapping, the directional light still
    // contributes some illumination here. We counter this by
    // using bright point lights that visually dominate. The
    // coloured pools and their blending should still be clearly
    // visible against the blue grid walls.
    // ═══════════════════════════════════════════════════════════

    // Red light — back-left corner
    auto redLight = registry.create();
    registry.emplace<Position>(redLight, glm::vec3(-6.0f, 2.0f, 4.0f));
    registry.emplace<PointLight>
    (
        redLight,
        glm::vec3(3.0f, 0.2f, 0.2f),   // bright red
        0.02f, 0.09f, 0.032f
    );

    // Green light — back-right area
    auto greenLight = registry.create();
    registry.emplace<Position>(greenLight, glm::vec3(-2.0f, 2.0f, 4.0f));
    registry.emplace<PointLight>
    (
        greenLight,
        glm::vec3(0.2f, 3.0f, 0.2f),   // bright green
        0.02f, 0.09f, 0.032f
    );

    // Blue light — front-centre
    auto blueLight = registry.create();
    registry.emplace<Position>(blueLight, glm::vec3(-4.0f, 2.0f, 8.0f));
    registry.emplace<PointLight>
    (
        blueLight,
        glm::vec3(0.2f, 0.2f, 3.0f),   // bright blue
        0.02f, 0.09f, 0.032f
    );

    // White accent light — centre of room, shows how all three
    // blend together and provides a neutral reference
    auto whiteLight = registry.create();
    registry.emplace<Position>(whiteLight, glm::vec3(-4.0f, 3.0f, 6.0f));
    registry.emplace<PointLight>
    (
        whiteLight,
        glm::vec3(1.0f, 1.0f, 1.0f),   // neutral white
        0.02f, 0.14f, 0.07f             // shorter range (tighter pool)
    );

    // ═══════════════════════════════════════════════════════════
    // PHYSICS LAB — Gravity, collision, and friction demos
    //
    // All physics demo entities have a DemoReset component so
    // they loop automatically on a timer.
    // ═══════════════════════════════════════════════════════════

    // Cube 1: on the shelf, nudged off the edge → falls to floor
    {
        glm::vec3 startPos(19.0f, 3.5f, 19.0f);
        glm::vec3 startVel(-0.5f, 0.0f, 0.0f);

        auto cube = registry.create();
        registry.emplace<Position>(cube, startPos);
        registry.emplace<Velocity>(cube, startVel);
        registry.emplace<AABBCollider>(cube, glm::vec3(0.5f), false);
        registry.emplace<Gravity>(cube);
        registry.emplace<OnGround>(cube);
        registry.emplace<DemoReset>(cube, startPos, startVel, 6.0f, 0.0f);
        registry.emplace<MeshRenderer>
        (
            cube,
            cubeMesh->getVAO(), 0u,
            litShader->getId(), gridOrange->getId(),
            true, cubeMesh->getIndexCount()
        );
    }

    // Cube 2: dropped from near the ceiling (pure gravity test)
    {
        glm::vec3 startPos(15.0f, 3.5f, 17.0f);
        glm::vec3 startVel(0.0f);

        auto cube = registry.create();
        registry.emplace<Position>(cube, startPos);
        registry.emplace<Velocity>(cube, startVel);
        registry.emplace<AABBCollider>(cube, glm::vec3(0.5f), false);
        registry.emplace<Gravity>(cube);
        registry.emplace<OnGround>(cube);
        registry.emplace<DemoReset>(cube, startPos, startVel, 4.0f, 0.0f);
        registry.emplace<MeshRenderer>
        (
            cube,
            cubeMesh->getVAO(), 0u,
            litShader->getId(), gridOrange->getId(),
            true, cubeMesh->getIndexCount()
        );
    }

    // Cube 3: sliding across the floor (friction demo)
    {
        glm::vec3 startPos(14.0f, 0.5f, 14.0f);
        glm::vec3 startVel(3.0f, 0.0f, 1.0f);

        auto cube = registry.create();
        registry.emplace<Position>(cube, startPos);
        registry.emplace<Velocity>(cube, startVel);
        registry.emplace<AABBCollider>(cube, glm::vec3(0.5f), false);
        registry.emplace<Gravity>(cube);
        registry.emplace<OnGround>(cube);
        registry.emplace<CharacterPhysics>(cube); // provides friction values
        registry.emplace<DemoReset>(cube, startPos, startVel, 5.0f, 0.0f);
        registry.emplace<MeshRenderer>
        (
            cube,
            cubeMesh->getVAO(), 0u,
            litShader->getId(), gridOrange->getId(),
            true, cubeMesh->getIndexCount()
        );
    }

    // Point light in physics lab for visibility
    auto labLight = registry.create();
    registry.emplace<Position>(labLight, glm::vec3(17.0f, 3.0f, 17.0f));
    registry.emplace<PointLight>
    (
        labLight,
        glm::vec3(1.5f, 1.5f, 1.5f),   // bright white
        0.1f, 0.045f, 0.0075f
    );

    return level;
}
```

---

## Step 6: Update the Camera Start Position

The showcase level is larger and positioned differently from the original box room. Update the camera start position in `main.cpp` so the player spawns in the main hall:

```cpp
Camera camera(glm::vec3(10.0f, 1.7f, 3.0f));  // centre of main hall, eye height
```

---

## Step 7: Build and Test

Build and run. You should see:

### Main Hall
- A large open room with orange grid walls and grey floor/ceiling
- Warm orange point light on the left, cool blue on the right
- A static cube in the centre — walk into it to verify collision works
- Directional sunlight illuminating the whole space

### Light Room (through the left doorway)
- Blue-tinted grid walls
- Four point lights: red (back-left), green (back-right), blue (front-centre), white (centre ceiling)
- Coloured pools visible on walls and floor
- Where light radii overlap, colours blend additively (red + green = yellow, red + blue = magenta, etc.)
- The white accent light provides a neutral reference so you can see how colours affect the same surface differently

### Physics Lab (through the front doorway)
- A raised shelf (platform at y=2) in the back-right corner
- **Cube 1**: nudged off the shelf, falls to the ground. Resets every 6 seconds
- **Cube 2**: dropped from near the ceiling, pure gravity fall. Resets every 4 seconds
- **Cube 3**: sliding across the floor, decelerating via friction. Resets every 5 seconds
- All cubes loop — watch them repeatedly to verify consistent behaviour

---

## What to Look For

Use this checklist to verify features:

- [ ] **Grid textures render correctly** — lines visible, scale looks reasonable
- [ ] **Doorways work** — you can walk between rooms through the gaps
- [ ] **Directional light** — visible as overall scene illumination
- [ ] **Multiple point lights** — you can see more than one coloured pool simultaneously (confirms Ch 10a multi-light fix works)
- [ ] **Point light blending** — in the light room, overlapping lights produce mixed colours
- [ ] **Point light attenuation** — light intensity fades with distance from each source
- [ ] **Static cube collision** — you cannot walk through the hall cube (if player collision is implemented)
- [ ] **Gravity** — falling cubes drop and hit the ground/shelf
- [ ] **Ground detection** — cubes stop falling when they land (not falling through the floor)
- [ ] **Friction** — the sliding cube decelerates and stops
- [ ] **Wall collision** — cubes do not pass through walls
- [ ] **Demo reset** — physics cubes snap back to their start positions after their timer expires

---

## Troubleshooting

**Only one point light renders:**
You need the multi-point-light changes from Chapter 10a. Check that `lit.frag` uses a `PointLightData` struct array and that `render_system.cpp` loops over all point lights instead of breaking after the first.

**Cubes fall through the floor:**
Check that `groundDetectionSystem` is running after `movementSystem` in your game loop. Verify that the floor surface has `normal.y >= 0.7f` (your ground detection check).

**Cubes do not reset:**
Ensure `demoResetSystem` is in your fixed timestep loop and that physics demo entities have all three required components: `Position`, `Velocity`, and `DemoReset`.

**Rooms are invisible:**
Make sure the textures are loading correctly — check the console for file-not-found errors. Verify that `buildSectorMeshes(level)` is called at the end of `createShowcaseLevel()`.

**Doorways are blocked:**
The doorway gaps rely on there being **no surface** in that area. If collision still blocks you, check that your collision system tests against level surfaces, not against sector bounding boxes.

**Light room looks the same as the main hall:**
Without shadow mapping, the directional light illuminates everything equally. The point lights should still be visible as bright coloured pools *on top of* the ambient lighting. If the point lights are not visible at all, check the multi-point-light fix above.

---

## Summary

You now have a multi-room dev showcase that demonstrates:

| Feature | Where to See It |
|---------|----------------|
| Textured rendering | All rooms — grid textures on every surface |
| Mesh loading | OBJ cubes placed throughout |
| Directional lighting | Global sun illumination |
| Multiple point lights | Main hall (warm/cool), light room (RGBW), physics lab (white) |
| Light colour blending | Light room — overlapping coloured lights mix additively |
| Level geometry | Three connected sectors with doorways |
| AABB collision | Walk into cubes and walls |
| Gravity | Falling cubes in physics lab |
| Ground detection | Cubes landing on floor and shelf |
| Friction | Sliding cube decelerating to a stop |
| Demo reset loop | Physics cubes reset on a timer for repeated viewing |

This showcase will be expanded in Showcase 2 (after Chapter 15a) to add doors, lifts, triggers, weapons, items, enemies, and a HUD.

---

*Next up: Back to the main tutorial track — **Chapter 11: Doors, Lifts & Triggers**.*
