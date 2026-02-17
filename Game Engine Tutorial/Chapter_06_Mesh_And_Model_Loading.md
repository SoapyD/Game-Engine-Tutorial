# Chapter 6: Mesh & Model Loading

## What You'll Learn
- Index buffers — eliminating duplicate vertices
- The OBJ file format — a simple text-based 3D model format
- Building a Mesh class that handles GPU upload
- Loading models from files
- Resource management basics — loading once, sharing across entities

---

## The Problem with Duplicate Vertices

In Chapter 5, our quad used 6 vertices for 2 triangles:

```
Triangle 1: bottom-left, bottom-right, top-right
Triangle 2: bottom-left, top-right, top-left
```

Bottom-left and top-right appear twice. For a quad that's 2 wasted vertices. For a complex model with thousands of triangles sharing vertices, the waste is enormous.

**Index buffers** solve this. Instead of listing every vertex for every triangle, we:
1. List each unique vertex once
2. Use an array of indices to say "triangle 1 uses vertices 0, 1, 2; triangle 2 uses vertices 0, 2, 3"

```
Vertices (unique):
  0: bottom-left
  1: bottom-right
  2: top-right
  3: top-left

Indices:
  0, 1, 2,    ← Triangle 1
  0, 2, 3     ← Triangle 2
```

4 vertices + 6 indices instead of 6 vertices. The savings scale dramatically with complex meshes.

---

## Vertex Data Structure

Let's define a proper vertex structure instead of using raw float arrays:

```cpp
// In a new file or in components.h

struct Vertex {
    glm::vec3 position;
    glm::vec3 normal;      // For lighting (Chapter 7)
    glm::vec2 texCoords;
};
```

### C++ Concept: Struct Memory Layout

`Vertex` contains 3 + 3 + 2 = 8 floats = 32 bytes per vertex. In C++, struct members are laid out sequentially in memory:

```
[posX, posY, posZ, normX, normY, normZ, texU, texV]
 ◄── position ──►  ◄──── normal ────►  ◄─ uv ─►
```

This contiguous layout is exactly what OpenGL's `glVertexAttribPointer` expects, so we can pass a vector of `Vertex` structs directly to the GPU.

---

## Building the Mesh Class

### src/engine/renderer/mesh.h

```cpp
#pragma once

#include <glad/glad.h>
#include <glm/glm.hpp>
#include <vector>
#include <string>

struct Vertex {
    glm::vec3 position;
    glm::vec3 normal;
    glm::vec2 texCoords;
};

class Mesh {
public:
    Mesh(const std::vector<Vertex>& vertices, const std::vector<unsigned int>& indices);
    ~Mesh();

    // Prevent copying (GPU resources shouldn't be duplicated accidentally)
    Mesh(const Mesh&) = delete;
    Mesh& operator=(const Mesh&) = delete;

    // Allow moving (transfer ownership)
    Mesh(Mesh&& other) noexcept;
    Mesh& operator=(Mesh&& other) noexcept;

    unsigned int getVAO() const { return m_vao; }
    unsigned int getIndexCount() const { return m_indexCount; }

private:
    unsigned int m_vao = 0;
    unsigned int m_vbo = 0;
    unsigned int m_ebo = 0;   // Element Buffer Object (index buffer)
    unsigned int m_indexCount = 0;

    void setupMesh(const std::vector<Vertex>& vertices,
                   const std::vector<unsigned int>& indices);
    void cleanup();
};
```

### C++ Concept: Delete and Move

```cpp
Mesh(const Mesh&) = delete;              // No copying
Mesh(Mesh&& other) noexcept;             // Moving is allowed
```

**Why no copying?** A Mesh owns GPU resources (VAO, VBO, EBO). If you copy a Mesh, both copies would think they own the same GPU resources. When one is destroyed, it deletes the resources — and the other now has dangling handles.

**Move semantics** solve this. When you "move" a Mesh, ownership transfers. The source gives up its handles (set to 0), and the destination takes them.

```cpp
Mesh a = loadMesh("cube.obj");    // a owns the GPU resources
Mesh b = std::move(a);            // b now owns them, a is empty
// a's destructor does nothing (handles are 0)
// b's destructor cleans up
```

### src/engine/renderer/mesh.cpp

```cpp
#include "engine/renderer/mesh.h"

Mesh::Mesh(const std::vector<Vertex>& vertices,
           const std::vector<unsigned int>& indices) {
    m_indexCount = static_cast<unsigned int>(indices.size());
    setupMesh(vertices, indices);
}

Mesh::~Mesh() {
    cleanup();
}

Mesh::Mesh(Mesh&& other) noexcept
    : m_vao(other.m_vao)
    , m_vbo(other.m_vbo)
    , m_ebo(other.m_ebo)
    , m_indexCount(other.m_indexCount)
{
    other.m_vao = 0;
    other.m_vbo = 0;
    other.m_ebo = 0;
    other.m_indexCount = 0;
}

Mesh& Mesh::operator=(Mesh&& other) noexcept {
    if (this != &other) {
        cleanup();
        m_vao = other.m_vao;
        m_vbo = other.m_vbo;
        m_ebo = other.m_ebo;
        m_indexCount = other.m_indexCount;
        other.m_vao = 0;
        other.m_vbo = 0;
        other.m_ebo = 0;
        other.m_indexCount = 0;
    }
    return *this;
}

void Mesh::setupMesh(const std::vector<Vertex>& vertices,
                     const std::vector<unsigned int>& indices) {
    glGenVertexArrays(1, &m_vao);
    glGenBuffers(1, &m_vbo);
    glGenBuffers(1, &m_ebo);

    glBindVertexArray(m_vao);

    // Upload vertex data
    glBindBuffer(GL_ARRAY_BUFFER, m_vbo);
    glBufferData(GL_ARRAY_BUFFER,
                 vertices.size() * sizeof(Vertex),
                 vertices.data(),
                 GL_STATIC_DRAW);

    // Upload index data
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, m_ebo);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER,
                 indices.size() * sizeof(unsigned int),
                 indices.data(),
                 GL_STATIC_DRAW);

    // Vertex attribute layout:
    // Position (location 0): 3 floats at offset 0
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex),
                          (void*)offsetof(Vertex, position));
    glEnableVertexAttribArray(0);

    // Normal (location 1): 3 floats at offset of 'normal' member
    glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex),
                          (void*)offsetof(Vertex, normal));
    glEnableVertexAttribArray(1);

    // Tex coords (location 2): 2 floats at offset of 'texCoords' member
    glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, sizeof(Vertex),
                          (void*)offsetof(Vertex, texCoords));
    glEnableVertexAttribArray(2);

    glBindVertexArray(0);
}

void Mesh::cleanup() {
    if (m_vao) glDeleteVertexArrays(1, &m_vao);
    if (m_vbo) glDeleteBuffers(1, &m_vbo);
    if (m_ebo) glDeleteBuffers(1, &m_ebo);
    m_vao = m_vbo = m_ebo = 0;
}
```

### C++ Concept: `offsetof`

```cpp
(void*)offsetof(Vertex, normal)
```

`offsetof` is a macro that returns the byte offset of a member within a struct. For our `Vertex`:

```
offsetof(Vertex, position)  = 0   (starts at the beginning)
offsetof(Vertex, normal)    = 12  (after 3 floats = 12 bytes)
offsetof(Vertex, texCoords) = 24  (after 6 floats = 24 bytes)
```

This is cleaner than manually calculating `(void*)(3 * sizeof(float))`.

### C++ Concept: `std::vector`

```cpp
std::vector<Vertex> vertices;
std::vector<unsigned int> indices;
```

`std::vector` is a dynamic array — it grows and shrinks as needed. Key operations:

```cpp
vertices.push_back(vertex);     // Add to end
vertices.size();                 // Number of elements
vertices.data();                 // Raw pointer to underlying array (for OpenGL)
vertices[0];                     // Access by index
vertices.clear();                // Remove all elements
```

Internally, a vector is a contiguous block of memory on the heap. When you `push_back` past its capacity, it allocates a larger block and copies everything over. For performance-critical code, you can `reserve()` capacity upfront:

```cpp
vertices.reserve(10000);  // Allocate space for 10000 vertices (no copies needed)
```

---

## Loading OBJ Files

The OBJ format is a simple text-based 3D model format. Here's a cube:

```obj
# Positions
v -0.5 -0.5  0.5
v  0.5 -0.5  0.5
v  0.5  0.5  0.5
v -0.5  0.5  0.5
# ... more vertices

# Texture coordinates
vt 0.0 0.0
vt 1.0 0.0
vt 1.0 1.0
vt 0.0 1.0

# Normals
vn 0.0 0.0 1.0

# Faces (vertex/texcoord/normal indices, 1-based)
f 1/1/1 2/2/1 3/3/1
f 1/1/1 3/3/1 4/4/1
```

Each line starts with a type prefix:
- `v` — vertex position
- `vt` — texture coordinate
- `vn` — normal vector
- `f` — face (triangle), referencing indices into the v/vt/vn lists

### src/engine/renderer/obj_loader.h

```cpp
#pragma once

#include "engine/renderer/mesh.h"
#include <string>

Mesh loadOBJ(const std::string& path);
```

### src/engine/renderer/obj_loader.cpp

```cpp
#include "engine/renderer/obj_loader.h"
#include <fstream>
#include <sstream>
#include <iostream>
#include <unordered_map>

Mesh loadOBJ(const std::string& path) {
    std::vector<glm::vec3> tempPositions;
    std::vector<glm::vec2> tempTexCoords;
    std::vector<glm::vec3> tempNormals;

    std::vector<Vertex> vertices;
    std::vector<unsigned int> indices;

    // Map to detect and reuse duplicate vertices
    // Key: "posIndex/texIndex/normIndex" string
    // Value: index in the final vertex array
    std::unordered_map<std::string, unsigned int> vertexMap;

    std::ifstream file(path);
    if (!file.is_open()) {
        std::cerr << "ERROR: Could not open OBJ file: " << path << std::endl;
        return Mesh(vertices, indices);
    }

    std::string line;
    while (std::getline(file, line)) {
        std::istringstream iss(line);
        std::string prefix;
        iss >> prefix;

        if (prefix == "v") {
            glm::vec3 pos;
            iss >> pos.x >> pos.y >> pos.z;
            tempPositions.push_back(pos);
        }
        else if (prefix == "vt") {
            glm::vec2 tex;
            iss >> tex.x >> tex.y;
            tempTexCoords.push_back(tex);
        }
        else if (prefix == "vn") {
            glm::vec3 norm;
            iss >> norm.x >> norm.y >> norm.z;
            tempNormals.push_back(norm);
        }
        else if (prefix == "f") {
            // Parse face — can have 3 or more vertices (we handle triangles and quads)
            std::vector<std::string> faceTokens;
            std::string token;
            while (iss >> token) {
                faceTokens.push_back(token);
            }

            // Triangulate: fan from first vertex (works for convex polygons)
            for (size_t i = 1; i + 1 < faceTokens.size(); i++) {
                std::string triTokens[3] = {
                    faceTokens[0], faceTokens[i], faceTokens[i + 1]
                };

                for (const auto& tok : triTokens) {
                    // Check if we've already created this exact vertex
                    auto it = vertexMap.find(tok);
                    if (it != vertexMap.end()) {
                        indices.push_back(it->second);
                        continue;
                    }

                    // Parse "posIndex/texIndex/normIndex"
                    std::istringstream tokenStream(tok);
                    std::string part;
                    int posIdx = 0, texIdx = 0, normIdx = 0;

                    // Position index (required)
                    std::getline(tokenStream, part, '/');
                    posIdx = std::stoi(part) - 1;  // OBJ is 1-based

                    // Texture coordinate index (optional)
                    if (std::getline(tokenStream, part, '/') && !part.empty()) {
                        texIdx = std::stoi(part) - 1;
                    }

                    // Normal index (optional)
                    if (std::getline(tokenStream, part, '/') && !part.empty()) {
                        normIdx = std::stoi(part) - 1;
                    }

                    Vertex vertex{};
                    vertex.position = tempPositions[posIdx];

                    if (!tempTexCoords.empty() && texIdx >= 0) {
                        vertex.texCoords = tempTexCoords[texIdx];
                    }

                    if (!tempNormals.empty() && normIdx >= 0) {
                        vertex.normal = tempNormals[normIdx];
                    }

                    unsigned int newIndex = static_cast<unsigned int>(vertices.size());
                    vertices.push_back(vertex);
                    indices.push_back(newIndex);
                    vertexMap[tok] = newIndex;
                }
            }
        }
        // Ignore other lines (comments #, materials, etc.)
    }

    std::cout << "Loaded OBJ: " << path
              << " (" << vertices.size() << " vertices, "
              << indices.size() / 3 << " triangles)" << std::endl;

    return Mesh(vertices, indices);
}
```

### C++ Concept: `std::unordered_map`

```cpp
std::unordered_map<std::string, unsigned int> vertexMap;
```

A hash map (dictionary). Key lookups are O(1) average time. We use it to detect duplicate vertices:

```cpp
vertexMap["1/1/1"] = 0;       // "vertex combo 1/1/1 is at index 0"
auto it = vertexMap.find("1/1/1");
if (it != vertexMap.end()) {
    // Found! Reuse index: it->second
}
```

### C++ Concept: `std::istringstream`

```cpp
std::istringstream iss(line);
std::string prefix;
iss >> prefix;
```

An `istringstream` wraps a string and lets you read from it like a file using `>>`. The `>>` operator skips whitespace and reads the next token. This is the idiomatic way to parse simple text formats in C++.

---

## Updating the Shader for Normals

Since our vertex data now includes normals, update the textured shader:

### assets/shaders/textured.vert (updated)

```glsl
#version 460 core

layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aNormal;     // NEW
layout (location = 2) in vec2 aTexCoord;   // Moved to location 2

out vec2 TexCoord;
out vec3 Normal;       // Pass to fragment shader for lighting (Chapter 7)
out vec3 FragPos;      // World-space position for lighting

uniform mat4 model;
uniform mat4 view;
uniform mat4 projection;

void main() {
    FragPos = vec3(model * vec4(aPos, 1.0));
    Normal = mat3(transpose(inverse(model))) * aNormal;  // Normal matrix
    TexCoord = aTexCoord;
    gl_Position = projection * view * vec4(FragPos, 1.0);
}
```

The `Normal` line looks complex. The **normal matrix** (`mat3(transpose(inverse(model)))`) ensures normals stay perpendicular to the surface even when the object is scaled non-uniformly. For now, just know it's the correct way to transform normals — we'll explain it fully in Chapter 7.

---

## Procedural Meshes — Building Geometry in Code

Not everything comes from files. Floors, walls, and debug shapes can be built in code:

### A Box (Cube) Generator

```cpp
Mesh createBox(float width, float height, float depth) {
    float w = width / 2.0f, h = height / 2.0f, d = depth / 2.0f;

    std::vector<Vertex> vertices = {
        // Front face (normal: 0, 0, 1)
        {{-w, -h,  d}, {0, 0, 1}, {0, 0}},
        {{ w, -h,  d}, {0, 0, 1}, {1, 0}},
        {{ w,  h,  d}, {0, 0, 1}, {1, 1}},
        {{-w,  h,  d}, {0, 0, 1}, {0, 1}},
        // Back face (normal: 0, 0, -1)
        {{ w, -h, -d}, {0, 0, -1}, {0, 0}},
        {{-w, -h, -d}, {0, 0, -1}, {1, 0}},
        {{-w,  h, -d}, {0, 0, -1}, {1, 1}},
        {{ w,  h, -d}, {0, 0, -1}, {0, 1}},
        // Top face (normal: 0, 1, 0)
        {{-w,  h,  d}, {0, 1, 0}, {0, 0}},
        {{ w,  h,  d}, {0, 1, 0}, {1, 0}},
        {{ w,  h, -d}, {0, 1, 0}, {1, 1}},
        {{-w,  h, -d}, {0, 1, 0}, {0, 1}},
        // Bottom face (normal: 0, -1, 0)
        {{-w, -h, -d}, {0, -1, 0}, {0, 0}},
        {{ w, -h, -d}, {0, -1, 0}, {1, 0}},
        {{ w, -h,  d}, {0, -1, 0}, {1, 1}},
        {{-w, -h,  d}, {0, -1, 0}, {0, 1}},
        // Right face (normal: 1, 0, 0)
        {{ w, -h,  d}, {1, 0, 0}, {0, 0}},
        {{ w, -h, -d}, {1, 0, 0}, {1, 0}},
        {{ w,  h, -d}, {1, 0, 0}, {1, 1}},
        {{ w,  h,  d}, {1, 0, 0}, {0, 1}},
        // Left face (normal: -1, 0, 0)
        {{-w, -h, -d}, {-1, 0, 0}, {0, 0}},
        {{-w, -h,  d}, {-1, 0, 0}, {1, 0}},
        {{-w,  h,  d}, {-1, 0, 0}, {1, 1}},
        {{-w,  h, -d}, {-1, 0, 0}, {0, 1}},
    };

    std::vector<unsigned int> indices;
    for (unsigned int face = 0; face < 6; face++) {
        unsigned int base = face * 4;
        indices.push_back(base + 0);
        indices.push_back(base + 1);
        indices.push_back(base + 2);
        indices.push_back(base + 0);
        indices.push_back(base + 2);
        indices.push_back(base + 3);
    }

    return Mesh(vertices, indices);
}
```

24 vertices (4 per face — we can't share corners because normals differ per face) and 36 indices (6 per face, 2 triangles each).

---

## Resource Management Basics

Multiple entities might use the same mesh (e.g. 50 crates all using the same cube mesh). We shouldn't load it 50 times. A simple approach:

```cpp
// Store loaded meshes by name
std::unordered_map<std::string, std::shared_ptr<Mesh>> meshCache;

std::shared_ptr<Mesh> getMesh(const std::string& name) {
    auto it = meshCache.find(name);
    if (it != meshCache.end()) {
        return it->second;
    }
    return nullptr;  // Not loaded
}

void storeMesh(const std::string& name, std::shared_ptr<Mesh> mesh) {
    meshCache[name] = mesh;
}
```

### C++ Concept: `std::shared_ptr`

```cpp
std::shared_ptr<Mesh> mesh = std::make_shared<Mesh>(vertices, indices);
```

A **shared pointer** is a smart pointer with reference counting. Multiple `shared_ptr`s can point to the same object. The object is destroyed when the last `shared_ptr` to it is destroyed.

This is perfect for resources: 50 entities can share the same mesh, and it's automatically cleaned up when no one references it anymore.

```cpp
auto mesh = std::make_shared<Mesh>(loadOBJ("cube.obj"));  // ref count: 1
auto copy = mesh;    // ref count: 2
copy = nullptr;      // ref count: 1
mesh = nullptr;      // ref count: 0 → Mesh is destroyed
```

For now we'll keep resource management simple. A full resource manager is a later concern.

---

## Using It All Together

In `main.cpp`, you can now load a model and create an entity:

```cpp
    // Load a mesh from file
    Mesh crateMesh = loadOBJ("assets/models/crate.obj");

    // Or generate one procedurally
    Mesh boxMesh = createBox(1.0f, 1.0f, 1.0f);

    // Create entities using the mesh
    auto crate = registry.create();
    registry.emplace<Position>(crate, glm::vec3(0.0f, 0.0f, -3.0f));
    registry.emplace<MeshRenderer>(crate, boxMesh.getVAO(),
                                    0u,  // vertexCount (unused with indices)
                                    texturedShader.getId(),
                                    crateTexture.getId(),
                                    true,  // useIndices
                                    boxMesh.getIndexCount());
```

---

## Update CMakeLists.txt

```cmake
add_executable(QEngine
    src/main.cpp
    src/engine/core/window.cpp
    src/engine/renderer/shader.cpp
    src/engine/renderer/camera.cpp
    src/engine/renderer/texture.cpp
    src/engine/renderer/mesh.cpp
    src/engine/renderer/obj_loader.cpp
    src/engine/renderer/stb_image_impl.cpp
    src/engine/ecs/systems/render_system.cpp
    src/engine/ecs/systems/movement_system.cpp
)
```

---

## What's Next

In **Chapter 7**, we'll add lighting — directional lights, point lights, and the Phong lighting model. This is what transforms flat-textured surfaces into a world with depth, shadow, and atmosphere. The normals we've been carefully passing through will finally be put to use.
