# Grasping Vines and Vinewall: original vs sacengine (findings from the exe, 2026-11-24, sacengine 62c1b23)

## Data (Spells.wad, Slif.SCAT)
- `vine.SPEL`: mana 500, casting time 100 (min 50), range 100, duration 40 s, exclusion tags PARA/PROT, ttex `vine`, tdrt `dirt`, proc `GraspingVinesRun`.
- `vwal.SPEL`: mana 1500, casting time 300 (min 150), duration 50 s, effectRange 55, fwid 5.0, fhgt 7.5, proc `VineWallRun`. The exe reads the height with the tag `fHei` (not `fhgt`), so it always uses the default 7.5 (same result).

## Procedure table (correction)
- In the name table, the function pointer comes AFTER the name (name + 0x1c), not before it. Hence: GraspingVinesRun = 0x42c2f0, VinesHoldRun = 0x42c090, VineWallRun = 0x42eda0, VineWallEarthEffect = 0x42e990, AirClawpultStart = 0x418aa0, VacuumEffectRun = 0x4187f0. ritual-sacdoc.md mislabels 0x42c2f0 as VinesHoldRun.

## Original: Grasping Vines
- GraspingVinesRun 0x42c2f0: stage 0 sound + dirt spray (0x436df0) + casting hands; stage 1 waits for the cast to finish; stage 16: height limit 5.0, duration = duration·60, 8 vines, fade (0+1)·60 = 60 ticks, switches to VinesHoldRun.
- Setup 0x42bd10: spell position = target position, z = terrain height (terrain ≤ 0 → spell removed). Offset = target position − spell position (including the vertical offset). If the target's feet are more than 5.0 above the terrain, the target is dropped (no hold; the visual effect still plays).
- Duration 0x4215b0: adds the name PARA; creature: duration × min(1, 1000/maxHP) (word +0x8fe); wizard: × 0.25. Truncated to int. The same function is called by Freeze (20 s), Lethargy (30 s), Vines (40 s) and Turn to Stone. This is the "40/30/20 s cap".
- VinesHoldRun 0x42c090 stage 1, every tick: target position = spell position + offset, flag +0x234 |= 0x80000. The spell's z = terrain height (0x44ecb0), so the vines move with the terrain (e.g. the Erupt mound). It does NOT zero the target's velocity and sends no FSM events.
- Flag 0x80000: movement controller 0x45a8b0 (`controlled = flags&1 && !0x80000`: no steering, walking, turning or flight control) and approach_order 0x45b620 (orders do not steer). Cleared at the end of ObjectMotion::advance (0x469dde). The motion integrator does not check it.
- End: stage 240 removes PARA, stage 241 waits for the fade, then the object disappears.

## Original: object order within a tick (verified 2026-11-24)
- Loop 0x453940: from the head of the active list [0x4dd1a4], then via +0x3c4; every object (without the removed flag 0x10000) gets run = vfunc +0xb0(ticks).
- Insertion 0x493410(1, after), called from activation 0x4681f0: without `after` → at the head of the list; with `after` → right after it. Creation 0x467d50(type, after) passes `after` on to activation.
- Who passes `after`: 0x44e850 (spell object creation) uses its 1st argument, i.e. the creating object. The Vinewall hold (0x4378b0) → right after the wall. A summoned creature (0x481900 → 0x458510) → right after the casting wizard. Map objects (0x458300, 0x4a41c3 with after = 0) → at the head. The Grasping Vines object is a child of the caster (per Andrei, 0x44e740; I did not trace the call from casting all the way).
- Consequence: newer children sit right after their parent, before older ones. A creature's movement (controller 0x45a8b0 and ObjectMotion::advance 0x469760, call at 0x45b288) is computed in its own run.
- Hold (S, after wizard A) vs target (T, after its own wizard B, or wizard B itself): S runs before T if and only if A is before B in the list. This is fixed for a given game but different for the two players. For the caster's own creature, S always runs before T.
- S before T: after being pinned, T still moves by its velocity for that tick and is drawn there (Squall ≤ 0.4 u, Flurry ≤ 0.15 u, gravity ≤ 0.27 u); the next tick it is pinned again. S after T: drawn exactly at the vines' position. In both cases there is no drift, and flag 0x80000 works (set in the same tick or the previous one, cleared at the end of advance).
- Sacengine: effects always run before creatures (like "S before T"), but a creature with `fixed` does not move at all, so there is no one-tick displacement.

## Original: Vinewall
- VineWallRun 0x42eda0, stages: 0 setup (0x42ea60), 1 casting, 16 after casting, 240 fade, 241 end. Setup: ends at 0/0, growth flags 3 (both ends), max length effectRange 55 per side, width fwid 5, height 7.5, timer 50 s.
- Growth (verified): 0x42ec10 runs every tick in stages 1 and 16 (also while casting) and calls 0x427cd0 with a step of 0.25 u/tick per end (15 u/s), up to ±55 from the center (full length after 220 ticks = 3.67 s). An end stops permanently when the terrain is ≤ 0 (hole/void) or when the 20 u grid cell (0x4dd600) contains a building whose footprint the end enters (0x427be0 → 0x491fc0, mask 2). Visual vines: for each growing end with probability 1/4 per tick (rand & 3 == 0), at the end point ±0.5 (0x42eb30). The opacity parameter [+4] rises by 1/60 per tick up to 1 (1 s); during the fade it drops by 0.01 per tick (1.67 s).
- 0x427960(wall, left, right, 20, 8) → 0x4702f0 on 0x4d5f00: the wall segment is reported to the AI danger/avoidance map; not part of the grabbing mechanic.
- Grabbing (0x427b20 → 0x4279f0 → 0x42ed10) only in stage ≥ 16, i.e. after the cast has finished. The 50 s timer also counts from the end of casting.
- Field +0x304 (verified): a common field of networked objects. Set on activation (0x468316 to −1, then 0x499f00): without network −1; with network the result of an owner query (word +0x308); > 0 → local object (added to the list at +0x2f0), 0 → copy from another machine. So the wall only grabs on its owner's machine; the grab reaches other players as a normal networked object creation. Always true in single player; sacengine (deterministic lockstep) has no equivalent and does not need one. The same field is checked by approach_order 0x45b634 and many other procedures.
- Position test: the unit's box (in the wall's frame) overlaps the band ±fwid/2 = ±2.5 between the ends, and vertically the box overlaps [terrain, terrain + 7.5].
- 0x42ed10: skips allies (vfunc +0x70), units named PARA and units named `vwal` (the spell's name). Does NOT check PROT (shields).
- Duration: max(remaining wall time, 10 s), then in the setup × min(1, 1000/maxHP), or × 0.25 for a wizard.
- 0x4378b0 directly creates a VinesHoldRun object: 8 vines, height limit = 7.5 (wall field +0x5c), fade (5+1)·60 = 360 ticks = 6 s. The name `vwal` remains until the object disappears (hold + 6 s), so the same wall can only grab again 6 s after release.

## Original: Flurry (AirClawpult)
- Hit: damage, then a VacuumEffectRun 0x4187f0 object for 90 ticks (1.5 s). Every tick, everyone (mask 5) within a radius growing up to effectRange 20 gets push_direction (0x44dd30) toward the center, 0.15 u/tick, axes xyz.
- On a vined creature this works like any push: it changes the velocity, but the position is overwritten every tick. In the original, neither Flurry nor Squall moves a vined creature; at most a one-tick displacement by its velocity when the hold runs before the target (see object order).

## Sacengine (state.d)
- graspingVines ~12191: duration `(wizard ? 0.25 : 1000/maxHealth)·duration` WITHOUT min(1, …) → creatures with < 1000 HP are held longer than 40 s (e.g. 500 HP → 80 s). ceil instead of truncation.
- Target rejection: isHitboxFlying (hitbox bottom ≥ terrain + 3 or not on ground, "TODO: measure"). Original: feet > terrain + 5.
- Hold: updateGraspingVines overwrites the position every frame; `fixed` blocks walking, push(), catapult (no velocity), tornado.
- Effects are updated before creatures, and the hold before pushbacks (29309) and Flurry (29393).
- updateFlurryImplosion writes `obj.position` and `fallingVelocity` without checking `fixed`.
- updatePushback (tumbling): `fallingVelocity += vel` without checking `fixed`.
- updateVortexEffect: catapult (fixed → tumbling) and then `fallingVelocity += …`, z ≥ 1 and anti-gravity, without checking `fixed`. Meanstalk (throwing) sets `fallingVelocity` directly.
- updateCreaturePosition, tumbling: gravity and damping are applied BEFORE `if(fixed) break`, so a vined creature in the air accumulates downward velocity (limit ~250 u/s) that it never uses. At the end of the function: fixed and z > terrain with onGround movement → tumbling with velocity 0.
- Landing in the stunned state (~16028): tumbling, v.z ≤ 0 and z + v.z/60 ≤ terrain → onGround + dealFallDamage. With a vined creature held above the terrain this can repeat (tumbling → accumulated velocity → "landing" with damage → tumbling again). Hypothesis, not verified by running it.
- A normal teleport does not reset fallingVelocity (only the wizardVoid branch on the ground does).
- startFlying (~9747) and land do not check `fixed` → a vined creature can start taking off.
- Teleport under vines: graspingVinesOffset is only used in predictCenter, getShotTargetPosition, melee attacks and Rock. Other targeted spells (Fireball, Lightning, Wrath, Swarm, Dragonfire, Chain Lightning, Intestinal Vaporization, Charm, Blind Rage, Bombardment, Rhinok projectiles, Breath of Life, etc.) use the real position.
- Vinewall: grabs after casting ends (matches). Grabs when the center distance < 1.75 (0.35·5); original: the box overlaps ±2.5. Skips ccProtected (shields); original only PARA. Duration min(remaining, remaining·1000/HP); original max(remaining, 10)·min(1, 1000/HP). 6 s cooldown per wall (original: 6 s fade, name shared by all walls). No 7.5 height limit.
- Vinewall growth: expansionSpeed = 55/castingLimit = 55/240 ≈ 0.229 u/frame (original 0.25 u/tick, full 55 after 3.67 s instead of 4 s). Stop conditions as in the original (building or no ground, ≤ effectRange from the center). Grows only in VinewallCasting and then in the growing state; wall duration counted from the start of casting (castingLimit 4 s + 50 s), original: 50 s from the end of casting (casting takes 2.5–5 s depending on level). Height `top` grows over 1 s (original: opacity over 1 s, the grab test always uses 7.5). Fade 1 s (original 1.67 s). 180 vines spread evenly over 4 s (original: random, 1/4 per tick per growing end).

## Video "vine_squall.mp4" (sacengine, 12.4 s, 2026-11-24)
- 0–3.6 s: the player's wizard is vined, 3 enemy Squalls shoot from a hill. HP (bar in the top-right corner, red rows) almost unchanged: 88 → 86.
- 3.7 s: teleport to the altar (player and manahoars). In the same frame HP drops 87 → 52 (~40% of the bar) with no visible hit.
- Then in steps: 4.7 s 52 → 43, 5.2 s 43 → 11, 5.7–6.6 s small drops down to 0. At 4.7 and 5.2 dark streaks cross the frame (dusty projectiles?), source not visible.
- ~6.7 s HP = 0, HUD disappears. From ~7.0 s the camera follows a small figure (the lying wizard) flying very fast across the map, over the chasm, up to a tree; then the ghost (sparkles).
- Interpretation: the damage at the moment of teleport fits a "landing" with velocity accumulated under the vines (teleport does not reset it), and the flight after death fits the release of the accumulated velocity when the vines let go (dying → canPersistCC = false). The original does not accumulate anything beyond the object's normal velocity (the Squall push converges to 0.4 u/tick and does not add up; gravity with ×0.99 damping gives at most ~0.27 u/tick).
- To confirm: a recording (.rcp) of this game → replay it and print the wizard's fallingVelocity/movement/mode every frame.

## Conclusions for the Trello cards
1. Cap: missing min(1, 1000/HP) in graspingVines (and check the same formula in freeze/lethargy).
2. Takeoff/landing: block startFlying/land when `fixed`.
3. Pushing upwards and Flurry: Flurry, Pushback (tumbling), Vortex and Meanstalk bypass `fixed`.
4. Fall damage: in the original the vines overwrite the position and move with the terrain, so Erupt on a vined creature should not cause fall damage (conclusion from the code; to test in-game: Erupt on a vined hoar).
5. Vine+squall: see the video section. Fix direction: when `fixed`, do not accumulate fallingVelocity (do not add pushback, do not apply gravity, do not toggle tumbling/onGround in a loop), and reset it on teleport under vines.
6. Vinewall: count the wall duration from the end of casting; growth step 0.25 u/tick, independent of casting time.

## Not verified
- The call that creates the Grasping Vines object when casting (that the caster is its parent) – taken from Andrei (0x44e740), not traced in the exe.
- Which side comes first in the list (load order of wizards from the map).
- Whether the names (PARA, vwal) disappear together with the hold object (assumed).
- The mechanism from the video has not been confirmed by running sacengine.
