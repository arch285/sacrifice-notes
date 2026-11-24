# Squall pushback: original vs sacengine

Values come from the original Sacrifice.exe, Spells.wad and Fsm.wad. SacrificeModern (andrei-cb) was used as a guide. I compared against sacengine 62c1b23.

## Spell data (`abma` SPEL, sacengine `amba` / SpellTag.squallShoot)

- amount 520, damage type 67, speed 1.0/tick, range 150, proc `GenericShootStart`
- parameters: `thit` = `push`, `fpsh` = 8.0
- This is the only SPEL in Spells.wad that has a `push` hit or an `fpsh` parameter.

## Original

- GenericShoot hit handler (0x411bb0, `push` branch at 0x411e82): if the projectile hit an object, it enters stage 4 with a 30-tick counter (0.5 s). A hit on the terrain does nothing.
- Stage 4 (0x412116) runs once per tick: `0x44dd30(target, projectileVelocity, fpsh*0.05, axes=3)`. `fpsh` defaults to 2.0 and 0.05 is the constant at 0x4bf880, so for Squall the amount is 8*0.05 = 0.4/tick (24/s). After 30 ticks, or as soon as the target is gone, the effect ends.
- 0x44dd30 (`push_direction` in SacrificeModern):
  - Does nothing if the target's motion flags contain 0x10000.
  - Uses only the horizontal direction (axes x,y), normalized, so the projectile's pitch does not matter.
  - For each pushed axis: `v = v + (amount*dir - v) * 0.5`. The push blends the velocity toward 24/s along the shot direction. It does not add to the velocity, so a faster velocity or one in a different direction (for example from tumbling) is overridden.
- Motion flag 0x10000 on creatures comes from FSM state flags 0x10/0x80 (norm.FSMC). Those states are teleport ability, undead, all respawn states, vivify, and being lifted, carried or sacrificed by a sac doctor. Every other state can be pushed, including DAMG, FALL, PARA and DETH main.
- Creature motion has the 0x40 drag (velocity *0.99 per tick). On the ground it also has 0.02/tick friction (motion_damp). Walking uses local velocity, so it does not cancel the push.
- Simulated result:
  - on the ground: 10.9 units after the 30 ticks, at about 21/s. The creature then brakes and stops after about 0.78 s, for a **total of about 13.9 units**.
  - flying: there is no ground friction, only the 0.99 drag. The model gives about 22 units after 1 s, about 35 after 2 s, and **about 50 units total**. I have not checked this in game.

## sacengine (state.d: `Pushback` ~5443, `pushback` ~13076, `updatePushback` ~26052)

- Constant 22.5/s for 40 frames (2/3 s), applied directly to the position, for exactly 15 units. Speed does not ramp up or down.
- The direction is the full 3D projectile direction. On the ground z is snapped to the terrain, so a shot from above or below pushes less horizontally.
- Tumbling: a one-time `fallingVelocity += 22.5*direction`, then the effect ends.
- Flying: handled the same as on the ground (15 units).
- No immunity apart from `effects.fixed`.

## Differences

1. Ground distance: about 13.9 units with a smooth ramp and braking, against 15 units at a constant speed.
2. Duration: 0.5 s of pushing plus about 0.3 s of braking, against 0.67 s.
3. Direction: the original pushes horizontally at full strength whatever the pitch; sacengine uses the 3D direction.
4. Tumbling: the original steers the horizontal velocity toward 24/s for 30 ticks; sacengine adds an impulse once.
5. Immunity: the original skips teleporting, undead, respawning and vivify creatures and creatures handled by a sac doctor; sacengine pushes them.
6. Flying creatures: about 50 units in the original model against 15 in sacengine. This is the biggest gap, but it still needs an in-game check.

## Possible change (if you want it)

Store a horizontal velocity on the creature (or in `Pushback`) and, for 30 frames, apply `v += (24*dirXY - v)*0.5`. Every frame, move by `v/60`, multiply by 0.99 and, on the ground, subtract 0.02*60/s of speed until it reaches 0. While tumbling, apply the same blend to the xy part of `fallingVelocity`. Skip creatures whose mode matches the FSM states with flags 0x10/0x80.
