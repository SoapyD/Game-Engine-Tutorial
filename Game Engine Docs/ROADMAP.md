# Quake-Style ECS Game Engine Tutorial — Roadmap

## Project Name: **QEngine**

A ground-up 3D FPS engine in C++ using ECS architecture, inspired by Quake.

---

## Tech Stack

| Purpose | Library | Why |
|---|---|---|
| ECS | EnTT | Header-only, modern C++, industry-respected |
| Windowing/Input | GLFW | Cross-platform, lightweight, standard |
| OpenGL Loading | GLAD | Generates only what you need |
| Math | GLM | Mirrors GLSL syntax, header-only |
| Image Loading | stb_image | Single header, no dependencies |
| Model Loading | Assimp (later) or tinyobjloader | Start simple, upgrade later |
| Audio | miniaudio | Single header, cross-platform |
| Networking | ENet | Reliable UDP, proven in games |
| Build System | CMake | Industry standard |

---

## Chapter Outline

### Phase 1: Foundation (Chapters 0-3)
Getting a window open, triangles on screen, and the ECS wired up.

| Ch | Title | Key Concepts |
|----|-------|-------------|
| 0 | Dev Environment Setup | CMake, vcpkg/manual deps, project structure |
| 1 | Window & OpenGL Context | GLFW, GLAD, OpenGL core profile, game loop |
| 2 | Shader System | GLSL, vertex/fragment shaders, shader class |
| 3 | ECS Foundation | EnTT, entities, components, systems, queries |

**Milestone:** Coloured triangle rendered via ECS.

---

### Phase 2: 3D Rendering (Chapters 4-7)
Going from triangles to a textured 3D world you can walk around in.

| Ch | Title | Key Concepts |
|----|-------|-------------|
| 4 | 3D Transforms & Camera | Model/View/Projection, GLM, FPS camera |
| 5 | Textures & Materials | stb_image, UV mapping, texture units |
| 6 | Mesh & Model Loading | Vertex data, index buffers, OBJ loading |
| 7 | Lighting | Phong/Blinn-Phong, normals, point/directional lights |

**Milestone:** Textured, lit 3D scene with FPS camera movement.

---

### Phase 3: Game World (Chapters 8-11)
Building out the level and interacting with it.

| Ch | Title | Key Concepts |
|----|-------|-------------|
| 8 | Level Geometry & BSP | BSP concepts, level loading, PVS (simplified) |
| 9 | Collision Detection | AABB, raycasting, swept collision, spatial hashing |
| 10 | Physics & Movement | Gravity, friction, Quake-style air control, stair stepping |
| 11 | Doors, Lifts & Triggers | Entity interactions, trigger volumes, state machines |

**Milestone:** Walkable level with collision, physics, and interactive elements.

---

### Phase 4: Player & Gameplay (Chapters 12-15a)
Giving the player a real body and turning the engine into a playable game.

| Ch | Title | Key Concepts |
|----|-------|-------------|
| 12 | Weapons & Projectiles | Hitscan, projectile entities, damage system |
| 13 | Player Body & Debug HUD | Physical player body, reversed camera sync, input→velocity pipeline, health clamping, stb_easy_font debug overlay |
| 14 | Moon Jumping & Platforming | Low-gravity tuning, variable jump height, coyote time, platforming test arena, moving platform riding |
| 15 | Gameplay Polish | Crosshair, basic HUD, death/respawn, damage feedback, knockback |
| 15a | Cleanup | Refactor scene_setup, fix typos, consolidate systems, document tick order |

**Milestone:** Player with a physical body that jumps between platforms, rides lifts, takes lava damage, and dies/respawns. Debug HUD showing health and FPS.

---

### Phase 5: TrenchBroom Integration (Chapters 16-20)
Replacing the hardcoded level with a proper level editor workflow.

| Ch | Title | Key Concepts |
|----|-------|-------------|
| 16 | .map Parser & Brush Rendering | TrenchBroom .map format, plane→polygon→triangle conversion, brush-to-mesh, texture mapping |
| 17 | Entity Mapping & FGD | FGD entity definitions, classname→ECS factory functions, point vs brush entities |
| 18 | Brush Collision | Collision from brush planes, Minkowski expansion against faces, spatial hash with brush AABBs |
| 19 | TrenchBroom Config & Workflow | GameConfig.cfg, texture browser integration, hot reload, test level construction |
| 20 | Final Level & Integration | Complete playable TrenchBroom level exercising all features: rooms, doors, lifts, lava, platforms, weapons, pickups, lighting |

**Milestone:** Fully playable level authored in TrenchBroom with all engine features working end-to-end.

---

## Dependency Graph

```
Ch 0 (Setup)
  └── Ch 1 (Window)
       └── Ch 2 (Shaders)
            ├── Ch 3 (ECS)
            │    └── ALL subsequent chapters depend on this
            └── Ch 4 (3D Transforms)
                 ├── Ch 5 (Textures)
                 │    └── Ch 6 (Models)
                 │         └── Ch 7 (Lighting)
                 │              └── Ch 8 (Levels/BSP)
                 │                   └── Ch 9 (Collision)
                 │                        └── Ch 10 (Physics)
                 │                             ├── Ch 11 (Triggers)
                 │                             └── Ch 12 (Weapons)
                 │                                  └── Ch 13 (Player Body)
                 │                                       └── Ch 14 (Platforming)
                 │                                            └── Ch 15 (Gameplay Polish)
                 │                                                 └── Ch 15a (Cleanup)
                 │                                                      └── Ch 16 (.map Parser)
                 │                                                           └── Ch 17 (Entity Mapping)
                 │                                                                └── Ch 18 (Brush Collision)
                 │                                                                     └── Ch 19 (TB Config)
                 │                                                                          └── Ch 20 (Final Level)
```

---

## Progress Tracker

| Chapter | Status | Notes |
|---------|--------|-------|
| 0 | Complete | Dev environment, CMake, dependencies |
| 1 | Complete | GLFW window, OpenGL context, game loop, delta time |
| 2 | Complete | GLSL shaders, Shader class, first triangle |
| 3 | Complete | EnTT, components, systems, triangle as entity |
| 4 | Complete | Model/View/Projection, FPS camera, WASD + mouse look |
| 5 | Complete | stb_image, Texture class, UV coords, filtering, tiling |
| 6 | Complete | Vertex struct, Mesh class, OBJ loader, index buffers, procedural meshes |
| 7 | Complete | Phong lighting, directional/point lights, light components, lightmap concepts |
| 8 | Complete | Sectors, surfaces, portals, level loading, simplified BSP, backface culling |
| 9 | Complete | AABB, raycasting, swept collision, spatial hashing, Minkowski difference |
| 10 | Complete | Gravity, friction, fixed timestep, jumping, Quake air control, stair stepping |
| 11 | Complete | State machines, trigger volumes, doors, lifts, teleporters, damage zones |
| 12 | Complete | Hitscan/projectile weapons, splash damage, rocket jumping, weapon switching |
| 13 | Complete | Physical player body, reversed camera sync, input→velocity, health clamping, stb_easy_font debug HUD |
| 14 | Planned | Low-gravity tuning, variable jump height, coyote time, platforming test arena |
| 15 | Planned | Crosshair, basic HUD, death/respawn, damage feedback |
| 15a | Planned | Refactor scene_setup, fix typos, consolidate systems |
| 16 | Planned | TrenchBroom .map parser, plane→polygon conversion, brush-to-mesh rendering |
| 17 | Planned | FGD entity definitions, classname→ECS factory mapping |
| 18 | Planned | Collision from brush planes, Minkowski expansion, spatial hash integration |
| 19 | Planned | TrenchBroom GameConfig.cfg, texture browser, hot reload workflow |
| 20 | Planned | Complete playable TrenchBroom level with all features integrated |
