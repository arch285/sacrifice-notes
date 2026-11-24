# Creature collision push: sacengine vs original (2026-11-24, sacengine 62c1b23, SacrificeModern 24fc0e5)

Goal: match the original. This is a separate mechanism from the Squall pushback.

## Original (verified in Sacrifice.exe)
- Motion integrator, 0x469b3e–0x469c2a: the collision call (vtable +0xc0) returns the object the movement hit (stored at +0x250), and state |= 1. If the collision mask (+0x24c) has 0x40000000 (creatures: 0xc0004015) and the movement fraction to contact == 0.0 (0x4bf624), it calls push_from(self, hit object's position +0x1b8, 0x3daaaaab = 1/12, axes 3). Then delta += velocity change · ticks, and a second collision pass runs that ignores the hit object.
- Fraction 0 means that at the start of the tick the creature already overlaps the other one, or touches it and keeps moving into it. Simply walking up to an obstacle (fraction > 0) only clips the movement (creatures slide along the face, mask 0x80000000).
- push_from 0x44dc00: does nothing if motion flag 0x10000 is set (respawning, undead, teleport, Vivify, being carried/sacrificed by the sac doctor, i.e. FSM states with flag 0x10 or 0x80). Direction: from the other object's position (origin) to the center of the pushed object's own boxes (0x44e020), x and y only, normalized. Velocity: v = v + (target − v)·0.5 (0x4bf904), target = 1/12 units/tick = 5 units/s.
- Only the moving (currently updated) object is pushed. The other creature gets nothing unless it moves while in contact itself.
- The push is a velocity, so it has inertia. On the ground: friction of 0.02 units/tick plus ×0.99/tick (from SacrificeModern motion.cpp, motion_damp 0x469700).
- Simulation: a single push event on the ground moves the creature ~0.065 units and dies out after 2 ticks (a slight bump). Continuous contact on the ground: ~3.8 units/s. Flying creature (no ground friction): a single event gives ~4.2 units of drift (1.9 units after 1 s), continuous contact 5 units/s. Flying not verified in game.

## Sacengine (state.d positionAfterCollision ~16313, stats.d collisionFixupSpeed = 5)
- Two phases. Blocking: hitbox shrunk by 0.1 on each side; on overlap, movement along the axis of smallest overlap is reverted to the old position (allows ~0.1 units of penetration).
- Fixup: hitbox grown by 0.1; for each obstacle, overlap along the axis of smallest overlap (x/y, also z when flying), summed, capped at 5 units/s (0.083/frame), applied directly to the position (moveOnGround on the ground, if the position is on the ground). No velocity and no inertia; it stops exactly at a 0.1 gap.
- Active in isReactingToObstacles modes.

## Differences
1. Mechanism: the original changes velocity (inertia + friction), sacengine moves the position.
2. Direction: the original pushes radially away from the other creature's position (horizontal only), so creatures slide around each other; sacengine pushes along one world axis (the face with the smallest overlap).
3. Strength: the 5 units/s number is the same, but in the original it is a target that velocity moves halfway towards per tick, and friction eats most of it: a single bump is ~0.065 units.
4. Trigger: the original only pushes on contact at the start of the tick (overlapping or pressing into the other creature); creatures standing next to each other do not repel. Sacengine already pushes at a gap below 0.1.
5. Flying: the original drifts (no friction), sacengine behaves the same as on the ground, plus the z axis.
6. Immunity: the original uses motion flag 0x10000; sacengine a list of modes.

## Not verified
- Flying creature drift in game.
- The exact set of objects a creature's collision returns (whether buildings also trigger the push; mask 0x15 suggests they do).
