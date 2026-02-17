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

### Phase 4: Gameplay (Chapters 12-15)
Turning the engine into a game.

| Ch | Title | Key Concepts |
|----|-------|-------------|
| 12 | Weapons & Projectiles | Hitscan, projectile entities, damage system |
| 13 | Items & Pickups | Health, ammo, armour, respawning items |
| 14 | Enemy AI | State machines, pathfinding basics, line of sight |
| 15 | HUD & UI | 2D rendering, health bar, ammo counter, crosshair |

**Milestone:** Playable single-player FPS with enemies, items, and HUD.

---

### Phase 5: Audio (Chapter 16)

| Ch | Title | Key Concepts |
|----|-------|-------------|
| 16 | Audio System | miniaudio, 3D positional audio, sound effects, music |

**Milestone:** Spatial audio for weapons, footsteps, and ambience.

---

### Phase 6: Multiplayer (Chapters 17-19)
The hardest part — networking.

| Ch | Title | Key Concepts |
|----|-------|-------------|
| 17 | Networking Foundation | Client-server model, ENet, reliable/unreliable channels |
| 18 | State Synchronisation | Snapshots, delta compression, entity interpolation |
| 19 | Client-Side Prediction | Input prediction, server reconciliation, lag compensation |

**Milestone:** Two players in the same level, shooting each other over a network.

---

### Phase 7: Polish (Chapter 20)

| Ch | Title | Key Concepts |
|----|-------|-------------|
| 20 | Particles, Effects & Polish | Particle system, muzzle flash, explosions, screen shake |

**Milestone:** A polished, complete Quake-like FPS.

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
                 │                             ├── Ch 12 (Weapons)
                 │                             │    └── Ch 13 (Items)
                 │                             │         └── Ch 14 (AI)
                 │                             │              └── Ch 15 (HUD)
                 │                             └── Ch 16 (Audio)
                 └── Ch 17 (Networking)
                      └── Ch 18 (State Sync)
                           └── Ch 19 (Prediction)

Ch 20 (Polish) depends on everything
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
| 13 | Complete | Health/ammo/armour/weapon pickups, respawning, item bob, damage absorption |
| 14 | Complete | FSM AI, line of sight, A* pathfinding, nav graphs, enemy types as data |
| 15 | Complete | Orthographic HUD, crosshair, health/ammo bars, bitmap fonts, damage flash |
| 16 | Complete | miniaudio setup, AudioManager class, 3D positional audio, attenuation, AudioSource/PlaySoundOnce components |
| 17 | Complete | Client-server architecture, ENet, PacketWriter/Reader serialization, NetworkId, game loop split |
| 18 | Complete | Server snapshots, delta compression, InterpolationBuffer, entity creation/destruction over network |
| 19 | Complete | Input buffering, local prediction, server reconciliation, lag compensation with PositionHistory rewind |
| 20 | Complete | ParticlePool (object pool), billboard rendering, emitters, screen shake, view bob, interpolation functions |
