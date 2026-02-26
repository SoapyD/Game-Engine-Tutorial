# Chapter 14: Moon Jumping & Platforming

## What You'll Learn
- Tuning gravity and jump force for a floaty, moon-like feel
- Variable jump height — tap for a short hop, hold for a full leap
- Coyote time — a grace period that lets you jump just after leaving an edge
- Building a platforming test arena with floating platforms
- Verifying that lifts and moving platforms carry the player

---

## The Goal

Chapter 13 gave the player a physical body with gravity, friction, and jumping. The default values are tuned for Quake-style ground combat — snappy, fast, grounded. Now we want the opposite: **floaty, forgiving, platformer-friendly** movement that feels like jumping on the moon.

We'll also build a proper test arena with platforms at varying heights so you can feel the movement and verify that everything works.

---

## Step 1: Tune the Physics

The `CharacterPhysics` component has all the knobs we need. The defaults from Chapter 10 are Quake-tuned:

| Parameter | Quake Default | Moon Tuned | Why |
|-----------|--------------|------------|-----|
| `gravity.strength` | 20.0 | 8.0 | Lower gravity = longer hang time |
| `jumpForce` | 8.0 | 10.0 | Higher jump to match lower gravity |
| `maxAirSpeed` | 1.0 | 3.0 | More air control for platforming |
| `airFriction` | 0.1 | 0.1 | Keep low for smooth air movement |
| `maxGroundSpeed` | 7.0 | 7.0 | Ground speed stays the same |
| `groundFriction` | 6.0 | 6.0 | Snappy ground stopping stays |

### Update `scene_setup.cpp`

Change the player's `Gravity` and `CharacterPhysics` setup:

```cpp
// ─── Player entity ────────────────────────────────────────
auto player = registry.create();
registry.emplace<Position>(player, glm::vec3(15.0f, 1.7f, 15.0f));
registry.emplace<Velocity>(player);
registry.emplace<AABBCollider>(player, glm::vec3(0.3f, 0.85f, 0.3f), false);
registry.emplace<Gravity>(player, 8.0f);                        // CHANGED: lower gravity
registry.emplace<OnGround>(player);
auto& charPhys = registry.emplace<CharacterPhysics>(player);    // CHANGED: grab reference
charPhys.jumpForce = 10.0f;                                     // NEW: higher jump
charPhys.maxAirSpeed = 3.0f;                                    // NEW: better air control
registry.emplace<Health>(player, 100.0f, 100.0f);
registry.emplace<PlayerInput>(player);
registry.emplace<TagPlayer>(player);
```

Build and run — WASD + Spacebar should now give you a noticeably floaty jump with good air control. You'll hang in the air longer and can steer mid-jump.

---

## Step 2: Variable Jump Height

Right now, tapping Spacebar gives the same jump as holding it. In a good platformer, you want **tap = short hop, hold = full jump**. This gives the player fine-grained height control.

The technique is simple: when the player **releases** the jump key while still moving upward, we cut the vertical velocity.

### Add to `CharacterPhysics` in `components.h`

```cpp
struct CharacterPhysics
{
    float groundFriction = 6.0f;
    float airFriction = 0.1f;
    float maxGroundSpeed = 7.0f;
    float maxAirSpeed = 1.0f;
    float groundAcceleration = 10.0f;
    float airAcceleration = 10.0f;
    float jumpForce = 8.0f;
    float stepHeight = 0.5f;
    float jumpCutMultiplier = 0.4f;   // NEW: velocity multiplier on early jump release
};
```

When `jumpCutMultiplier = 0.4`, releasing jump early cuts your upward speed to 40% — a noticeable difference without feeling jarring.

### Add to `PlayerInput` in `components.h`

We need to detect the **transition** from jump-pressed to jump-released. Add a field to track last frame's state:

```cpp
struct PlayerInput
{
    bool fire = false;
    int weaponSwitch = -1;
    glm::vec3 wishDir = glm::vec3(0.0f);
    bool jump = false;
    bool jumpPrevious = false;   // NEW: was jump held last frame?
};
```

### Update `player_movement_system.cpp`

Add the jump-cut logic after the jump initiation:

```cpp
void playerMovementSystem(entt::registry& registry)
{
    const auto& config = registry.ctx().get<PhysicsConfig>();
    float dt = config.fixedDeltaTime;

    auto view = registry.view<PlayerInput, Velocity, OnGround, CharacterPhysics>();

    for (auto [entity, input, vel, ground, phys] : view.each())
    {
        // ─── Jumping ─────────────────────────────────────────
        if (input.jump && ground.value)
        {
            vel.value.y = phys.jumpForce;
            ground.value = false;
        }

        // ─── Variable jump height ────────────────────────────  NEW
        // If the player released jump while still rising, cut velocity
        if (!input.jump && input.jumpPrevious && vel.value.y > 0.0f)
        {
            vel.value.y *= phys.jumpCutMultiplier;
        }
        input.jumpPrevious = input.jump;

        // ─── Horizontal acceleration (Quake-style) ──────────
        glm::vec3 wishDir = input.wishDir;
        float wishSpeed;
        float accel;

        if (ground.value)
        {
            wishSpeed = phys.maxGroundSpeed;
            accel = phys.groundAcceleration;
        }
        else
        {
            wishSpeed = phys.maxAirSpeed;
            accel = phys.airAcceleration;
        }

        if (glm::length(wishDir) < 0.01f)
            continue;

        float currentSpeed = glm::dot(
            glm::vec3(vel.value.x, 0.0f, vel.value.z), wishDir
        );
        float addSpeed = wishSpeed - currentSpeed;

        if (addSpeed <= 0.0f)
            continue;

        float accelSpeed = accel * wishSpeed * dt;
        if (accelSpeed > addSpeed)
            accelSpeed = addSpeed;

        vel.value.x += wishDir.x * accelSpeed;
        vel.value.z += wishDir.z * accelSpeed;
    }
}
```

The key line is:
```cpp
if (!input.jump && input.jumpPrevious && vel.value.y > 0.0f)
```

This fires exactly once — on the frame where jump transitions from pressed to released — and only while the player is still rising. Cutting velocity while falling would feel wrong.

---

## Step 3: Coyote Time

**Coyote time** (named after Wile E. Coyote running off a cliff) gives the player a brief grace period after walking off an edge where they can still jump. Without it, players who press jump one frame too late after leaving a platform get nothing — it feels unfair and frustrating.

### How it works

```
                    ┌─ coyote window (0.1s) ─┐
                    │                         │
  ████████████████──┘                         │ ← can still jump here
                    ground.value = false       │
                                              └─ too late, falling
```

Instead of checking `ground.value` for jump, we check whether the player was on the ground within the last N seconds.

### Add to `CharacterPhysics` in `components.h`

```cpp
struct CharacterPhysics
{
    float groundFriction = 6.0f;
    float airFriction = 0.1f;
    float maxGroundSpeed = 7.0f;
    float maxAirSpeed = 1.0f;
    float groundAcceleration = 10.0f;
    float airAcceleration = 10.0f;
    float jumpForce = 8.0f;
    float stepHeight = 0.5f;
    float jumpCutMultiplier = 0.4f;
    float coyoteTime = 0.1f;          // NEW: grace period in seconds
    float coyoteTimer = 0.0f;         // NEW: runtime countdown
};
```

### Update `player_movement_system.cpp`

Replace the jump section:

```cpp
// ─── Coyote time tracking ────────────────────────────
if (ground.value)
{
    phys.coyoteTimer = phys.coyoteTime;  // reset while grounded
}
else
{
    phys.coyoteTimer -= dt;  // count down while airborne
}

bool canJump = ground.value || phys.coyoteTimer > 0.0f;

// ─── Jumping ─────────────────────────────────────────
if (input.jump && canJump)
{
    vel.value.y = phys.jumpForce;
    ground.value = false;
    phys.coyoteTimer = 0.0f;  // consume the coyote window
}
```

The `coyoteTimer = 0.0f` after jumping prevents the player from getting a double jump — once you use the coyote window, it's consumed.

### The full updated `player_movement_system.cpp`

```cpp
#include "engine/ecs/systems/player_movement_system.h"
#include "engine/ecs/components.h"
#include "engine/physics/physics_config.h"

void playerMovementSystem(entt::registry& registry)
{
    const auto& config = registry.ctx().get<PhysicsConfig>();
    float dt = config.fixedDeltaTime;

    auto view = registry.view<PlayerInput, Velocity, OnGround, CharacterPhysics>();

    for (auto [entity, input, vel, ground, phys] : view.each())
    {
        // ─── Coyote time tracking ────────────────────────────
        if (ground.value)
        {
            phys.coyoteTimer = phys.coyoteTime;
        }
        else
        {
            phys.coyoteTimer -= dt;
        }

        bool canJump = ground.value || phys.coyoteTimer > 0.0f;

        // ─── Jumping ─────────────────────────────────────────
        if (input.jump && canJump)
        {
            vel.value.y = phys.jumpForce;
            ground.value = false;
            phys.coyoteTimer = 0.0f;
        }

        // ─── Variable jump height ────────────────────────────
        if (!input.jump && input.jumpPrevious && vel.value.y > 0.0f)
        {
            vel.value.y *= phys.jumpCutMultiplier;
        }
        input.jumpPrevious = input.jump;

        // ─── Horizontal acceleration (Quake-style) ──────────
        glm::vec3 wishDir = input.wishDir;
        float wishSpeed;
        float accel;

        if (ground.value)
        {
            wishSpeed = phys.maxGroundSpeed;
            accel = phys.groundAcceleration;
        }
        else
        {
            wishSpeed = phys.maxAirSpeed;
            accel = phys.airAcceleration;
        }

        if (glm::length(wishDir) < 0.01f)
            continue;

        float currentSpeed = glm::dot(
            glm::vec3(vel.value.x, 0.0f, vel.value.z), wishDir
        );
        float addSpeed = wishSpeed - currentSpeed;

        if (addSpeed <= 0.0f)
            continue;

        float accelSpeed = accel * wishSpeed * dt;
        if (accelSpeed > addSpeed)
            accelSpeed = addSpeed;

        vel.value.x += wishDir.x * accelSpeed;
        vel.value.z += wishDir.z * accelSpeed;
    }
}
```

---

## Step 4: The Platforming Test Arena

Now we need somewhere to test all this. We'll add floating platforms at various heights to the showcase level in `scene_setup.cpp`.

### Platform Layout

```
Top-down view of the 30x30 room:

     0         10        20        30
  0  ┌─────────────────────────────┐
     │                             │
     │  [teleporter]               │
  5  │       [shelf+cubes]         │
     │                             │
     │              [P1]           │
 10  │  [lights]         [P2]     │
     │                      ↓      │
     │              [P3]  [P4]     │
 15  │  [hallcube] ─────→ [door]  │
     │                             │
     │         [P5 moving]         │
 20  │                             │
     │                             │
 25  │  [lift]    [lava]  [P6]    │
     │                             │
 30  └─────────────────────────────┘

P1: Low platform (y=1.5) — easy first jump
P2: Medium platform (y=3.0) — requires a bigger jump
P3: Stepping stone (y=2.0) — bridge between P1 and P4
P4: High platform (y=4.0) — requires jump from P2 or P3
P5: Moving platform — rises and falls
P6: Small platform over lava (y=1.5) — risk/reward
```

### Add platforms to `scene_setup.cpp`

Add this section after the existing physics demos:

```cpp
// ═══════════════════════════════════════════════════════════
// CHAPTER 14 — Platforming Test Arena
// ═══════════════════════════════════════════════════════════

// P1: Low platform — easy first jump from ground level
{
    auto platform = registry.create();
    registry.emplace<Position>(platform, glm::vec3(15.0f, 0.75f, 10.0f));
    registry.emplace<Scale>(platform, glm::vec3(3.0f, 1.5f, 3.0f));
    registry.emplace<AABBCollider>(platform,
        glm::vec3(1.5f, 0.75f, 1.5f), false);
    registry.emplace<MeshRenderer>(platform, cubeMesh->getVAO(), 0u,
        litShader->getId(), gridBlue->getId(),
        true, cubeMesh->getIndexCount());
}

// P2: Medium platform — requires a bigger jump
{
    auto platform = registry.create();
    registry.emplace<Position>(platform, glm::vec3(21.0f, 1.5f, 10.0f));
    registry.emplace<Scale>(platform, glm::vec3(3.0f, 3.0f, 3.0f));
    registry.emplace<AABBCollider>(platform,
        glm::vec3(1.5f, 1.5f, 1.5f), false);
    registry.emplace<MeshRenderer>(platform, cubeMesh->getVAO(), 0u,
        litShader->getId(), gridBlue->getId(),
        true, cubeMesh->getIndexCount());
}

// P3: Stepping stone — bridge between P1 and P4
{
    auto platform = registry.create();
    registry.emplace<Position>(platform, glm::vec3(18.0f, 1.0f, 13.0f));
    registry.emplace<Scale>(platform, glm::vec3(2.0f, 2.0f, 2.0f));
    registry.emplace<AABBCollider>(platform,
        glm::vec3(1.0f, 1.0f, 1.0f), false);
    registry.emplace<MeshRenderer>(platform, cubeMesh->getVAO(), 0u,
        litShader->getId(), gridGreen->getId(),
        true, cubeMesh->getIndexCount());
}

// P4: High platform — jump from P2 or P3
{
    auto platform = registry.create();
    registry.emplace<Position>(platform, glm::vec3(22.0f, 2.0f, 13.0f));
    registry.emplace<Scale>(platform, glm::vec3(3.0f, 4.0f, 3.0f));
    registry.emplace<AABBCollider>(platform,
        glm::vec3(1.5f, 2.0f, 1.5f), false);
    registry.emplace<MeshRenderer>(platform, cubeMesh->getVAO(), 0u,
        litShader->getId(), gridBlue->getId(),
        true, cubeMesh->getIndexCount());
}

// P5: Moving platform — rises and falls in the middle of the room
{
    auto movePlat = registry.create();
    glm::vec3 moveLow(15.0f, 0.1f, 20.0f);
    glm::vec3 moveHigh(15.0f, 3.1f, 20.0f);

    registry.emplace<Position>(movePlat, moveLow);
    registry.emplace<Scale>(movePlat, glm::vec3(3.0f, 0.2f, 3.0f));
    registry.emplace<AABBCollider>(movePlat,
        glm::vec3(1.5f, 0.1f, 1.5f), false);
    registry.emplace<MeshRenderer>(movePlat, cubeMesh->getVAO(), 0u,
        litShader->getId(), gridOrange->getId(),
        true, cubeMesh->getIndexCount());
    registry.emplace<Mover>(movePlat, moveLow, moveHigh,
        1.5f, 2.0f, 0.0f, 0.0f, MoverState::Idle, true);

    // Trigger to activate the moving platform
    auto moveTrigger = registry.create();
    registry.emplace<Position>(moveTrigger, glm::vec3(15.0f, 0.3f, 20.0f));
    registry.emplace<AABBCollider>(moveTrigger,
        glm::vec3(1.5f, 0.3f, 1.5f), true);
    registry.emplace<TriggerVolume>(moveTrigger,
        TriggerAction::ActivateMover, movePlat,
        glm::vec3(0.0f), 0.0f, "", false, false, 0.5f, 0.0f);
}

// P6: Small platform hovering over the lava pit — risk/reward
{
    auto platform = registry.create();
    registry.emplace<Position>(platform, glm::vec3(24.0f, 0.75f, 25.0f));
    registry.emplace<Scale>(platform, glm::vec3(2.0f, 1.5f, 2.0f));
    registry.emplace<AABBCollider>(platform,
        glm::vec3(1.0f, 0.75f, 1.0f), false);
    registry.emplace<MeshRenderer>(platform, cubeMesh->getVAO(), 0u,
        litShader->getId(), gridGreen->getId(),
        true, cubeMesh->getIndexCount());
}
```

---

## Step 5: Verify Platform Riding

The player should be **carried** by moving platforms (lifts and the new P5 mover). This already works because:

1. The mover system moves the platform's `Position`
2. The ground detection system checks entity colliders and snaps the player to the top
3. The collision system prevents the player from falling through

However, there's a subtlety: the **ground detection** system only snaps vertically. If a platform moves horizontally, the player would slide off. For vertical lifts (which is all we have), this isn't an issue — the player rides up and down naturally.

If you try standing on the existing lift (at position 10, 25) or the new P5 mover (at 15, 20), you should rise with it when it activates.

### Test checklist

Try each of these and verify they work:

- [ ] Jump onto P1 from ground level
- [ ] Jump from P1 to P2 (requires full jump hold)
- [ ] Tap-jump from P1 — should be a noticeably shorter hop
- [ ] Walk off P1's edge and jump during the coyote window — should still jump
- [ ] Walk off P1's edge and wait too long — should fall
- [ ] Jump across the stepping stones P1 → P3 → P4
- [ ] Stand on P5's trigger to activate it, ride it up
- [ ] Jump from P6 over the lava — miss and take damage, health drops to 0
- [ ] Ride the existing lift (at the back-left of the room)

---

## What Changed — Summary

| File | Change |
|------|--------|
| `components.h` | Added `jumpCutMultiplier`, `coyoteTime`, `coyoteTimer` to `CharacterPhysics`; added `jumpPrevious` to `PlayerInput` |
| `scene_setup.cpp` | Moon-tuned gravity (8.0), jump force (10.0), air speed (3.0); added 6 platforming platforms |
| `player_movement_system.cpp` | Added coyote time tracking, variable jump height, jump-cut logic |

---

## What You Should See

After building and running:

1. **Floaty jumps** — the player hangs in the air noticeably longer than before
2. **Variable jump height** — tap Space for a short hop, hold for full height. The difference should be very clear.
3. **Coyote time** — walk off a platform edge and press jump within ~0.1 seconds. You'll still jump. Wait longer and you'll fall.
4. **Platforming arena** — six platforms at various heights to jump between
5. **Moving platform** — step on P5's trigger, ride it upward
6. **Lava danger** — miss the jump to P6 and fall into the red lava pool, watch health tick down

### Troubleshooting

**Jumps feel too floaty / not floaty enough:**
- Adjust `Gravity` strength (lower = more floaty). Try values between 5.0 and 15.0.
- Adjust `jumpForce` to match — higher gravity needs higher jump force.

**Variable jump doesn't feel different:**
- Lower `jumpCutMultiplier` for a more dramatic effect (0.2 = very short tap-jump). Higher values (0.6-0.8) make the difference subtler.

**Coyote time feels too generous / not generous enough:**
- `coyoteTime = 0.1` is standard. 0.15 is generous. 0.05 is tight. Most platformers use 0.08-0.12.

**Player slides off moving platform:**
- This only happens with horizontally-moving platforms. Our movers are all vertical, so this shouldn't occur. If you add horizontal movers later, you'll need to track the platform's velocity and add it to the player's movement.

**Can't reach P4:**
- Jump from P2 (medium height) or from P3 (stepping stone). A full-hold jump from P2 should reach P4's top.

---

## New C++ Concept: Detecting Input Transitions

The variable jump height requires detecting when a button **transitions** from pressed to released — not just whether it's currently pressed. This is a common pattern in game input:

```cpp
// Detect "just released" — was pressed, now isn't
bool justReleased = !input.jump && input.jumpPrevious;

// Detect "just pressed" — wasn't pressed, now is
bool justPressed = input.jump && !input.jumpPrevious;

// Update for next frame
input.jumpPrevious = input.jump;
```

The `jumpPrevious` field stores last frame's state so we can compare. This pattern works for any button — fire, use, crouch, etc. The key is updating `jumpPrevious` **after** all checks that use it, at the end of the frame's input processing.

---

## What's Next

Movement feels good — the player can jump between platforms with satisfying air control and forgiving coyote time. In **Chapter 15**, we'll add the gameplay layer: a crosshair, expanded HUD, death and respawn, and damage feedback.
