# Chapter 15: Gameplay Polish

## What You'll Learn
- Rendering a crosshair in the centre of the screen
- Expanding the debug HUD with ammo and weapon info
- Death detection and automatic respawn
- A damage flash overlay when the player takes a hit
- Storing a spawn point for respawning

---

## Step 1: PlayerState Component

We need to track some per-player gameplay state that doesn't fit neatly into the existing components: where the player spawns, damage flash timers, and death state.

### Add to `components.h`

```cpp
// Per-player gameplay state
struct PlayerState
{
    glm::vec3 spawnPoint = glm::vec3(0.0f);
    float damageFlashTimer = 0.0f;       // counts down after taking damage
    float damageFlashDuration = 0.3f;    // how long the flash lasts
    float previousHealth = 100.0f;       // tracks health to detect damage
    float respawnDelay = 1.5f;           // seconds to wait before respawning
    float respawnTimer = 0.0f;           // counts down when dead
    bool isDead = false;
};
```

### Attach it to the player in `scene_setup.cpp`

After the existing player setup, add:

```cpp
auto& playerState = registry.emplace<PlayerState>(player);
playerState.spawnPoint = glm::vec3(15.0f, 1.7f, 15.0f);
playerState.previousHealth = 100.0f;
```

---

## Step 2: The Crosshair

A crosshair is just a small `+` shape drawn at the centre of the screen. We'll render it using the same HUD shader and draw calls from Chapter 13.

### Add a crosshair draw function to `debug_hud_system.cpp`

Add this helper function above `debugHudSystem`:

```cpp
// ─── Draw a crosshair at screen centre ───────────────────────
static void drawCrosshair(int windowWidth, int windowHeight,
    unsigned int shaderId, const glm::mat4& projection)
{
    float cx = windowWidth * 0.5f;
    float cy = windowHeight * 0.5f;
    float size = 10.0f;     // half-length of each arm
    float thickness = 1.0f; // half-thickness of each arm

    // Two thin rectangles forming a + shape
    // Horizontal bar
    float hVerts[] = {
        cx - size, cy - thickness, 0.0f, 0.0f,
        cx + size, cy - thickness, 0.0f, 0.0f,
        cx + size, cy + thickness, 0.0f, 0.0f,
        cx - size, cy + thickness, 0.0f, 0.0f,
    };
    // Vertical bar
    float vVerts[] = {
        cx - thickness, cy - size, 0.0f, 0.0f,
        cx + thickness, cy - size, 0.0f, 0.0f,
        cx + thickness, cy + size, 0.0f, 0.0f,
        cx - thickness, cy + size, 0.0f, 0.0f,
    };

    unsigned int indices[] = { 0, 1, 2, 0, 2, 3 };

    glUseProgram(shaderId);
    GLint loc = glGetUniformLocation(shaderId, "projection");
    glUniformMatrix4fv(loc, 1, GL_FALSE, &projection[0][0]);

    // Identity model matrix (crosshair coords are already in screen space)
    glm::mat4 model(1.0f);
    loc = glGetUniformLocation(shaderId, "model");
    glUniformMatrix4fv(loc, 1, GL_FALSE, &model[0][0]);

    loc = glGetUniformLocation(shaderId, "textColor");
    glUniform3f(loc, 1.0f, 1.0f, 1.0f);  // white crosshair

    unsigned int vao, vbo, ebo;
    glGenVertexArrays(1, &vao);
    glGenBuffers(1, &vbo);
    glGenBuffers(1, &ebo);
    glBindVertexArray(vao);

    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ebo);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices),
        indices, GL_DYNAMIC_DRAW);

    glEnableVertexAttribArray(0);
    glVertexAttribPointer(0, 2, GL_FLOAT, GL_FALSE, 16, (void*)0);

    // Draw horizontal bar
    glBindBuffer(GL_ARRAY_BUFFER, vbo);
    glBufferData(GL_ARRAY_BUFFER, sizeof(hVerts), hVerts, GL_DYNAMIC_DRAW);
    glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);

    // Draw vertical bar
    glBufferData(GL_ARRAY_BUFFER, sizeof(vVerts), vVerts, GL_DYNAMIC_DRAW);
    glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);

    glBindVertexArray(0);
    glDeleteBuffers(1, &ebo);
    glDeleteBuffers(1, &vbo);
    glDeleteVertexArrays(1, &vao);
}
```

### Call it from `debugHudSystem`

At the end of the function, before re-enabling depth testing:

```cpp
// ─── Crosshair ───────────────────────────────────────────
drawCrosshair(windowWidth, windowHeight, shader, ortho);

// Re-enable depth testing for the next frame's 3D rendering
glEnable(GL_DEPTH_TEST);
```

---

## Step 3: Expanded Debug HUD

Let's show more useful info: ammo count, current weapon name, and a "DEAD" indicator.

### Update `debugHudSystem` in `debug_hud_system.cpp`

Replace the gather/draw section with this expanded version:

```cpp
void debugHudSystem(entt::registry& registry, int windowWidth,
    int windowHeight, float fps)
{
    auto* hudConfig = registry.ctx().find<HudConfig>();
    if (!hudConfig || hudConfig->shaderId == 0) return;

    glm::mat4 ortho = glm::ortho(0.0f, (float)windowWidth,
        (float)windowHeight, 0.0f, -1.0f, 1.0f);

    glDisable(GL_DEPTH_TEST);

    // ─── Gather debug info ───────────────────────────────────
    float health = 0.0f;
    float maxHealth = 0.0f;
    int shells = 0, rockets = 0;
    const char* weaponName = "None";
    bool isDead = false;

    auto playerView = registry.view<Health, TagPlayer>();
    for (auto [entity, hp] : playerView.each())
    {
        health = hp.current;
        maxHealth = hp.max;

        if (registry.all_of<Ammo>(entity))
        {
            auto& ammo = registry.get<Ammo>(entity);
            shells = ammo.shells;
            rockets = ammo.rockets;
        }

        if (registry.all_of<WeaponInventory>(entity))
        {
            auto& inv = registry.get<WeaponInventory>(entity);
            if (!inv.weapons.empty())
            {
                switch (inv.weapons[inv.currentWeapon].type)
                {
                    case WeaponType::Shotgun:        weaponName = "Shotgun"; break;
                    case WeaponType::SuperShotgun:   weaponName = "Super Shotgun"; break;
                    case WeaponType::Nailgun:        weaponName = "Nailgun"; break;
                    case WeaponType::RocketLauncher: weaponName = "Rocket Launcher"; break;
                    case WeaponType::GrenadeLauncher:weaponName = "Grenade Launcher"; break;
                    case WeaponType::LighteningGun:  weaponName = "Lightning Gun"; break;
                    case WeaponType::Railgun:        weaponName = "Railgun"; break;
                }
            }
        }

        if (registry.all_of<PlayerState>(entity))
        {
            isDead = registry.get<PlayerState>(entity).isDead;
        }
    }

    // ─── Draw text ───────────────────────────────────────────
    float textScale = 2.0f;
    unsigned int shader = hudConfig->shaderId;

    // FPS — always white
    char fpsText[64];
    snprintf(fpsText, sizeof(fpsText), "FPS: %.0f", fps);
    drawText(5.0f, 5.0f, fpsText, shader, ortho, textScale,
        glm::vec3(1.0f));

    // Health — colour-coded
    char healthText[64];
    snprintf(healthText, sizeof(healthText), "HP: %.0f / %.0f",
        health, maxHealth);

    glm::vec3 healthColor(1.0f);
    if (health <= 0.0f)
        healthColor = glm::vec3(1.0f, 0.0f, 0.0f);
    else if (health < 30.0f)
        healthColor = glm::vec3(1.0f, 1.0f, 0.0f);

    drawText(5.0f, 20.0f, healthText, shader, ortho, textScale,
        healthColor);

    // Weapon + Ammo
    char weaponText[128];
    snprintf(weaponText, sizeof(weaponText),
        "Weapon: %s  |  Shells: %d  Rockets: %d",
        weaponName, shells, rockets);
    drawText(5.0f, 35.0f, weaponText, shader, ortho, textScale,
        glm::vec3(0.8f, 0.8f, 1.0f));

    // Death message
    if (isDead)
    {
        const char* deathText = "YOU DIED - Respawning...";
        // Centre the death text on screen
        float textWidth = 24.0f * 7.0f;  // approximate width
        drawText((windowWidth / textScale - textWidth) * 0.5f,
            windowHeight / textScale * 0.4f,
            const_cast<char*>(deathText), shader, ortho, textScale,
            glm::vec3(1.0f, 0.0f, 0.0f));
    }

    // ─── Crosshair (only when alive) ─────────────────────────
    if (!isDead)
    {
        drawCrosshair(windowWidth, windowHeight, shader, ortho);
    }

    // ─── Damage flash overlay ────────────────────────────────
    float flashAlpha = 0.0f;
    auto stateView = registry.view<PlayerState, TagPlayer>();
    for (auto [entity, state] : stateView.each())
    {
        if (state.damageFlashTimer > 0.0f)
        {
            flashAlpha = state.damageFlashTimer / state.damageFlashDuration;
        }
    }

    if (flashAlpha > 0.0f)
    {
        drawDamageFlash(windowWidth, windowHeight, shader,
            ortho, flashAlpha);
    }

    glEnable(GL_DEPTH_TEST);
}
```

---

## Step 4: Damage Flash

When the player takes damage, we'll flash a red overlay across the screen that fades out quickly. This is the same technique Quake and DOOM use for damage feedback.

### Add the flash draw function to `debug_hud_system.cpp`

Add this above `debugHudSystem`:

```cpp
// ─── Draw a full-screen red overlay (damage flash) ───────────
static void drawDamageFlash(int windowWidth, int windowHeight,
    unsigned int shaderId, const glm::mat4& projection, float alpha)
{
    float w = (float)windowWidth;
    float h = (float)windowHeight;

    // Full-screen quad
    float verts[] = {
        0.0f, 0.0f, 0.0f, 0.0f,
        w,    0.0f, 0.0f, 0.0f,
        w,    h,    0.0f, 0.0f,
        0.0f, h,    0.0f, 0.0f,
    };
    unsigned int indices[] = { 0, 1, 2, 0, 2, 3 };

    glUseProgram(shaderId);
    GLint loc = glGetUniformLocation(shaderId, "projection");
    glUniformMatrix4fv(loc, 1, GL_FALSE, &projection[0][0]);

    glm::mat4 model(1.0f);
    loc = glGetUniformLocation(shaderId, "model");
    glUniformMatrix4fv(loc, 1, GL_FALSE, &model[0][0]);

    // Red with fading alpha
    loc = glGetUniformLocation(shaderId, "textColor");
    glUniform3f(loc, alpha * 0.8f, 0.0f, 0.0f);

    // Enable blending for the transparent overlay
    glEnable(GL_BLEND);
    glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);

    unsigned int vao, vbo, ebo;
    glGenVertexArrays(1, &vao);
    glGenBuffers(1, &vbo);
    glGenBuffers(1, &ebo);

    glBindVertexArray(vao);
    glBindBuffer(GL_ARRAY_BUFFER, vbo);
    glBufferData(GL_ARRAY_BUFFER, sizeof(verts), verts, GL_DYNAMIC_DRAW);
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ebo);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices),
        indices, GL_DYNAMIC_DRAW);

    glEnableVertexAttribArray(0);
    glVertexAttribPointer(0, 2, GL_FLOAT, GL_FALSE, 16, (void*)0);

    glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);

    glDisable(GL_BLEND);

    glBindVertexArray(0);
    glDeleteBuffers(1, &ebo);
    glDeleteBuffers(1, &vbo);
    glDeleteVertexArrays(1, &vao);
}
```

> **Note**: We're using the `textColor` uniform's red channel as a brightness value. Since the HUD fragment shader outputs `vec4(textColor, 1.0)`, the flash appears as a solid red overlay that fades by reducing the red channel. For a proper alpha-blended flash you'd need to modify the fragment shader to support an alpha uniform — we'll note that for Chapter 15a.

---

## Step 5: Damage Detection and Flash Trigger

We need to detect when the player takes damage and trigger the flash. The simplest approach: compare current health to the previous frame's health.

### New system: `src/engine/ecs/systems/player_state_system.h`

```cpp
#pragma once

#include <entt/entt.hpp>

void playerStateSystem(entt::registry& registry);
```

### New system: `src/engine/ecs/systems/player_state_system.cpp`

```cpp
#include "engine/ecs/systems/player_state_system.h"
#include "engine/ecs/components.h"
#include "engine/physics/physics_config.h"

void playerStateSystem(entt::registry& registry)
{
    const auto& config = registry.ctx().get<PhysicsConfig>();
    float dt = config.fixedDeltaTime;

    auto view = registry.view<Health, PlayerState, Position, Velocity,
        TagPlayer>();

    for (auto [entity, health, state, pos, vel] : view.each())
    {
        // ─── Damage flash ────────────────────────────────────
        if (health.current < state.previousHealth)
        {
            state.damageFlashTimer = state.damageFlashDuration;
        }
        state.previousHealth = health.current;

        // Count down the flash timer
        if (state.damageFlashTimer > 0.0f)
        {
            state.damageFlashTimer -= dt;
        }

        // ─── Death detection ─────────────────────────────────
        if (health.current <= 0.0f && !state.isDead)
        {
            state.isDead = true;
            state.respawnTimer = state.respawnDelay;
            vel.value = glm::vec3(0.0f);  // stop all movement
        }

        // ─── Respawn countdown ───────────────────────────────
        if (state.isDead)
        {
            state.respawnTimer -= dt;
            if (state.respawnTimer <= 0.0f)
            {
                // Reset everything
                pos.value = state.spawnPoint;
                vel.value = glm::vec3(0.0f);
                health.current = health.max;
                state.isDead = false;
                state.damageFlashTimer = 0.0f;
                state.previousHealth = health.max;
            }
        }

        // ─── Prevent input while dead ────────────────────────
        if (state.isDead && registry.all_of<PlayerInput>(entity))
        {
            auto& input = registry.get<PlayerInput>(entity);
            input.wishDir = glm::vec3(0.0f);
            input.jump = false;
            input.fire = false;
        }
    }
}
```

This system handles three things:
1. **Damage detection**: if health went down since last frame, start the flash timer
2. **Death**: when health hits 0, freeze the player and start a respawn countdown
3. **Respawn**: after the delay, teleport to spawn point and reset everything

---

## Step 6: Wire It All Up

### Add `playerStateSystem` to the tick order in `main.cpp`

The state system needs to run **after** damage is applied (after `triggerSystem` and `combatSystem`) but before the next frame's rendering:

```cpp
#include "engine/ecs/systems/player_state_system.h"  // NEW
```

```cpp
while (fixedTimestep.step())
{
    weaponSwitchSystem(registry);
    playerMovementSystem(registry);
    physicsSystem(registry);
    moverSystem(registry);
    collisionSystem(registry, spatialHash, level);
    movementSystem(registry);
    groundDetectionSystem(registry, level);
    combatSystem(registry, level);
    lifetimeSystem(registry);
    triggerSystem(registry);
    playerStateSystem(registry);          // NEW — after all damage sources
    demoResetSystem(registry);
}
```

### Add to `CMakeLists.txt`

```cmake
src/engine/ecs/systems/player_state_system.cpp
```

---

## The Full Updated `debug_hud_system.cpp`

For reference, here's the complete file with all additions:

```cpp
#include "engine/ecs/systems/debug_hud_system.h"
#include "engine/ecs/components.h"
#include <glad/glad.h>
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>
#include <stb_easy_font.h>
#include <cstdio>
#include <vector>

// ─── Internal: draw a string at screen position (x, y) ──────────
static void drawText(float x, float y, const char* text,
    unsigned int shaderId, const glm::mat4& projection, float scale,
    const glm::vec3& color)
{
    static char vertexBuffer[4096 * 16];
    int numQuads = stb_easy_font_print(x, y, const_cast<char*>(text),
        nullptr, vertexBuffer, sizeof(vertexBuffer));

    if (numQuads <= 0) return;

    std::vector<unsigned int> indices;
    indices.reserve(numQuads * 6);
    for (int i = 0; i < numQuads; i++)
    {
        unsigned int base = i * 4;
        indices.push_back(base + 0);
        indices.push_back(base + 1);
        indices.push_back(base + 2);
        indices.push_back(base + 0);
        indices.push_back(base + 2);
        indices.push_back(base + 3);
    }

    unsigned int vao, vbo, ebo;
    glGenVertexArrays(1, &vao);
    glGenBuffers(1, &vbo);
    glGenBuffers(1, &ebo);

    glBindVertexArray(vao);

    glBindBuffer(GL_ARRAY_BUFFER, vbo);
    glBufferData(GL_ARRAY_BUFFER, numQuads * 64,
        vertexBuffer, GL_DYNAMIC_DRAW);

    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ebo);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER,
        indices.size() * sizeof(unsigned int),
        indices.data(), GL_DYNAMIC_DRAW);

    glEnableVertexAttribArray(0);
    glVertexAttribPointer(0, 2, GL_FLOAT, GL_FALSE, 16, (void*)0);

    glUseProgram(shaderId);

    GLint loc = glGetUniformLocation(shaderId, "projection");
    glUniformMatrix4fv(loc, 1, GL_FALSE, &projection[0][0]);

    glm::mat4 model = glm::mat4(1.0f);
    model = glm::scale(model, glm::vec3(scale, scale, 1.0f));
    loc = glGetUniformLocation(shaderId, "model");
    glUniformMatrix4fv(loc, 1, GL_FALSE, &model[0][0]);

    loc = glGetUniformLocation(shaderId, "textColor");
    glUniform3fv(loc, 1, &color[0]);

    glDrawElements(GL_TRIANGLES, (int)indices.size(),
        GL_UNSIGNED_INT, 0);

    glBindVertexArray(0);
    glDeleteBuffers(1, &ebo);
    glDeleteBuffers(1, &vbo);
    glDeleteVertexArrays(1, &vao);
}

// ─── Draw a crosshair at screen centre ───────────────────────
static void drawCrosshair(int windowWidth, int windowHeight,
    unsigned int shaderId, const glm::mat4& projection)
{
    float cx = windowWidth * 0.5f;
    float cy = windowHeight * 0.5f;
    float size = 10.0f;
    float thickness = 1.0f;

    float hVerts[] = {
        cx - size, cy - thickness, 0.0f, 0.0f,
        cx + size, cy - thickness, 0.0f, 0.0f,
        cx + size, cy + thickness, 0.0f, 0.0f,
        cx - size, cy + thickness, 0.0f, 0.0f,
    };
    float vVerts[] = {
        cx - thickness, cy - size, 0.0f, 0.0f,
        cx + thickness, cy - size, 0.0f, 0.0f,
        cx + thickness, cy + size, 0.0f, 0.0f,
        cx - thickness, cy + size, 0.0f, 0.0f,
    };
    unsigned int indices[] = { 0, 1, 2, 0, 2, 3 };

    glUseProgram(shaderId);
    GLint loc = glGetUniformLocation(shaderId, "projection");
    glUniformMatrix4fv(loc, 1, GL_FALSE, &projection[0][0]);

    glm::mat4 model(1.0f);
    loc = glGetUniformLocation(shaderId, "model");
    glUniformMatrix4fv(loc, 1, GL_FALSE, &model[0][0]);

    loc = glGetUniformLocation(shaderId, "textColor");
    glUniform3f(loc, 1.0f, 1.0f, 1.0f);

    unsigned int vao, vbo, ebo;
    glGenVertexArrays(1, &vao);
    glGenBuffers(1, &vbo);
    glGenBuffers(1, &ebo);
    glBindVertexArray(vao);

    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ebo);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices),
        indices, GL_DYNAMIC_DRAW);

    glEnableVertexAttribArray(0);
    glVertexAttribPointer(0, 2, GL_FLOAT, GL_FALSE, 16, (void*)0);

    glBindBuffer(GL_ARRAY_BUFFER, vbo);
    glBufferData(GL_ARRAY_BUFFER, sizeof(hVerts),
        hVerts, GL_DYNAMIC_DRAW);
    glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);

    glBufferData(GL_ARRAY_BUFFER, sizeof(vVerts),
        vVerts, GL_DYNAMIC_DRAW);
    glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);

    glBindVertexArray(0);
    glDeleteBuffers(1, &ebo);
    glDeleteBuffers(1, &vbo);
    glDeleteVertexArrays(1, &vao);
}

// ─── Draw a full-screen red overlay (damage flash) ───────────
static void drawDamageFlash(int windowWidth, int windowHeight,
    unsigned int shaderId, const glm::mat4& projection, float alpha)
{
    float w = (float)windowWidth;
    float h = (float)windowHeight;

    float verts[] = {
        0.0f, 0.0f, 0.0f, 0.0f,
        w,    0.0f, 0.0f, 0.0f,
        w,    h,    0.0f, 0.0f,
        0.0f, h,    0.0f, 0.0f,
    };
    unsigned int indices[] = { 0, 1, 2, 0, 2, 3 };

    glUseProgram(shaderId);
    GLint loc = glGetUniformLocation(shaderId, "projection");
    glUniformMatrix4fv(loc, 1, GL_FALSE, &projection[0][0]);

    glm::mat4 model(1.0f);
    loc = glGetUniformLocation(shaderId, "model");
    glUniformMatrix4fv(loc, 1, GL_FALSE, &model[0][0]);

    loc = glGetUniformLocation(shaderId, "textColor");
    glUniform3f(loc, alpha * 0.8f, 0.0f, 0.0f);

    glEnable(GL_BLEND);
    glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);

    unsigned int vao, vbo, ebo;
    glGenVertexArrays(1, &vao);
    glGenBuffers(1, &vbo);
    glGenBuffers(1, &ebo);

    glBindVertexArray(vao);
    glBindBuffer(GL_ARRAY_BUFFER, vbo);
    glBufferData(GL_ARRAY_BUFFER, sizeof(verts), verts, GL_DYNAMIC_DRAW);
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ebo);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices),
        indices, GL_DYNAMIC_DRAW);

    glEnableVertexAttribArray(0);
    glVertexAttribPointer(0, 2, GL_FLOAT, GL_FALSE, 16, (void*)0);

    glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);

    glDisable(GL_BLEND);

    glBindVertexArray(0);
    glDeleteBuffers(1, &ebo);
    glDeleteBuffers(1, &vbo);
    glDeleteVertexArrays(1, &vao);
}

// ─── Main HUD system ────────────────────────────────────────
void debugHudSystem(entt::registry& registry, int windowWidth,
    int windowHeight, float fps)
{
    auto* hudConfig = registry.ctx().find<HudConfig>();
    if (!hudConfig || hudConfig->shaderId == 0) return;

    glm::mat4 ortho = glm::ortho(0.0f, (float)windowWidth,
        (float)windowHeight, 0.0f, -1.0f, 1.0f);

    glDisable(GL_DEPTH_TEST);

    // ─── Gather debug info ───────────────────────────────────
    float health = 0.0f;
    float maxHealth = 0.0f;
    int shells = 0, rockets = 0;
    const char* weaponName = "None";
    bool isDead = false;

    auto playerView = registry.view<Health, TagPlayer>();
    for (auto [entity, hp] : playerView.each())
    {
        health = hp.current;
        maxHealth = hp.max;

        if (registry.all_of<Ammo>(entity))
        {
            auto& ammo = registry.get<Ammo>(entity);
            shells = ammo.shells;
            rockets = ammo.rockets;
        }

        if (registry.all_of<WeaponInventory>(entity))
        {
            auto& inv = registry.get<WeaponInventory>(entity);
            if (!inv.weapons.empty())
            {
                switch (inv.weapons[inv.currentWeapon].type)
                {
                    case WeaponType::Shotgun:
                        weaponName = "Shotgun"; break;
                    case WeaponType::SuperShotgun:
                        weaponName = "Super Shotgun"; break;
                    case WeaponType::Nailgun:
                        weaponName = "Nailgun"; break;
                    case WeaponType::RocketLauncher:
                        weaponName = "Rocket Launcher"; break;
                    case WeaponType::GrenadeLauncher:
                        weaponName = "Grenade Launcher"; break;
                    case WeaponType::LighteningGun:
                        weaponName = "Lightning Gun"; break;
                    case WeaponType::Railgun:
                        weaponName = "Railgun"; break;
                }
            }
        }

        if (registry.all_of<PlayerState>(entity))
        {
            isDead = registry.get<PlayerState>(entity).isDead;
        }
    }

    // ─── Draw text ───────────────────────────────────────────
    float textScale = 2.0f;
    unsigned int shader = hudConfig->shaderId;

    char fpsText[64];
    snprintf(fpsText, sizeof(fpsText), "FPS: %.0f", fps);
    drawText(5.0f, 5.0f, fpsText, shader, ortho, textScale,
        glm::vec3(1.0f));

    char healthText[64];
    snprintf(healthText, sizeof(healthText), "HP: %.0f / %.0f",
        health, maxHealth);

    glm::vec3 healthColor(1.0f);
    if (health <= 0.0f)
        healthColor = glm::vec3(1.0f, 0.0f, 0.0f);
    else if (health < 30.0f)
        healthColor = glm::vec3(1.0f, 1.0f, 0.0f);

    drawText(5.0f, 20.0f, healthText, shader, ortho, textScale,
        healthColor);

    char weaponText[128];
    snprintf(weaponText, sizeof(weaponText),
        "Weapon: %s  |  Shells: %d  Rockets: %d",
        weaponName, shells, rockets);
    drawText(5.0f, 35.0f, weaponText, shader, ortho, textScale,
        glm::vec3(0.8f, 0.8f, 1.0f));

    if (isDead)
    {
        drawText(5.0f, 55.0f, "YOU DIED - Respawning...",
            shader, ortho, textScale * 1.5f,
            glm::vec3(1.0f, 0.0f, 0.0f));
    }

    // ─── Crosshair (only when alive) ─────────────────────────
    if (!isDead)
    {
        drawCrosshair(windowWidth, windowHeight, shader, ortho);
    }

    // ─── Damage flash overlay ────────────────────────────────
    float flashAlpha = 0.0f;
    auto stateView = registry.view<PlayerState, TagPlayer>();
    for (auto [entity, state] : stateView.each())
    {
        if (state.damageFlashTimer > 0.0f)
        {
            flashAlpha = state.damageFlashTimer
                / state.damageFlashDuration;
        }
    }

    if (flashAlpha > 0.0f)
    {
        drawDamageFlash(windowWidth, windowHeight, shader,
            ortho, flashAlpha);
    }

    glEnable(GL_DEPTH_TEST);
}
```

---

## What Changed — Summary

| File | Change |
|------|--------|
| `components.h` | Added `PlayerState` struct (spawn point, damage flash, death/respawn) |
| `scene_setup.cpp` | Attached `PlayerState` to the player entity |
| `debug_hud_system.cpp` | Added crosshair, expanded HUD (weapon, ammo, death text), damage flash overlay |
| `player_state_system.h/cpp` | **New** — damage detection, death, respawn |
| `main.cpp` | Added `playerStateSystem` to tick order, added include |
| `CMakeLists.txt` | Added `player_state_system.cpp` |

---

## What You Should See

After building and running:

1. **White crosshair** in the centre of the screen — aim at things
2. **Expanded HUD** — top-left shows FPS, health, weapon name, ammo counts
3. **Walk into lava** — screen flashes red, health ticks down, drops to 0
4. **"YOU DIED"** message appears in red when health reaches 0
5. **Automatic respawn** after 1.5 seconds — teleported back to spawn point with full health
6. **Crosshair disappears while dead** — no shooting when dead
7. **Shoot yourself with a rocket** — splash damage triggers the red flash

### Troubleshooting

**Damage flash not visible:**
- Check that `PlayerState` is attached to the player entity
- The flash uses `textColor` as brightness — make sure the HUD shader is loaded
- Verify `playerStateSystem` runs after `triggerSystem` in the tick order

**Never respawns:**
- Check that `respawnTimer` counts down — `playerStateSystem` must be in the fixed timestep loop
- Verify `respawnDelay` isn't too high

**Crosshair off-centre:**
- The crosshair uses `windowWidth * 0.5f` and `windowHeight * 0.5f`. Make sure `window.getWidth()` / `window.getHeight()` return the correct framebuffer size (not the window size — these differ on high-DPI displays)

---

## What's Next

The game now has a complete gameplay loop: move, jump, shoot, take damage, die, respawn. In **Chapter 15a**, we'll clean everything up — fix typos, refactor the scene setup, document the system tick order, and ensure everything is consistent before moving on to TrenchBroom integration.
