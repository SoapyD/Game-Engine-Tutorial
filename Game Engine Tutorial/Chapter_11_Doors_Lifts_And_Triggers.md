# Chapter 11: Doors, Lifts & Triggers

## What You'll Learn
- State machines — how interactive objects track what they're doing
- Trigger volumes — detecting when a player enters an area
- Doors that open and close
- Lifts (elevators) that carry the player
- Timed events — delays, auto-close, sequences
- How Quake defines these entities

---

## How Quake Did It

In Quake, every interactive level element is a **brush entity** — a piece of level geometry with special behaviour defined by key-value properties. A door might look like:

```
{
  "classname" "func_door"
  "angle" "90"
  "speed" "100"
  "wait" "3"
  "lip" "8"
}
```

The engine sees `func_door` and knows: this geometry should move, in a direction, at a speed, wait some seconds, then return. No scripting language needed — just data.

We'll build the same approach using ECS components.

---

## State Machines

Interactive objects exist in a state. A door is either closed, opening, open, or closing. A state machine manages these transitions:

```
        ┌─── triggered ───┐
        ▼                  │
    ┌────────┐         ┌────────┐
    │ CLOSED │────────▶│OPENING │
    └────────┘         └───┬────┘
        ▲                  │
        │           reached end
        │                  ▼
    ┌────────┐         ┌────────┐
    │CLOSING │◀────────│  OPEN  │
    └────────┘  timer  └────────┘
                expired
```

### The State Component

```cpp
// Add to components.h

enum class MoverState {
    Idle,       // At start position
    Moving,     // Moving to end position
    Waiting,    // At end position, waiting before returning
    Returning   // Moving back to start position
};

struct Mover {
    glm::vec3 startPos;          // Where it starts (closed position)
    glm::vec3 endPos;            // Where it ends (open position)
    float speed = 2.0f;          // Units per second
    float waitTime = 3.0f;       // Seconds to stay open
    float timer = 0.0f;          // Current timer
    float progress = 0.0f;       // 0.0 = start, 1.0 = end
    MoverState state = MoverState::Idle;
    bool requiresTrigger = true; // Must be triggered to start
};
```

### C++ Concept: `enum class` for State Machines

`enum class` is ideal for state machines:

```cpp
enum class MoverState { Idle, Moving, Waiting, Returning };

MoverState state = MoverState::Idle;

switch (state) {
    case MoverState::Idle:      /* ... */ break;
    case MoverState::Moving:    /* ... */ break;
    case MoverState::Waiting:   /* ... */ break;
    case MoverState::Returning: /* ... */ break;
}
```

The compiler can warn you if your `switch` doesn't handle all cases. With raw integers or old-style enums, typos and missing cases are silent bugs.

---

## Trigger Volumes

A trigger is an invisible box in the world. When the player enters it, something happens. Add to `components.h`:

```cpp
enum class TriggerAction {
    ActivateMover,     // Open a door, start a lift
    Teleport,          // Move the player somewhere
    Damage,            // Hurt the player (lava, spikes)
    Heal,              // Heal zone
    ChangeLevel,       // Load next level
    Message            // Display text to the player
};

struct TriggerVolume {
    TriggerAction action = TriggerAction::ActivateMover;
    entt::entity target = entt::null;  // Entity to activate (for ActivateMover)
    glm::vec3 destination;              // For teleport
    float value = 0.0f;                 // Damage/heal amount
    std::string message;                // For Message action
    bool onlyOnce = false;              // Fire once then disable
    bool triggered = false;             // Has been triggered (for onlyOnce)
    float cooldown = 0.0f;             // Minimum time between triggers
    float cooldownTimer = 0.0f;
};
```

A trigger entity has `Position`, `AABBCollider` (with `isTrigger = true`), and `TriggerVolume`.

---

## The Trigger System

### src/engine/ecs/systems/trigger_system.h

```cpp
#pragma once

#include <entt/entt.hpp>

void triggerSystem(entt::registry& registry, float dt);
```

### src/engine/ecs/systems/trigger_system.cpp

```cpp
#include "engine/ecs/systems/trigger_system.h"
#include "engine/ecs/components.h"
#include "engine/physics/aabb.h"

void triggerSystem(entt::registry& registry, float dt) {
    auto triggerView = registry.view<Position, AABBCollider, TriggerVolume>();

    for (auto [trigEntity, trigPos, trigCol, trigger] : triggerView.each()) {
        // Skip disabled triggers
        if (trigger.onlyOnce && trigger.triggered) continue;

        // Cooldown
        if (trigger.cooldownTimer > 0.0f) {
            trigger.cooldownTimer -= dt;
            continue;
        }

        AABB triggerBox = AABB::fromCenterSize(trigPos.value, trigCol.halfExtents);

        // Check against all entities with position and collider
        auto entityView = registry.view<Position, AABBCollider>();

        for (auto [entity, entPos, entCol] : entityView.each()) {
            if (entity == trigEntity) continue;
            if (entCol.isTrigger) continue;  // Triggers don't trigger other triggers

            AABB entityBox = AABB::fromCenterSize(entPos.value, entCol.halfExtents);

            if (!triggerBox.intersects(entityBox)) continue;

            // ─── Trigger activated! ──────────────────────────────
            switch (trigger.action) {
                case TriggerAction::ActivateMover: {
                    if (trigger.target != entt::null &&
                        registry.valid(trigger.target) &&
                        registry.all_of<Mover>(trigger.target)) {
                        auto& mover = registry.get<Mover>(trigger.target);
                        if (mover.state == MoverState::Idle) {
                            mover.state = MoverState::Moving;
                        }
                    }
                    break;
                }

                case TriggerAction::Teleport: {
                    entPos.value = trigger.destination;
                    // Reset velocity to prevent flying out of the teleporter
                    if (registry.all_of<Velocity>(entity)) {
                        registry.get<Velocity>(entity).value = glm::vec3(0.0f);
                    }
                    break;
                }

                case TriggerAction::Damage: {
                    if (registry.all_of<Health>(entity)) {
                        registry.get<Health>(entity).current -= trigger.value * dt;
                    }
                    break;
                }

                case TriggerAction::Heal: {
                    if (registry.all_of<Health>(entity)) {
                        auto& health = registry.get<Health>(entity);
                        health.current = std::min(
                            health.current + trigger.value * dt, health.max);
                    }
                    break;
                }

                case TriggerAction::ChangeLevel: {
                    // Set a flag or event — the game layer handles level loading
                    // We won't implement this here yet
                    break;
                }

                case TriggerAction::Message: {
                    // TODO: Display trigger.message on HUD (Chapter 15)
                    break;
                }
            }

            // Mark as triggered and start cooldown
            trigger.triggered = true;
            trigger.cooldownTimer = trigger.cooldown;

            if (trigger.onlyOnce) break;
        }
    }
}
```

---

## The Mover System

This system handles the actual movement of doors, lifts, and any other moving geometry.

### src/engine/ecs/systems/mover_system.h

```cpp
#pragma once

#include <entt/entt.hpp>

void moverSystem(entt::registry& registry, float dt);
```

### src/engine/ecs/systems/mover_system.cpp

```cpp
#include "engine/ecs/systems/mover_system.h"
#include "engine/ecs/components.h"

void moverSystem(entt::registry& registry, float dt) {
    auto view = registry.view<Position, Mover>();

    for (auto [entity, pos, mover] : view.each()) {
        switch (mover.state) {

            case MoverState::Idle:
                // Sitting at start position, waiting to be triggered
                break;

            case MoverState::Moving: {
                // Move toward end position
                float distance = glm::length(mover.endPos - mover.startPos);
                float step = (mover.speed / distance) * dt;
                mover.progress += step;

                if (mover.progress >= 1.0f) {
                    mover.progress = 1.0f;
                    mover.state = MoverState::Waiting;
                    mover.timer = mover.waitTime;
                }

                // Interpolate position
                pos.value = glm::mix(mover.startPos, mover.endPos, mover.progress);
                break;
            }

            case MoverState::Waiting: {
                // At end position, counting down
                mover.timer -= dt;

                if (mover.timer <= 0.0f) {
                    mover.state = MoverState::Returning;
                }
                break;
            }

            case MoverState::Returning: {
                // Move back to start
                float distance = glm::length(mover.endPos - mover.startPos);
                float step = (mover.speed / distance) * dt;
                mover.progress -= step;

                if (mover.progress <= 0.0f) {
                    mover.progress = 0.0f;
                    mover.state = MoverState::Idle;
                }

                pos.value = glm::mix(mover.startPos, mover.endPos, mover.progress);
                break;
            }
        }
    }
}
```

### `glm::mix` — Linear Interpolation

```cpp
glm::mix(a, b, t)  // Returns a + (b - a) * t
```

When `t = 0.0`, returns `a`. When `t = 1.0`, returns `b`. Values between smoothly blend. This is also called **lerp** (linear interpolation) — one of the most fundamental operations in game development.

---

## Building a Door

A door is an entity with:
- `Position` (its current world position)
- `MeshRenderer` (so it's visible)
- `Mover` (defines start/end positions and behaviour)
- `AABBCollider` (so the player can't walk through it when closed)

Plus a trigger in front of the door:

```cpp
// The door itself
auto door = registry.create();
glm::vec3 closedPos(5.0f, 0.0f, 0.0f);
glm::vec3 openPos(5.0f, 4.0f, 0.0f);  // Slides upward (like Quake doors)

registry.emplace<Position>(door, closedPos);
registry.emplace<MeshRenderer>(door, doorMesh.getVAO(), 0u,
                                litShader.getId(), doorTexture.getId(),
                                true, doorMesh.getIndexCount());
registry.emplace<Mover>(door, closedPos, openPos, 3.0f, 4.0f, 0.0f, 0.0f,
                          MoverState::Idle, true);
registry.emplace<AABBCollider>(door, glm::vec3(0.1f, 2.0f, 1.5f), false);

// Trigger zone in front of the door
auto doorTrigger = registry.create();
registry.emplace<Position>(doorTrigger, glm::vec3(4.0f, 1.0f, 0.0f));
registry.emplace<AABBCollider>(doorTrigger, glm::vec3(1.5f, 1.5f, 2.0f), true);
registry.emplace<TriggerVolume>(doorTrigger,
    TriggerAction::ActivateMover,
    door,                      // target: the door entity
    glm::vec3(0.0f),           // destination (unused for mover)
    0.0f,                      // value (unused)
    "",                        // message (unused)
    false,                     // not once-only
    false,                     // not yet triggered
    1.0f,                      // 1 second cooldown between triggers
    0.0f                       // cooldown timer starts at 0
);
```

When the player walks into the trigger zone, the door slides up. After 4 seconds, it slides back down.

---

## Building a Lift

A lift is a floor surface that carries the player up or down. Same Mover component, different axis:

```cpp
auto lift = registry.create();
glm::vec3 bottomPos(0.0f, 0.0f, -8.0f);
glm::vec3 topPos(0.0f, 6.0f, -8.0f);

registry.emplace<Position>(lift, bottomPos);
registry.emplace<MeshRenderer>(lift, platformMesh.getVAO(), 0u,
                                litShader.getId(), metalTexture.getId(),
                                true, platformMesh.getIndexCount());
registry.emplace<Mover>(lift, bottomPos, topPos, 2.0f, 2.0f, 0.0f, 0.0f,
                          MoverState::Idle, true);
registry.emplace<AABBCollider>(lift, glm::vec3(1.5f, 0.1f, 1.5f), false);

// Trigger on the lift platform itself (step on it to activate)
auto liftTrigger = registry.create();
registry.emplace<Position>(liftTrigger, glm::vec3(0.0f, 0.3f, -8.0f));
registry.emplace<AABBCollider>(liftTrigger, glm::vec3(1.5f, 0.3f, 1.5f), true);
registry.emplace<TriggerVolume>(liftTrigger,
    TriggerAction::ActivateMover, lift,
    glm::vec3(0.0f), 0.0f, "", false, false, 0.5f, 0.0f);
```

### Carrying the Player

When a lift moves, the player standing on it needs to move too. There are two approaches:

**Approach A: Parent the player to the lift** — track which entity the player is standing on and add the lift's movement delta to the player each frame.

**Approach B: Let physics handle it** — the lift's collider moves upward, pushing the player up via collision response.

Approach B is simpler and works with our existing collision system. The player stands on the lift's AABB; when the lift moves up, the collision system prevents the player from falling through, effectively carrying them.

---

## A Teleporter

```cpp
auto teleportTrigger = registry.create();
registry.emplace<Position>(teleportTrigger, glm::vec3(8.0f, 0.5f, 3.0f));
registry.emplace<AABBCollider>(teleportTrigger, glm::vec3(1.0f, 1.5f, 1.0f), true);
registry.emplace<TriggerVolume>(teleportTrigger,
    TriggerAction::Teleport,
    entt::null,                         // no target entity
    glm::vec3(-8.0f, 1.0f, -3.0f),    // destination
    0.0f, "", false, false, 1.0f, 0.0f);
```

Walk into it and you're instantly at the destination.

---

## A Lava Pool (Damage Zone)

```cpp
auto lava = registry.create();
registry.emplace<Position>(lava, glm::vec3(0.0f, -0.5f, 10.0f));
registry.emplace<AABBCollider>(lava, glm::vec3(5.0f, 0.5f, 5.0f), true);
registry.emplace<TriggerVolume>(lava,
    TriggerAction::Damage,
    entt::null,
    glm::vec3(0.0f),
    25.0f,      // 25 damage per second
    "", false, false, 0.0f, 0.0f);  // No cooldown — continuous damage
```

---

## Lambda Functions (Preview)

As your trigger system grows, you might want arbitrary behaviour:

```cpp
// C++ Concept: Lambda functions
auto myAction = [](entt::registry& reg, entt::entity triggerer) {
    // Do whatever you want
    if (reg.all_of<Health>(triggerer)) {
        reg.get<Health>(triggerer).current -= 10.0f;
    }
};

// Lambdas are anonymous functions you can store and call later
myAction(registry, playerEntity);
```

### C++ Concept: `std::function`

To store a lambda in a component:

```cpp
#include <functional>

struct CustomTrigger {
    std::function<void(entt::registry&, entt::entity)> callback;
};
```

`std::function` is a general-purpose wrapper that can hold any callable: lambdas, function pointers, functors. It has some overhead (heap allocation for large lambdas), but it's the easiest way to store arbitrary behaviour.

We won't use this approach heavily — it goes against the "data-only components" philosophy of ECS. But for special one-off level scripting, it's pragmatic.

---

## Updated Tick Order

```
1.  InputSystem
2.  CharacterMovementSystem
3.  HandleJump
4.  PhysicsSystem
5.  MoverSystem             ← NEW: update doors, lifts
6.  CollisionSystem
7.  MovementSystem
8.  GroundDetectionSystem
9.  TriggerSystem           ← NEW: detect trigger overlaps
10. DeathSystem
11. RenderSystem
```

The mover system runs before collision so that moving platforms push the player correctly. The trigger system runs after movement so it detects the player's final position.

---

## What's Next

In **Chapter 12**, we'll add weapons and projectiles — hitscan guns, rockets, damage dealing, and the foundation of FPS combat.
