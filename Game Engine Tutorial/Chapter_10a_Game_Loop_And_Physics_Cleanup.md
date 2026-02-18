# Chapter 10a: Game Loop & Physics Cleanup

> **Prerequisites:** Chapter 10 (Physics & Movement) completed. You should have a working game loop with gravity, jumping, friction, and basic collision response.

---

## Why This Chapter?

If you have been following along, your `main.cpp` is starting to groan under its own weight. The fixed timestep logic is a chunk of boilerplate sitting right in your game loop. Gravity is `9.81f` in one system and `9.8f` in another (or is it `-9.81f`?). Collision layer masks are `0x01` and `0x02` at file scope with no indication of what they mean. And the order your systems run in? That is just "whatever order you happened to type the function calls."

None of this is *broken*. It all works. But it is the kind of code that quietly rots. You change the gravity constant in one place and forget the other. You reorder a system call and spend an hour debugging why jumping feels wrong. You come back in two weeks and cannot remember what `0x04` means.

This chapter is a **cleanup pass** -- no new features, just organising what we have. We will extract four small, focused pieces:

1. **FixedTimestep** -- wraps the accumulator pattern in a reusable class
2. **PhysicsConfig** -- centralises all those magic numbers
3. **CollisionLayers** -- gives names to bitmask constants
4. **System phase ordering** -- documents and formalises the execution order

By the end, your game loop will be shorter, your physics parameters will live in one place, and the next person to read your code (including future you) will thank you.

---

## The Problem: Before

Let us look at what a typical `main.cpp` game loop looks like after Chapter 10. Yours may differ in details, but the shape is usually something like this:

```cpp
// main.cpp - after Chapter 10
// ... includes, window/input/resource setup from Chapter 5a ...

// Magic numbers at file scope
constexpr uint32_t LAYER_PLAYER     = 0x01;
constexpr uint32_t LAYER_WORLD      = 0x02;
constexpr uint32_t LAYER_ENEMY      = 0x04;
constexpr uint32_t LAYER_PROJECTILE = 0x08;

int main()
{
	Window window(1280, 720, "QEngine");
	InputManager input;
	input.init(window.getHandle());
	ResourceManager resources;

	// ... resource loading, mesh creation ...

	entt::registry registry;
	setupScene(registry, /* ... */);

	// Fixed timestep variables scattered in main
	const float fixedDeltaTime = 1.0f / 60.0f;
	float accumulator = 0.0f;
	auto previousTime = std::chrono::high_resolution_clock::now();

	while (!window.shouldClose())
	{
		auto currentTime = std::chrono::high_resolution_clock::now();
		float deltaTime = std::chrono::duration<float>(currentTime - previousTime).count();
		previousTime = currentTime;

		// Clamp to avoid spiral of death
		if (deltaTime > 0.25f)
			deltaTime = 0.25f;

		accumulator += deltaTime;

		window.pollEvents();
		input.update();
		inputSystem(registry);

		while (accumulator >= fixedDeltaTime)
		{
			// Physics with magic numbers baked into systems
			gravitySystem(registry, fixedDeltaTime);       // has 9.81f inside
			movementSystem(registry, fixedDeltaTime);       // has friction 0.85f, air control 0.3f inside
			collisionSystem(registry, fixedDeltaTime);      // has layer checks using raw hex
			jumpSystem(registry);                           // has jumpVelocity 5.0f inside

			accumulator -= fixedDeltaTime;
		}

		float alpha = accumulator / fixedDeltaTime;

		renderSystem(registry, alpha);

		window.swapBuffers();
	}

	// ... cleanup ...
}
```

Count the problems:

- **The accumulator pattern** is ~15 lines of boilerplate that will be identical in every project. It has nothing to do with *your* game.
- **Physics constants** are buried inside system functions. To tweak jump height, you have to find which file `5.0f` lives in. Good luck if two systems both reference gravity with slightly different values.
- **Collision layers** are raw hex literals. `0x04` means "enemy" but only if you remember that.
- **System ordering** is implicit. There is no documentation for *why* input comes before physics, or why render is last. Reordering is one careless cut-and-paste away.

Let us fix each of these.

---

## 1. The FixedTimestep Class

### The Concept: Accumulator Pattern

The fixed timestep accumulator is a well-known pattern in game development. The idea is simple: real time marches forward in irregular chunks (your frame delta time), but physics needs to advance in fixed, deterministic steps. The accumulator bridges the gap -- it collects real time and dispenses it in fixed-size portions.

The three operations are always the same:

1. **Accumulate** -- add the frame's delta time to the accumulator (with a clamp to prevent the "spiral of death")
2. **Step** -- while the accumulator holds at least one fixed step's worth of time, consume it and run physics
3. **Get alpha** -- after stepping, the leftover fraction tells the renderer how far between the previous and current physics state we are, enabling interpolation for smooth visuals

This pattern never changes between projects. It is a perfect candidate for extraction into a class.

### The Code

```cpp
// engine/core/fixed_timestep.h
#pragma once

#include <chrono>

class FixedTimestep
{
public:
	explicit FixedTimestep(float timestep = 1.0f / 60.0f, float maxDeltaTime = 0.25f)
		: m_timestep(timestep)
		, m_maxDeltaTime(maxDeltaTime)
		, m_accumulator(0.0f)
		, m_previousTime(std::chrono::high_resolution_clock::now())
	{
	}

	// Call once per frame, at the top of the game loop.
	// Measures elapsed time since last call and adds it to the accumulator.
	void accumulate()
	{
		auto currentTime = std::chrono::high_resolution_clock::now();
		float deltaTime = std::chrono::duration<float>(currentTime - m_previousTime).count();
		m_previousTime = currentTime;

		// Clamp to prevent spiral of death.
		// If a frame takes longer than maxDeltaTime (e.g. breakpoint, window drag),
		// we just pretend less time passed. Physics will slow down, but it won't
		// try to simulate 10 seconds of physics in one frame.
		if (deltaTime > m_maxDeltaTime)
			deltaTime = m_maxDeltaTime;

		m_accumulator += deltaTime;
	}

	// Call in a while loop. Returns true if there is enough accumulated time
	// for one fixed step, and consumes that step's worth of time.
	bool step()
	{
		if (m_accumulator >= m_timestep)
		{
			m_accumulator -= m_timestep;
			return true;
		}
		return false;
	}

	// Returns the interpolation factor (0.0 to 1.0) representing how far
	// between the previous and current physics states we are.
	// Use this to interpolate positions for smooth rendering.
	float getAlpha() const
	{
		return m_accumulator / m_timestep;
	}

	// Getters for the configuration values.
	float getTimestep() const { return m_timestep; }
	float getMaxDeltaTime() const { return m_maxDeltaTime; }

private:
	float m_timestep;
	float m_maxDeltaTime;
	float m_accumulator;
	std::chrono::high_resolution_clock::time_point m_previousTime;
};
```

### Why This Design?

**Why a class and not free functions?** The accumulator and previous-time are state that must persist across frames. A class is the natural home for persistent state with a small interface.

**Why `explicit` on the constructor?** This prevents accidental implicit conversion from a `float` to a `FixedTimestep`. It is a good habit for any single-argument constructor.

**Why does `accumulate()` handle timing internally?** You *could* pass `deltaTime` in from outside, and there are valid reasons to do so (testability, custom time sources). But for 99% of cases, the class measuring its own wall-clock time is simpler and eliminates a category of bugs (forgetting to update the clock, passing the wrong delta). If you need a manual override later, you can add an overload:

```cpp
// Optional: manual accumulation for testing or replays
void accumulate(float deltaTime)
{
	if (deltaTime > m_maxDeltaTime)
		deltaTime = m_maxDeltaTime;

	m_accumulator += deltaTime;
}
```

**Why is `step()` a boolean and not a callback?** Keeping the loop in the caller's code means you can see exactly what runs at fixed rate. A callback-based API hides the loop body behind a `std::function` and makes debugging harder. Simplicity wins.

**What is the "spiral of death"?** If your physics step takes longer than the fixed timestep to compute, each frame adds more time to the accumulator than it drains. The accumulator grows without bound, and the game freezes as it tries to simulate an ever-increasing number of steps. The `maxDeltaTime` clamp prevents this -- physics will slow down instead of locking up. 0.25 seconds (4 FPS equivalent) is a common clamp value.

---

## 2. The PhysicsConfig Struct

### The Concept: Registry Context

EnTT's registry has a feature called **context variables** -- arbitrary data you can attach to the registry itself rather than to any entity. This is perfect for global configuration that many systems need to read.

```cpp
// Emplacing a context variable (do this once, during setup)
registry.ctx().emplace<PhysicsConfig>();

// Accessing it from any system (read-only)
const auto& config = registry.ctx().get<PhysicsConfig>();

// Accessing it for modification
auto& config = registry.ctx().get<PhysicsConfig>();
config.gravity = 4.9f; // low gravity level!
```

Context variables are:

- **Singleton by type** -- there is exactly one `PhysicsConfig` in the registry
- **Accessed by type** -- no string keys, no IDs, just `get<T>()`
- **Lifetime-managed by the registry** -- they are destroyed when the registry is destroyed

This is exactly what we want for physics parameters. They are global to the simulation, needed by multiple systems, and there should be exactly one set of them.

### The Code

```cpp
// engine/physics/physics_config.h
#pragma once

struct PhysicsConfig
{
	// Gravitational acceleration (positive = downward, applied as negative Y).
	// Earth-like default. Set lower for floaty platformers, higher for heavy feels.
	float gravity = 9.81f;

	// Ground friction multiplier, applied per fixed step to horizontal velocity.
	// 1.0 = no friction (ice), 0.0 = instant stop. Typical range: 0.8 - 0.95.
	float friction = 0.85f;

	// Air control multiplier. Scales how much horizontal input affects velocity
	// while airborne. 0.0 = no air control, 1.0 = full ground-like control.
	float airControl = 0.3f;

	// Maximum horizontal speed (units per second). Velocity is clamped to this
	// after all forces are applied. Prevents infinite acceleration.
	float maxSpeed = 8.0f;

	// Instantaneous vertical velocity applied when jumping (units per second).
	// This is set directly, not added -- you either jump or you don't.
	float jumpVelocity = 5.0f;

	// Fixed physics timestep. Stored here for convenience so systems that
	// need it don't have to receive it as a separate parameter.
	float fixedDeltaTime = 1.0f / 60.0f;
};
```

### Why This Design?

**Why a struct and not a class?** All members are public data with sensible defaults. There is no invariant to protect, no complex construction logic. A plain struct with aggregate initialisation is the right tool. Do not add complexity where none is needed.

**Why default member initialisers?** They serve double duty: they document the expected range of each value, and they let you construct a `PhysicsConfig` with zero arguments and get a playable game. You can override individual values:

```cpp
auto& config = registry.ctx().emplace<PhysicsConfig>();
config.gravity = 4.9f;    // low-gravity level
config.jumpVelocity = 7.0f; // higher jumps to compensate
```

**Why store `fixedDeltaTime` in PhysicsConfig?** Physics systems need the timestep to integrate velocities. Rather than passing it as a separate parameter to every system function, we put it where the rest of the physics parameters live. One fewer function argument, one fewer thing to get wrong.

**Why not `constexpr`?** These values will likely be loaded from a config file or tweaked at runtime (think debug sliders). Making them `constexpr` would prevent that. The defaults are compile-time constants in spirit, but the struct itself should be mutable.

### Updating Your Systems

Here is how a system changes. Before:

```cpp
void gravitySystem(entt::registry& registry, float dt)
{
	auto view = registry.view<Velocity, Gravity>();
	for (auto [entity, velocity, gravity] : view.each())
	{
		velocity.y -= 9.81f * dt;  // magic number!
	}
}
```

After:

```cpp
void gravitySystem(entt::registry& registry)
{
	const auto& config = registry.ctx().get<PhysicsConfig>();

	auto view = registry.view<Velocity, Gravity>();
	for (auto [entity, velocity, gravity] : view.each())
	{
		velocity.y -= config.gravity * config.fixedDeltaTime;
	}
}
```

Notice the system no longer takes `dt` as a parameter. It reads everything it needs from the registry context. This is a meaningful improvement: the system's signature now honestly reflects its dependencies (just the registry), and the physics configuration is always consistent across all systems because they all read the same struct.

Apply the same treatment to your movement, jump, and collision systems:

```cpp
void movementSystem(entt::registry& registry)
{
	const auto& config = registry.ctx().get<PhysicsConfig>();

	auto view = registry.view<Position, Velocity, OnGround>();
	for (auto [entity, position, velocity, onGround] : view.each())
	{
		float frictionFactor = onGround.grounded ? config.friction : 1.0f;
		float controlFactor = onGround.grounded ? 1.0f : config.airControl;

		velocity.x *= frictionFactor;

		// Clamp horizontal speed
		if (std::abs(velocity.x) > config.maxSpeed)
			velocity.x = config.maxSpeed * (velocity.x > 0.0f ? 1.0f : -1.0f);

		position.x += velocity.x * config.fixedDeltaTime;
		position.y += velocity.y * config.fixedDeltaTime;
	}
}

void jumpSystem(entt::registry& registry)
{
	const auto& config = registry.ctx().get<PhysicsConfig>();

	auto view = registry.view<Velocity, OnGround, JumpInput>();
	for (auto [entity, velocity, onGround, jump] : view.each())
	{
		if (jump.requested && onGround.grounded)
		{
			velocity.y = config.jumpVelocity;
			onGround.grounded = false;
		}
		jump.requested = false;
	}
}
```

---

## 3. The CollisionLayers Namespace

### The Concept: `constexpr` Bitmask Constants

Collision layers are a classic bitmask use case. Each layer is a single bit, and an entity's collision mask is the bitwise OR of all layers it interacts with. The raw hex values are fine for the computer, but terrible for humans.

```cpp
// What does this mean? You tell me.
collider.mask = 0x03;

// Much better.
collider.mask = CollisionLayers::Player | CollisionLayers::World;
```

### The Code

```cpp
// engine/physics/collision_layers.h
#pragma once

#include <cstdint>

namespace CollisionLayers
{
	// Each layer is a single bit. Use bitwise OR to combine.
	// An entity collides with another if (a.layer & b.mask) != 0.

	constexpr uint32_t None       = 0x00;
	constexpr uint32_t Player     = 0x01;
	constexpr uint32_t World      = 0x02;
	constexpr uint32_t Enemy      = 0x04;
	constexpr uint32_t Projectile = 0x08;
	constexpr uint32_t Trigger    = 0x10;

	// Common combined masks for convenience.
	constexpr uint32_t All        = 0xFFFFFFFF;
	constexpr uint32_t Solid      = Player | World | Enemy;
	constexpr uint32_t Shootable  = Enemy | World;
}
```

### Why This Design?

**Why a namespace and not an `enum class`?** An `enum class` would require a `static_cast` every time you do bitwise operations, because `enum class` deliberately prevents implicit conversion to integer. That makes the calling code ugly:

```cpp
// With enum class -- verbose and noisy
collider.mask = static_cast<uint32_t>(CollisionLayer::Player)
              | static_cast<uint32_t>(CollisionLayer::World);

// With namespace constants -- clean and readable
collider.mask = CollisionLayers::Player | CollisionLayers::World;
```

You *could* define operator overloads for the enum class, but that is a lot of boilerplate for something that `constexpr` constants in a namespace give you for free.

**Why `constexpr`?** These values are compile-time constants. `constexpr` tells the compiler they can be used in constant expressions (template arguments, array sizes, `switch` cases) and guarantees zero runtime cost. They will be folded into the code at compile time, just like `#define` macros, but with type safety and proper scoping.

**Why combined masks?** `Solid` and `Shootable` encode common collision rules that would otherwise be duplicated everywhere. If you add a new solid layer, you update `Solid` in one place.

### Using the Layers

When creating entities:

```cpp
// Player entity
auto& playerCollider = registry.emplace<BoxCollider>(player);
playerCollider.layer = CollisionLayers::Player;
playerCollider.mask  = CollisionLayers::World | CollisionLayers::Enemy | CollisionLayers::Trigger;

// World geometry
auto& wallCollider = registry.emplace<BoxCollider>(wall);
wallCollider.layer = CollisionLayers::World;
wallCollider.mask  = CollisionLayers::All; // everything collides with walls

// A trigger zone (non-solid, detects overlap only)
auto& triggerCollider = registry.emplace<BoxCollider>(doorTrigger);
triggerCollider.layer = CollisionLayers::Trigger;
triggerCollider.mask  = CollisionLayers::Player; // only triggered by player
```

In your collision system:

```cpp
// Before: what does this mean?
if ((a.layer & b.mask) && (a.layer != 0x10))

// After: self-documenting
if ((a.layer & b.mask) && (a.layer != CollisionLayers::Trigger))
```

---

## 4. System Phase Ordering

### The Problem

Right now, system execution order is implicit -- it is just the order you wrote the function calls. There is no indication of *why* that order matters, or what would break if you changed it.

We are not going to build a full scheduler here. That is a significant piece of architecture that we do not need yet, and premature abstraction is just as dangerous as premature optimisation. Instead, we are going to **document the phases** with clear comments and a simple enum.

### The Phase Enum

```cpp
// engine/core/system_phase.h
#pragma once

// Defines the conceptual phases of the game loop.
// Systems should run in this order. This enum exists for documentation
// and future use (e.g. a scheduler), not for runtime dispatch.
//
// Phase order:
//   1. Input       - Poll events, read input state, set intent components.
//                    Must run before anything reads input.
//
//   2. Physics     - Fixed timestep. Gravity, movement, collision detection
//                    and response. Runs 0-N times per frame inside the
//                    accumulator loop.
//
//   3. GameLogic   - Gameplay rules that respond to physics results.
//                    Health, scoring, state machines, AI decisions.
//                    Runs once per frame, after all physics steps.
//
//   4. LateUpdate  - Post-logic cleanup. Camera follow, animation blending,
//                    transform hierarchy propagation. Anything that must
//                    read the final state of other systems.
//
//   5. Render      - Read positions (interpolated by alpha), submit draw
//                    calls. Must be last.

enum class SystemPhase
{
	Input,
	Physics,
	GameLogic,
	LateUpdate,
	Render
};
```

### Why This Ordering?

This is not arbitrary. Each phase has a reason for its position:

**Input first** because every other system might read input. If you run physics before input, you are simulating with *last frame's* input -- one frame of latency that players can feel.

**Physics second** because it runs at fixed rate inside the accumulator and must produce a consistent simulation state before game logic reacts to it. If collision detection runs after game logic, your "is the player touching the goal?" check uses stale positions.

**GameLogic third** because it needs the results of physics (who collided with what, where is everything) to make decisions. This is where you check if the player reached the exit, if an enemy took damage, if a timer expired.

**LateUpdate fourth** because it needs the final game state. The camera should follow the player's *final* position this frame, not the position before game logic teleported them.

**Render last** because it is read-only. It should never modify game state. It reads positions, interpolates by alpha, and draws.

---

## The Solution: After

Now let us put it all together. Here is the refactored game loop:

```cpp
// main.cpp - after cleanup

#include "engine/core/window.h"
#include "engine/core/input_manager.h"
#include "engine/core/resource_manager.h"
#include "engine/core/mesh_factory.h"
#include "engine/core/fixed_timestep.h"
#include "engine/core/system_phase.h"
#include "engine/physics/physics_config.h"
#include "engine/physics/collision_layers.h"

// ... other includes ...

int main()
{
	Window window(1280, 720, "QEngine");
	InputManager input;
	input.init(window.getHandle());
	ResourceManager resources;

	// ... resource loading, mesh creation ...

	entt::registry registry;

	// --- Configuration ---
	auto& physicsConfig = registry.ctx().emplace<PhysicsConfig>();
	// Override defaults if desired:
	// physicsConfig.gravity = 15.0f;  // heavier feel
	// physicsConfig.airControl = 0.5f; // more air control

	// --- Entity creation ---
	setupScene(registry, /* ... */);
	auto& playerCollider = registry.emplace<BoxCollider>(player);
	playerCollider.layer = CollisionLayers::Player;
	playerCollider.mask  = CollisionLayers::World | CollisionLayers::Enemy | CollisionLayers::Trigger;

	// --- Game loop ---
	FixedTimestep fixedTimestep(physicsConfig.fixedDeltaTime);

	while (!window.shouldClose())
	{
		fixedTimestep.accumulate();

		// -- Phase: Input --
		window.pollEvents();
		input.update();
		inputSystem(registry);

		// -- Phase: Physics (fixed timestep) --
		while (fixedTimestep.step())
		{
			gravitySystem(registry);
			movementSystem(registry);
			collisionSystem(registry);
			jumpSystem(registry);
		}

		// -- Phase: GameLogic --
		// (future: scoring, health, AI, state machines)

		// -- Phase: LateUpdate --
		cameraFollowSystem(registry);

		// -- Phase: Render --
		renderSystem(registry, fixedTimestep.getAlpha());

		window.swapBuffers();
	}

	// ... cleanup ...
}
```

Compare this with the "before" version. The game loop is now:

- **Shorter** -- the accumulator boilerplate is gone, replaced by three method calls
- **Self-documenting** -- phase comments make the execution order explicit
- **Magic-number-free** -- all physics constants live in `PhysicsConfig`, all collision layers have names
- **Consistent** -- every physics system reads the same config struct, so gravity cannot disagree with itself

And we did not add any frameworks, any virtual dispatch, any complex abstractions. Just a small class, a struct, a namespace, and some comments.

---

## Project Structure

After this cleanup, your project tree should look like this (showing only the files we added or modified):

```
engine/
├── core/
│   ├── fixed_timestep.h      ← NEW: accumulator pattern
│   └── system_phase.h        ← NEW: phase ordering documentation
├── physics/
│   ├── physics_config.h      ← NEW: centralised physics parameters
│   ├── collision_layers.h    ← NEW: named bitmask constants
│   ├── gravity_system.h      ← MODIFIED: reads PhysicsConfig from context
│   ├── movement_system.h     ← MODIFIED: reads PhysicsConfig from context
│   ├── collision_system.h    ← MODIFIED: uses CollisionLayers namespace
│   └── jump_system.h         ← MODIFIED: reads PhysicsConfig from context
└── ...
main.cpp                       ← MODIFIED: uses FixedTimestep, phase comments
```

All four new files are header-only. No `.cpp` files, no additional compilation units, no build system changes. Include and go.

---

## Exercises

1. **Add a debug overlay** that displays the current `PhysicsConfig` values on screen. Bonus: make them adjustable with keyboard shortcuts so you can tweak physics feel at runtime.

2. **Add a `CollisionLayers::Pickup` layer** (value `0x20`). Create a pickup entity that only the player can collide with, and add it to a `Collectible` combined mask.

3. **Add the manual `accumulate(float deltaTime)` overload** to `FixedTimestep`. Write a test that accumulates exactly 3 timesteps worth of time and verifies that `step()` returns `true` exactly 3 times, then `false`.

4. **Create a "moon gravity" power-up** that halves `PhysicsConfig::gravity` for 5 seconds. Think about where the timer logic belongs (hint: Phase 3, GameLogic).

---

## Key Takeaways

- **The accumulator pattern is always the same.** Extract it once, use it forever. The `FixedTimestep` class is maybe 40 lines and eliminates a whole class of copy-paste bugs.

- **Registry context is your friend for global config.** `registry.ctx().emplace<T>()` gives you a typed singleton attached to the registry's lifetime. No globals, no singletons, no service locators -- just a struct that lives where your entities live.

- **Named constants are not optional.** `CollisionLayers::Player | CollisionLayers::World` is code that explains itself. `0x03` is code that demands a comment, and comments go stale.

- **Document your system order even if you do not enforce it.** A `// -- Phase: Physics --` comment costs nothing and saves real debugging time. When you are ready for a proper scheduler (probably around Chapter 15 or so), the phases are already defined.

- **Cleanup chapters are features.** Readable, maintainable code is not a luxury. It is what lets you keep building. Every chapter from here on benefits from the work we did today.

---

*Next up: **Chapter 11 -- Doors, Lifts & Triggers**, where we will build interactive level elements and find out why our new phase ordering makes it trivial to decide where trigger logic belongs.*
