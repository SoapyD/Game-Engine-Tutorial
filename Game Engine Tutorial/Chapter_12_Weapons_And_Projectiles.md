# Chapter 12: Weapons & Projectiles

## What You'll Learn
- Hitscan weapons — instant-hit guns like shotguns and railguns
- Projectile weapons — rockets, grenades, and other moving objects
- Weapon switching and cooldowns
- Damage dealing through the ECS
- Muzzle flash and impact effects (placeholder for Chapter 20)
- How Quake handled its weapon arsenal

---

## Two Types of Weapon

FPS games use two fundamentally different weapon types:

### Hitscan
The shot is instant. When you click, a ray is cast from the camera and whatever it hits takes damage immediately. No travel time.

- Quake: Shotgun, Super Shotgun, Nailgun (debatable), Lightning Gun
- Used for: fast, reliable weapons

### Projectile
A physical entity is spawned and travels through the world. It has a position, velocity, and collider. When it hits something, it deals damage and is destroyed.

- Quake: Rocket Launcher, Grenade Launcher
- Used for: area-denial, splash damage, skill shots

---

## Weapon Components

Add to `components.h`:

```cpp
enum class WeaponType {
    Shotgun,
    SuperShotgun,
    Nailgun,
    RocketLauncher,
    GrenadeLauncher,
    LightningGun,
    Railgun
};

enum class FireMode {
    Hitscan,
    Projectile
};

struct Weapon {
    WeaponType type;
    FireMode fireMode;
    float damage = 10.0f;
    float fireRate = 0.5f;          // Seconds between shots
    float cooldownRemaining = 0.0f;
    float range = 1000.0f;          // Hitscan max range
    float spread = 0.0f;            // Cone of inaccuracy (radians)
    int pelletCount = 1;            // Shotguns fire multiple pellets
    float projectileSpeed = 0.0f;   // For projectile weapons
    float splashRadius = 0.0f;      // Area of effect damage radius
    float splashDamage = 0.0f;      // Damage at center of splash
    int ammoPerShot = 1;
};

struct WeaponInventory {
    std::vector<Weapon> weapons;
    int currentWeapon = 0;
};

struct Ammo {
    int shells = 0;
    int nails = 0;
    int rockets = 0;
    int cells = 0;
};

// Attached to projectile entities
struct Projectile {
    float damage;
    float splashRadius;
    float splashDamage;
    entt::entity owner = entt::null;  // Who fired it (for kill credit)
};
```

---

## Weapon Definitions

A factory function that creates weapon data. No classes — just filling in a struct:

```cpp
Weapon createWeapon(WeaponType type) {
    Weapon w{};
    w.type = type;

    switch (type) {
        case WeaponType::Shotgun:
            w.fireMode = FireMode::Hitscan;
            w.damage = 4.0f;         // Per pellet
            w.fireRate = 0.5f;
            w.range = 500.0f;
            w.spread = 0.05f;        // Small spread cone
            w.pelletCount = 6;
            w.ammoPerShot = 1;
            break;

        case WeaponType::SuperShotgun:
            w.fireMode = FireMode::Hitscan;
            w.damage = 4.0f;
            w.fireRate = 0.8f;
            w.range = 400.0f;
            w.spread = 0.08f;
            w.pelletCount = 14;
            w.ammoPerShot = 2;
            break;

        case WeaponType::Nailgun:
            w.fireMode = FireMode::Projectile;
            w.damage = 9.0f;
            w.fireRate = 0.1f;
            w.projectileSpeed = 30.0f;
            w.ammoPerShot = 1;
            break;

        case WeaponType::RocketLauncher:
            w.fireMode = FireMode::Projectile;
            w.damage = 100.0f;
            w.fireRate = 0.8f;
            w.projectileSpeed = 20.0f;
            w.splashRadius = 5.0f;
            w.splashDamage = 80.0f;
            w.ammoPerShot = 1;
            break;

        case WeaponType::GrenadeLauncher:
            w.fireMode = FireMode::Projectile;
            w.damage = 100.0f;
            w.fireRate = 0.6f;
            w.projectileSpeed = 15.0f;
            w.splashRadius = 5.0f;
            w.splashDamage = 80.0f;
            w.ammoPerShot = 1;
            break;

        case WeaponType::LightningGun:
            w.fireMode = FireMode::Hitscan;
            w.damage = 30.0f;        // Per second (continuous fire)
            w.fireRate = 0.05f;       // Very fast "ticks"
            w.range = 15.0f;          // Short range
            w.spread = 0.0f;
            w.pelletCount = 1;
            w.ammoPerShot = 1;
            break;

        case WeaponType::Railgun:
            w.fireMode = FireMode::Hitscan;
            w.damage = 80.0f;
            w.fireRate = 1.5f;
            w.range = 1000.0f;
            w.spread = 0.0f;
            w.pelletCount = 1;
            w.ammoPerShot = 1;
            break;
    }

    return w;
}
```

---

## The Combat System

### src/engine/ecs/systems/combat_system.h

```cpp
#pragma once

#include <entt/entt.hpp>
#include "engine/level/level.h"

void combatSystem(entt::registry& registry, const Level& level, float dt);
```

### src/engine/ecs/systems/combat_system.cpp

```cpp
#include "engine/ecs/systems/combat_system.h"
#include "engine/ecs/components.h"
#include "engine/physics/raycast.h"
#include "engine/physics/aabb.h"
#include <random>

// Random number generator for spread
static std::mt19937 rng(std::random_device{}());

// Apply spread to a direction vector
glm::vec3 applySpread(const glm::vec3& direction, float spread) {
    if (spread <= 0.0f) return direction;

    std::uniform_real_distribution<float> dist(-spread, spread);
    glm::vec3 spread_dir = direction;
    spread_dir.x += dist(rng);
    spread_dir.y += dist(rng);
    spread_dir.z += dist(rng);
    return glm::normalize(spread_dir);
}

// Find the closest entity hit by a ray
struct EntityHit {
    entt::entity entity;
    float distance;
    glm::vec3 point;
};

std::optional<EntityHit> raycastEntities(
    entt::registry& registry, const Ray& ray,
    entt::entity ignore, float maxRange)
{
    std::optional<EntityHit> closest;
    float closestDist = maxRange;

    auto view = registry.view<Position, AABBCollider>();
    for (auto [entity, pos, col] : view.each()) {
        if (entity == ignore) continue;
        if (col.isTrigger) continue;

        AABB box = AABB::fromCenterSize(pos.value, col.halfExtents);
        auto hit = rayIntersectsAABB(ray, box);

        if (hit.has_value() && hit.value() < closestDist) {
            closestDist = hit.value();
            closest = EntityHit{
                entity,
                hit.value(),
                ray.pointAt(hit.value())
            };
        }
    }

    return closest;
}

void fireHitscan(entt::registry& registry, const Level& level,
                  entt::entity shooter, const Weapon& weapon,
                  const glm::vec3& origin, const glm::vec3& direction) {

    for (int i = 0; i < weapon.pelletCount; i++) {
        glm::vec3 dir = applySpread(direction, weapon.spread);
        Ray ray{ origin, dir };

        // Check against entities
        auto entityHit = raycastEntities(registry, ray, shooter, weapon.range);

        // Check against level geometry
        float levelDist = weapon.range;
        // (Would trace ray against level surfaces here — similar to Chapter 9)

        if (entityHit.has_value() && entityHit->distance < levelDist) {
            // Hit an entity — apply damage
            if (registry.all_of<Health>(entityHit->entity)) {
                registry.get<Health>(entityHit->entity).current -= weapon.damage;
            }
            // TODO: spawn impact effect at entityHit->point (Chapter 20)
        } else {
            // Hit level geometry (or nothing)
            // TODO: spawn wall-hit decal/particles (Chapter 20)
        }
    }
}

void fireProjectile(entt::registry& registry, entt::entity shooter,
                     const Weapon& weapon,
                     const glm::vec3& origin, const glm::vec3& direction,
                     unsigned int projectileMeshVAO, unsigned int projectileShader) {

    auto projectile = registry.create();

    registry.emplace<Position>(projectile, origin + direction * 0.5f);
    registry.emplace<Velocity>(projectile, direction * weapon.projectileSpeed);
    registry.emplace<AABBCollider>(projectile,
        glm::vec3(0.15f, 0.15f, 0.15f), false);
    registry.emplace<Projectile>(projectile,
        weapon.damage, weapon.splashRadius, weapon.splashDamage, shooter);
    registry.emplace<Lifetime>(projectile, 10.0f);  // Despawn after 10 seconds

    // Visual representation
    registry.emplace<MeshRenderer>(projectile, projectileMeshVAO, 0u,
                                    projectileShader, 0u, true, 36u);
}

void combatSystem(entt::registry& registry, const Level& level, float dt) {
    // ─── Weapon cooldowns ────────────────────────────────────────
    auto weaponView = registry.view<WeaponInventory>();
    for (auto [entity, inv] : weaponView.each()) {
        for (auto& weapon : inv.weapons) {
            if (weapon.cooldownRemaining > 0.0f) {
                weapon.cooldownRemaining -= dt;
            }
        }
    }

    // ─── Handle fire input ───────────────────────────────────────
    auto shooterView = registry.view<Position, PlayerInput, WeaponInventory>();

    for (auto [entity, pos, input, inv] : shooterView.each()) {
        if (!input.fire) continue;
        if (inv.weapons.empty()) continue;

        Weapon& weapon = inv.weapons[inv.currentWeapon];
        if (weapon.cooldownRemaining > 0.0f) continue;

        // Check ammo
        // (simplified — you'd check the specific ammo type)

        // Get firing direction from camera
        // In a real implementation, this comes from the camera's front vector
        // For now, assume it's stored in a component or passed in
        glm::vec3 fireDir = glm::vec3(0.0f, 0.0f, -1.0f);  // Placeholder
        // TODO: get actual camera forward direction

        glm::vec3 fireOrigin = pos.value + glm::vec3(0.0f, 0.7f, 0.0f); // Eye height

        if (weapon.fireMode == FireMode::Hitscan) {
            fireHitscan(registry, level, entity, weapon, fireOrigin, fireDir);
        } else {
            // Projectile mesh/shader would be passed in or looked up from resources
            // fireProjectile(registry, entity, weapon, fireOrigin, fireDir,
            //                rocketMeshVAO, rocketShader);
        }

        weapon.cooldownRemaining = weapon.fireRate;

        // TODO: play fire sound (Chapter 16)
        // TODO: muzzle flash effect (Chapter 20)
    }

    // ─── Projectile collision ────────────────────────────────────
    auto projView = registry.view<Position, Velocity, AABBCollider, Projectile>();
    std::vector<entt::entity> toDestroy;

    for (auto [projEntity, pos, vel, col, proj] : projView.each()) {
        AABB projBox = AABB::fromCenterSize(pos.value, col.halfExtents);

        // Check against level geometry (simplified)
        bool hitLevel = false;
        for (const auto& sector : level.sectors) {
            if (pos.value.x < sector.boundsMin.x || pos.value.x > sector.boundsMax.x ||
                pos.value.y < sector.boundsMin.y || pos.value.y > sector.boundsMax.y ||
                pos.value.z < sector.boundsMin.z || pos.value.z > sector.boundsMax.z) {
                continue;
            }
            // A more precise check would test against individual surfaces
            // For now, hitting sector bounds counts as hitting a wall
        }

        // Check against entities
        auto entityView = registry.view<Position, AABBCollider, Health>();
        for (auto [target, tPos, tCol, health] : entityView.each()) {
            if (target == proj.owner) continue;  // Don't hit the shooter
            if (tCol.isTrigger) continue;

            AABB targetBox = AABB::fromCenterSize(tPos.value, tCol.halfExtents);
            if (projBox.intersects(targetBox)) {
                // Direct hit
                health.current -= proj.damage;

                // Splash damage — hurt nearby entities too
                if (proj.splashRadius > 0.0f) {
                    applySplashDamage(registry, pos.value,
                                      proj.splashRadius, proj.splashDamage,
                                      proj.owner);
                }

                toDestroy.push_back(projEntity);
                break;
            }
        }
    }

    for (auto e : toDestroy) {
        if (registry.valid(e)) {
            registry.destroy(e);
        }
    }
}
```

---

## Splash Damage

Rockets and grenades deal area damage. Everything within the splash radius takes damage, scaled by distance:

```cpp
void applySplashDamage(entt::registry& registry, const glm::vec3& center,
                        float radius, float maxDamage, entt::entity ignore) {

    auto view = registry.view<Position, Health>();

    for (auto [entity, pos, health] : view.each()) {
        if (entity == ignore) continue;

        float distance = glm::length(pos.value - center);
        if (distance > radius) continue;

        // Linear falloff: full damage at center, zero at edge
        float scale = 1.0f - (distance / radius);
        float damage = maxDamage * scale;
        health.current -= damage;

        // Knockback — push entity away from explosion
        if (registry.all_of<Velocity>(entity)) {
            glm::vec3 pushDir = glm::normalize(pos.value - center);
            float knockback = damage * 0.5f;  // Scale knockback with damage
            registry.get<Velocity>(entity).value += pushDir * knockback;
        }
    }
}
```

### Rocket Jumping

Notice the knockback code pushes entities away from explosions. If the player shoots a rocket at their own feet, the explosion pushes them upward. This is **rocket jumping** — an emergent mechanic from exactly this code. Quake didn't specifically code rocket jumping; it fell naturally out of the splash damage + knockback system.

To enable self-damage for rocket jumping:

```cpp
// Remove the `if (entity == ignore) continue;` check, or change it to:
if (entity == ignore) {
    // Self-damage is reduced
    health.current -= damage * 0.5f;
    // But still apply full knockback
}
```

---

## Weapon Switching

```cpp
void weaponSwitchSystem(entt::registry& registry) {
    auto view = registry.view<PlayerInput, WeaponInventory>();

    for (auto [entity, input, inv] : view.each()) {
        // Number keys 1-7 switch weapons
        // (input.weaponSwitch would be set by the input system)
        if (input.weaponSwitch >= 0 &&
            input.weaponSwitch < static_cast<int>(inv.weapons.size())) {
            inv.currentWeapon = input.weaponSwitch;
        }

        // Mouse wheel scroll
        // (input.scrollDelta would be set by the input system)
    }
}
```

Add to `PlayerInput`:

```cpp
struct PlayerInput {
    glm::vec2 moveDir;
    glm::vec2 lookDelta;
    bool jump = false;
    bool fire = false;
    int weaponSwitch = -1;    // -1 = no switch, 0-6 = weapon slot
};
```

---

## Lifetime System

Projectiles and temporary effects should auto-destroy:

```cpp
void lifetimeSystem(entt::registry& registry, float dt) {
    auto view = registry.view<Lifetime>();
    std::vector<entt::entity> expired;

    for (auto [entity, lifetime] : view.each()) {
        lifetime.remaining -= dt;
        if (lifetime.remaining <= 0.0f) {
            expired.push_back(entity);
        }
    }

    for (auto e : expired) {
        registry.destroy(e);
    }
}
```

---

## C++ Concept: `std::mt19937` — Random Numbers

```cpp
#include <random>

std::mt19937 rng(std::random_device{}());
std::uniform_real_distribution<float> dist(-0.1f, 0.1f);

float randomValue = dist(rng);  // Random float between -0.1 and 0.1
```

`std::mt19937` is the Mersenne Twister random number generator — fast, good quality randomness. `std::random_device{}()` seeds it from the OS's entropy source (hardware noise, etc.).

`std::uniform_real_distribution` produces evenly distributed floats in a range. There are also `std::normal_distribution` (bell curve), `std::uniform_int_distribution`, and others.

Never use `rand()` from C — it's poor quality, not thread-safe, and the modulo approach (`rand() % n`) isn't uniform.

---

## Setting Up the Player's Arsenal

```cpp
auto player = registry.create();
registry.emplace<Position>(player, glm::vec3(0.0f, 1.0f, 0.0f));
registry.emplace<Velocity>(player);
registry.emplace<Gravity>(player);
registry.emplace<OnGround>(player);
registry.emplace<CharacterPhysics>(player);
registry.emplace<AABBCollider>(player, glm::vec3(0.4f, 0.9f, 0.4f), false);
registry.emplace<Health>(player, 100.0f, 100.0f);
registry.emplace<PlayerInput>(player);
registry.emplace<TagPlayer>(player);

// Give the player weapons
WeaponInventory inv;
inv.weapons.push_back(createWeapon(WeaponType::Shotgun));
inv.weapons.push_back(createWeapon(WeaponType::RocketLauncher));
inv.currentWeapon = 0;
registry.emplace<WeaponInventory>(player, std::move(inv));

registry.emplace<Ammo>(player, 25, 0, 5, 0);  // 25 shells, 5 rockets
```

No weapon class hierarchy. No `WeaponBase` with virtual `fire()` methods. Just data and a system that reads it.

---

## Updated Tick Order

```
1.  InputSystem
2.  WeaponSwitchSystem      ← NEW
3.  CharacterMovementSystem
4.  HandleJump
5.  PhysicsSystem
6.  MoverSystem
7.  CollisionSystem
8.  MovementSystem
9.  GroundDetectionSystem
10. CombatSystem            ← NEW: handles firing and projectile collision
11. LifetimeSystem          ← NEW: destroys expired entities
12. TriggerSystem
13. DeathSystem
14. RenderSystem
```

---

## What's Next

In **Chapter 13**, we'll add items and pickups — health packs, ammo boxes, armour, and weapon pickups that respawn on timers. This completes the core item loop of an FPS.
