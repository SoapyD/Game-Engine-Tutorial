# Chapter 8: Level Geometry & BSP

## What You'll Learn
- How Quake represents its world (brushes, planes, and BSP trees)
- What a BSP tree is and why it exists
- A simplified level format for QEngine
- Loading and rendering level geometry
- Potentially Visible Set (PVS) — rendering only what the player can see
- Backface culling

---

## How Quake Builds a Level

In Quake, level designers don't place triangles. They work with **brushes** — convex 3D shapes (boxes, wedges, cylinders) that are combined to carve out rooms and corridors. A level compiler then:

1. Takes all the brushes
2. Clips them against each other to find the actual surfaces
3. Builds a **BSP tree** that partitions the space
4. Calculates **PVS** (which areas can see which other areas)
5. Generates **lightmaps**
6. Outputs a `.bsp` file the engine loads at runtime

Building a full BSP compiler is a large project on its own. For QEngine, we'll take a practical approach:

- **Understand** BSP concepts (you need this knowledge)
- **Use a simplified level format** that's easy to create and load
- **Implement BSP traversal** for rendering and collision queries
- Skip the full compiler — we'll build levels from predefined geometry

---

## What Is a BSP Tree?

BSP stands for **Binary Space Partitioning**. It recursively divides 3D space into two halves using planes.

### The Problem It Solves

Given a room with many walls, how do you know which walls are in front of which? Before hardware Z-buffers were fast, this was a real problem. BSP trees solve it by sorting geometry spatially.

### How It Works

1. Pick a plane (usually aligned with a wall surface)
2. Everything in **front** of the plane goes in the left subtree
3. Everything **behind** the plane goes in the right subtree
4. Repeat recursively until each leaf contains a small set of polygons (or is empty/solid)

```
         [Plane A]
        ╱         ╲
   Front A      Behind A
   [Plane B]    [Plane C]
   ╱    ╲       ╱    ╲
 ...    ...   ...    ...
```

### Leaves

The leaf nodes of a BSP tree represent **volumes** of space:
- **Empty leaves** — open space the player can occupy
- **Solid leaves** — inside walls, the player can't be here

This is incredibly useful for:
- **Collision** — if a point is in a solid leaf, it's inside a wall
- **Rendering** — only draw geometry in empty, visible leaves
- **PVS** — pre-compute which leaves can see each other

---

## Our Simplified Level Format

Instead of a full BSP compiler, we'll define levels as a list of **sectors** (rooms) connected by **portals** (openings). Each sector contains a list of surfaces (walls, floor, ceiling).

### The Level File Format (.qlvl)

A simple text format:

```
# QEngine Level Format
# sector <id> <min_x> <min_y> <min_z> <max_x> <max_y> <max_z>
# surface <sector_id> <type> <x1> <y1> <z1> <x2> <y2> <z2> <x3> <y3> <z3> <x4> <y4> <z4> <texture>
# portal <sector_a> <sector_b> <x1> <y1> <z1> <x2> <y2> <z2> <x3> <y3> <z3> <x4> <y4> <z4>
# entity <type> <x> <y> <z> [properties...]

sector 0 -10 0 -10 10 4 0
surface 0 floor -10 0 0 10 0 0 10 0 -10 -10 0 -10 floor_stone.png
surface 0 ceil -10 4 -10 10 4 -10 10 4 0 -10 4 0 ceil_dark.png
surface 0 wall -10 0 0 10 0 0 10 4 0 -10 4 0 wall_brick.png
surface 0 wall -10 0 -10 -10 0 0 -10 4 0 -10 4 -10 wall_brick.png
surface 0 wall 10 0 0 10 0 -10 10 4 -10 10 4 0 wall_brick.png
surface 0 wall -10 0 -10 10 0 -10 10 4 -10 -10 4 -10 wall_brick.png

sector 1 -10 0 0 10 4 10
portal 0 1 -3 0 0 3 0 0 3 4 0 -3 4 0

entity player_start 0 1 -5
entity light 3 3 -5 color 1.0 0.8 0.4
```

This is intentionally simple. Real engines use binary formats for speed — we use text for readability and easy editing.

---

## Level Data Structures

### src/engine/level/level.h

```cpp
#pragma once

#include <glm/glm.hpp>
#include <vector>
#include <string>
#include <memory>
#include "engine/renderer/mesh.h"

struct Surface {
    glm::vec3 vertices[4];     // Quad corners
    glm::vec3 normal;          // Computed from vertices
    std::string textureName;
    unsigned int textureId = 0;
};

struct Portal {
    int sectorA;
    int sectorB;
    glm::vec3 vertices[4];     // The opening between sectors
};

struct Sector {
    int id;
    glm::vec3 boundsMin;       // Axis-aligned bounding box
    glm::vec3 boundsMax;
    std::vector<Surface> surfaces;
    std::vector<int> portalIndices;  // Indices into Level::portals
    std::unique_ptr<Mesh> mesh;      // Combined mesh of all surfaces
};

struct LevelEntity {
    std::string type;          // "player_start", "light", "enemy", etc.
    glm::vec3 position;
    std::vector<std::pair<std::string, std::string>> properties;
};

struct Level {
    std::vector<Sector> sectors;
    std::vector<Portal> portals;
    std::vector<LevelEntity> entities;
};
```

### C++ Concept: `std::unique_ptr`

```cpp
std::unique_ptr<Mesh> mesh;
```

A `unique_ptr` is a smart pointer that owns its object exclusively — no sharing. When the `unique_ptr` is destroyed, the owned object is destroyed too.

Unlike `shared_ptr` (which uses reference counting), `unique_ptr` has **zero overhead** — it's the same cost as a raw pointer. Use it when there's a single clear owner.

```cpp
auto mesh = std::make_unique<Mesh>(vertices, indices);  // Create
mesh->getVAO();      // Use (-> works like a raw pointer)
// mesh is automatically deleted when it goes out of scope
```

You can't copy a `unique_ptr` (that would mean two owners), but you can **move** it:

```cpp
std::unique_ptr<Mesh> a = std::make_unique<Mesh>(...);
std::unique_ptr<Mesh> b = std::move(a);  // a is now nullptr, b owns it
```

---

## Loading the Level

### src/engine/level/level_loader.h

```cpp
#pragma once

#include "engine/level/level.h"
#include <string>
#include <unordered_map>

class LevelLoader {
public:
    Level load(const std::string& path,
               const std::unordered_map<std::string, unsigned int>& textureMap);

private:
    void parseSector(const std::string& line, Level& level);
    void parseSurface(const std::string& line, Level& level,
                      const std::unordered_map<std::string, unsigned int>& textureMap);
    void parsePortal(const std::string& line, Level& level);
    void parseEntity(const std::string& line, Level& level);

    void buildSectorMeshes(Level& level);
    glm::vec3 computeNormal(const glm::vec3& v0, const glm::vec3& v1,
                            const glm::vec3& v2);
};
```

### src/engine/level/level_loader.cpp

```cpp
#include "engine/level/level_loader.h"
#include <fstream>
#include <sstream>
#include <iostream>

Level LevelLoader::load(const std::string& path,
                         const std::unordered_map<std::string, unsigned int>& textureMap) {
    Level level;

    std::ifstream file(path);
    if (!file.is_open()) {
        std::cerr << "ERROR: Could not open level file: " << path << std::endl;
        return level;
    }

    std::string line;
    while (std::getline(file, line)) {
        // Skip comments and empty lines
        if (line.empty() || line[0] == '#') continue;

        std::istringstream iss(line);
        std::string type;
        iss >> type;

        if (type == "sector") {
            parseSector(line, level);
        } else if (type == "surface") {
            parseSurface(line, level, textureMap);
        } else if (type == "portal") {
            parsePortal(line, level);
        } else if (type == "entity") {
            parseEntity(line, level);
        }
    }

    buildSectorMeshes(level);

    std::cout << "Loaded level: " << path
              << " (" << level.sectors.size() << " sectors, "
              << level.portals.size() << " portals, "
              << level.entities.size() << " entities)" << std::endl;

    return level;
}

void LevelLoader::parseSector(const std::string& line, Level& level) {
    std::istringstream iss(line);
    std::string type;
    Sector sector;

    iss >> type >> sector.id
        >> sector.boundsMin.x >> sector.boundsMin.y >> sector.boundsMin.z
        >> sector.boundsMax.x >> sector.boundsMax.y >> sector.boundsMax.z;

    // Ensure vector is large enough
    if (sector.id >= static_cast<int>(level.sectors.size())) {
        level.sectors.resize(sector.id + 1);
    }
    level.sectors[sector.id] = std::move(sector);
}

void LevelLoader::parseSurface(const std::string& line, Level& level,
                                const std::unordered_map<std::string, unsigned int>& textureMap) {
    std::istringstream iss(line);
    std::string type, surfaceType;
    int sectorId;
    Surface surface;

    iss >> type >> sectorId >> surfaceType;

    for (int i = 0; i < 4; i++) {
        iss >> surface.vertices[i].x
            >> surface.vertices[i].y
            >> surface.vertices[i].z;
    }

    iss >> surface.textureName;

    // Compute face normal from first three vertices
    surface.normal = computeNormal(
        surface.vertices[0], surface.vertices[1], surface.vertices[2]);

    // Look up texture ID
    auto it = textureMap.find(surface.textureName);
    if (it != textureMap.end()) {
        surface.textureId = it->second;
    }

    if (sectorId < static_cast<int>(level.sectors.size())) {
        level.sectors[sectorId].surfaces.push_back(surface);
    }
}

void LevelLoader::parsePortal(const std::string& line, Level& level) {
    std::istringstream iss(line);
    std::string type;
    Portal portal;

    iss >> type >> portal.sectorA >> portal.sectorB;

    for (int i = 0; i < 4; i++) {
        iss >> portal.vertices[i].x
            >> portal.vertices[i].y
            >> portal.vertices[i].z;
    }

    int portalIndex = static_cast<int>(level.portals.size());
    level.portals.push_back(portal);

    // Link portal to both sectors
    if (portal.sectorA < static_cast<int>(level.sectors.size())) {
        level.sectors[portal.sectorA].portalIndices.push_back(portalIndex);
    }
    if (portal.sectorB < static_cast<int>(level.sectors.size())) {
        level.sectors[portal.sectorB].portalIndices.push_back(portalIndex);
    }
}

void LevelLoader::parseEntity(const std::string& line, Level& level) {
    std::istringstream iss(line);
    std::string type;
    LevelEntity entity;

    iss >> type >> entity.type
        >> entity.position.x >> entity.position.y >> entity.position.z;

    // Parse remaining key-value properties
    std::string key, value;
    while (iss >> key >> value) {
        entity.properties.push_back({key, value});
    }

    level.entities.push_back(entity);
}

glm::vec3 LevelLoader::computeNormal(const glm::vec3& v0, const glm::vec3& v1,
                                      const glm::vec3& v2) {
    glm::vec3 edge1 = v1 - v0;
    glm::vec3 edge2 = v2 - v0;
    return glm::normalize(glm::cross(edge1, edge2));
}

void LevelLoader::buildSectorMeshes(Level& level) {
    for (auto& sector : level.sectors) {
        std::vector<Vertex> vertices;
        std::vector<unsigned int> indices;

        for (const auto& surface : sector.surfaces) {
            unsigned int baseIndex = static_cast<unsigned int>(vertices.size());

            // Calculate UV coordinates based on surface dimensions
            // Simple planar projection for now
            float uScale = glm::length(surface.vertices[1] - surface.vertices[0]);
            float vScale = glm::length(surface.vertices[3] - surface.vertices[0]);

            // Four vertices for the quad
            vertices.push_back({surface.vertices[0], surface.normal, {0.0f, 0.0f}});
            vertices.push_back({surface.vertices[1], surface.normal, {uScale, 0.0f}});
            vertices.push_back({surface.vertices[2], surface.normal, {uScale, vScale}});
            vertices.push_back({surface.vertices[3], surface.normal, {0.0f, vScale}});

            // Two triangles for the quad
            indices.push_back(baseIndex + 0);
            indices.push_back(baseIndex + 1);
            indices.push_back(baseIndex + 2);
            indices.push_back(baseIndex + 0);
            indices.push_back(baseIndex + 2);
            indices.push_back(baseIndex + 3);
        }

        if (!vertices.empty()) {
            sector.mesh = std::make_unique<Mesh>(vertices, indices);
        }
    }
}
```

---

## Backface Culling

Triangles have a front face and a back face, determined by the order of their vertices (clockwise vs counter-clockwise when viewed from the front). The inside of a wall should never be visible.

**Backface culling** tells OpenGL to skip triangles facing away from the camera:

```cpp
glEnable(GL_CULL_FACE);
glCullFace(GL_BACK);           // Cull back faces (default)
glFrontFace(GL_CCW);           // Counter-clockwise = front (default)
```

Add this once during initialisation (after GLAD loads). This roughly halves the fragment shader workload — the GPU never even starts shading invisible back faces.

---

## Sector-Based Visibility (Simplified PVS)

Full PVS pre-computes a bit table: "from sector X, can you see sector Y?" This avoids rendering sectors behind walls.

Our simplified version at runtime:

1. Determine which sector the camera is in (point-in-AABB test)
2. Render that sector's geometry
3. For each portal in that sector, check if it's visible (frustum test)
4. If visible, render the connected sector too
5. Recurse (with a depth limit to prevent infinite loops)

```cpp
// Pseudo-code for portal-based rendering
void renderLevel(const Level& level, const Camera& camera, int maxDepth) {
    int cameraSector = findSectorContaining(level, camera.getPosition());
    if (cameraSector < 0) return;

    std::vector<bool> rendered(level.sectors.size(), false);
    renderSectorRecursive(level, cameraSector, camera, rendered, maxDepth);
}

void renderSectorRecursive(const Level& level, int sectorId,
                            const Camera& camera,
                            std::vector<bool>& rendered, int depth) {
    if (depth <= 0 || rendered[sectorId]) return;
    rendered[sectorId] = true;

    // Draw this sector's mesh
    drawSectorMesh(level.sectors[sectorId]);

    // Check portals
    for (int portalIdx : level.sectors[sectorId].portalIndices) {
        const Portal& portal = level.portals[portalIdx];
        int otherSector = (portal.sectorA == sectorId)
                          ? portal.sectorB : portal.sectorA;

        if (isPortalVisible(portal, camera)) {
            renderSectorRecursive(level, otherSector, camera,
                                  rendered, depth - 1);
        }
    }
}
```

### Finding Which Sector the Camera Is In

```cpp
int findSectorContaining(const Level& level, const glm::vec3& point) {
    for (size_t i = 0; i < level.sectors.size(); i++) {
        const auto& s = level.sectors[i];
        if (point.x >= s.boundsMin.x && point.x <= s.boundsMax.x &&
            point.y >= s.boundsMin.y && point.y <= s.boundsMax.y &&
            point.z >= s.boundsMin.z && point.z <= s.boundsMax.z) {
            return static_cast<int>(i);
        }
    }
    return -1;  // Not in any sector
}
```

This is a simple AABB (Axis-Aligned Bounding Box) containment test. For non-axis-aligned sectors, you'd need point-in-convex-hull tests — but axis-aligned sectors keep things simple.

---

## Creating a Test Level

You can create a simple test level programmatically while the file format matures:

```cpp
Level createTestLevel() {
    Level level;

    // Room 1: a simple box room
    Sector room1;
    room1.id = 0;
    room1.boundsMin = glm::vec3(-5.0f, 0.0f, -5.0f);
    room1.boundsMax = glm::vec3(5.0f, 4.0f, 5.0f);

    // Floor
    room1.surfaces.push_back({
        {glm::vec3(-5,0,5), glm::vec3(5,0,5), glm::vec3(5,0,-5), glm::vec3(-5,0,-5)},
        glm::vec3(0, 1, 0),  // normal: up
        "floor_stone.png", 0
    });

    // Ceiling
    room1.surfaces.push_back({
        {glm::vec3(-5,4,-5), glm::vec3(5,4,-5), glm::vec3(5,4,5), glm::vec3(-5,4,5)},
        glm::vec3(0, -1, 0),  // normal: down
        "ceil_dark.png", 0
    });

    // Four walls
    // Front wall (z = 5)
    room1.surfaces.push_back({
        {glm::vec3(-5,0,5), glm::vec3(5,0,5), glm::vec3(5,4,5), glm::vec3(-5,4,5)},
        glm::vec3(0, 0, -1),  // normal: into the room
        "wall_brick.png", 0
    });

    // Back wall (z = -5)
    room1.surfaces.push_back({
        {glm::vec3(5,0,-5), glm::vec3(-5,0,-5), glm::vec3(-5,4,-5), glm::vec3(5,4,-5)},
        glm::vec3(0, 0, 1),
        "wall_brick.png", 0
    });

    // Left wall (x = -5)
    room1.surfaces.push_back({
        {glm::vec3(-5,0,-5), glm::vec3(-5,0,5), glm::vec3(-5,4,5), glm::vec3(-5,4,-5)},
        glm::vec3(1, 0, 0),
        "wall_brick.png", 0
    });

    // Right wall (x = 5)
    room1.surfaces.push_back({
        {glm::vec3(5,0,5), glm::vec3(5,0,-5), glm::vec3(5,4,-5), glm::vec3(5,4,5)},
        glm::vec3(-1, 0, 0),
        "wall_brick.png", 0
    });

    level.sectors.push_back(std::move(room1));

    // Build meshes for all sectors
    // (call buildSectorMeshes or do it manually)

    return level;
}
```

This gives you a 10x4x10 box room to walk around in. Texture it with any image, add a light, and you have the beginnings of a Quake-like level.

---

## C++ Concept: Recursion

The portal rendering uses **recursion** — a function that calls itself:

```cpp
void renderSectorRecursive(..., int depth) {
    if (depth <= 0) return;  // Base case: stop recursing
    // ... do work ...
    renderSectorRecursive(..., depth - 1);  // Recursive call
}
```

Every recursive function needs:
1. A **base case** that stops the recursion (here: `depth <= 0` or `already rendered`)
2. A step that moves toward the base case (here: `depth - 1`)

Without a base case, you get infinite recursion → stack overflow → crash.

---

## What's Next

In **Chapter 9**, we'll add collision detection — AABB tests, raycasting, and spatial hashing. This is what prevents the player from walking through walls and enables bullets to hit targets.
