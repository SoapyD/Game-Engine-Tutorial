# QEngine — Architecture Reference

## Project Structure

```
QEngine/
├── CMakeLists.txt
├── extern/                     # Third-party libraries
│   ├── entt/
│   ├── glfw/
│   ├── glad/
│   ├── glm/
│   ├── stb/
│   └── miniaudio/
├── assets/                     # Game assets
│   ├── shaders/
│   ├── textures/
│   ├── models/
│   ├── levels/
│   └── sounds/
├── src/
│   ├── main.cpp                # Entry point
│   ├── engine/
│   │   ├── core/
│   │   │   ├── engine.h/.cpp       # Engine initialisation and main loop
│   │   │   ├── window.h/.cpp       # GLFW window wrapper
│   │   │   └── timer.h/.cpp        # Delta time, fixed timestep
│   │   ├── ecs/
│   │   │   ├── components.h        # All component definitions
│   │   │   └── systems/
│   │   │       ├── render_system.h/.cpp
│   │   │       ├── movement_system.h/.cpp
│   │   │       ├── input_system.h/.cpp
│   │   │       ├── collision_system.h/.cpp
│   │   │       ├── physics_system.h/.cpp
│   │   │       ├── combat_system.h/.cpp
│   │   │       ├── ai_system.h/.cpp
│   │   │       ├── audio_system.h/.cpp
│   │   │       ├── trigger_system.h/.cpp
│   │   │       └── network_system.h/.cpp
│   │   ├── renderer/
│   │   │   ├── shader.h/.cpp        # Shader loading & compilation
│   │   │   ├── mesh.h/.cpp          # Vertex data, VAO/VBO
│   │   │   ├── texture.h/.cpp       # Texture loading
│   │   │   ├── camera.h/.cpp        # View/projection math
│   │   │   └── light.h/.cpp         # Light data structures
│   │   ├── physics/
│   │   │   ├── aabb.h/.cpp          # Axis-aligned bounding boxes
│   │   │   ├── raycast.h/.cpp       # Ray-world intersection
│   │   │   └── collision.h/.cpp     # Collision detection & response
│   │   ├── audio/
│   │   │   └── audio_manager.h/.cpp # miniaudio wrapper
│   │   ├── network/
│   │   │   ├── server.h/.cpp        # Server-side networking
│   │   │   ├── client.h/.cpp        # Client-side networking
│   │   │   └── protocol.h           # Packet definitions
│   │   └── level/
│   │       ├── bsp.h/.cpp           # BSP tree loading & traversal
│   │       └── level_loader.h/.cpp  # Level file parsing
│   └── game/
│       ├── game.h/.cpp              # Game-specific setup (spawn entities, etc.)
│       ├── weapons.h/.cpp           # Weapon definitions
│       └── items.h/.cpp             # Item definitions
└── tests/                      # Optional: unit tests
```

---

## Component Registry

All components are plain data structs. No methods, no inheritance.

### Core Components

```cpp
// Identity & metadata
struct TagPlayer {};                    // Marks entity as the local player
struct TagEnemy {};                      // Marks entity as an enemy
struct TagProjectile {};                // Marks entity as a projectile
struct Name { std::string value; };     // Debug name

// Spatial
struct Position { glm::vec3 value; };
struct Rotation { glm::vec3 euler; };   // Pitch, yaw, roll
struct Scale { glm::vec3 value = glm::vec3(1.0f); };
struct Velocity { glm::vec3 value; };
struct Gravity { float strength = 9.81f; };
struct OnGround { bool value = false; };

// Rendering
struct MeshRenderer {
    unsigned int vao;
    unsigned int indexCount;
    unsigned int textureId;
    unsigned int shaderId;
};
struct Color { glm::vec4 value; };

// Collision
struct AABBCollider {
    glm::vec3 halfExtents;
    bool isTrigger = false;
};

// Health & Combat
struct Health { float current; float max; };
struct Damage { float amount; };
struct Lifetime { float remaining; };   // Auto-destroy after time

// Input
struct PlayerInput {
    glm::vec2 moveDir;
    glm::vec2 lookDelta;
    bool jump = false;
    bool fire = false;
};

// AI
struct AIChase { entt::entity target; float speed; };
struct AIPatrol {
    std::vector<glm::vec3> waypoints;
    int currentWaypoint = 0;
    float speed;
};

// Weapons
struct Weapon {
    float fireCooldown;
    float cooldownRemaining = 0.0f;
    float damage;
    float range;
    bool isHitscan;
};

// Audio
struct AudioSource {
    std::string soundFile;
    bool loop = false;
    float volume = 1.0f;
};

// Triggers
struct TriggerVolume {
    enum class Action { OpenDoor, Teleport, Damage, Heal };
    Action action;
    float value;
};

// Networking
struct NetworkId { uint32_t id; };
struct NetworkOwner { uint32_t clientId; };
```

---

## System Execution Order

Systems run in a fixed order each frame. This is the planned tick order:

```
1.  InputSystem          — Read keyboard/mouse, populate PlayerInput components
2.  AISystem             — AI entities decide their intentions
3.  PhysicsSystem        — Apply gravity, friction, velocity integration
4.  MovementSystem       — Apply velocity to position
5.  CollisionSystem      — Detect and resolve collisions
6.  TriggerSystem        — Check trigger volume overlaps
7.  CombatSystem         — Process weapon firing, damage application
8.  LifetimeSystem       — Count down Lifetime components, destroy expired
9.  DeathSystem          — Destroy entities with Health <= 0
10. AudioSystem          — Play/stop sounds based on component state
11. NetworkSendSystem    — Serialize and send state (if multiplayer)
12. NetworkReceiveSystem — Receive and apply remote state (if multiplayer)
13. RenderSystem         — Draw everything
```

---

## Key Design Decisions

### Fixed Timestep for Physics

Physics and game logic run at a fixed rate (e.g. 60 ticks/sec) independent of frame rate. Rendering can run at any frame rate with interpolation.

```
accumulator += frameDeltaTime;
while (accumulator >= FIXED_TIMESTEP) {
    RunGameSystems(FIXED_TIMESTEP);
    accumulator -= FIXED_TIMESTEP;
}
RenderSystem(interpolationAlpha);
```

### Engine vs Game Separation

- `src/engine/` contains reusable engine code (renderer, physics, ECS systems)
- `src/game/` contains game-specific logic (weapon stats, item definitions, level scripts)
- The engine doesn't know about game-specific concepts

### Resource Management

Resources (textures, meshes, shaders, sounds) are loaded once and referenced by ID. Components store handles, not the data itself.

### Networking Model

Client-server architecture (like Quake):
- Server is authoritative — it runs the real game state
- Clients send input commands to the server
- Server processes inputs, sends snapshots back
- Clients interpolate between snapshots and predict locally
