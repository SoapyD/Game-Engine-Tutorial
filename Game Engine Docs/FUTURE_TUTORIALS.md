# Future Tutorials

Tutorials that need to be written but don't yet exist. These cover concepts introduced illustratively in existing chapters that were deferred for later implementation.

---

## Advanced Movement

**Source**: Chapter 10 (Physics & Movement) — stair stepping section marked as "Concept — Future Chapter"

**What it covers**:
- Stair stepping — gliding up small height changes without jumping
- Uses swept AABB tests to try "move up, move forward, move back down"
- Requires a walking player entity with collision

**Prerequisites**:
- Player entity with Position, Velocity, AABBCollider, OnGround
- Swept AABB collision (Chapter 9)
- Ground detection system (Chapter 10)

**Why it was deferred**: The implementation requires multiple swept AABB checks in sequence and a player entity that walks around. Chapter 10 only has a fly camera, so there's nothing to walk up stairs with yet.

---
