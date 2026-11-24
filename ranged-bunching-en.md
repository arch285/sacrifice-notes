# Ranged creatures bunching up: original vs sacengine (exe, 2026-11-24, sacengine 62c1b23, SacrificeModern d0f826b)

Problem: in sacengine, ranged creatures bunch up too much. The original is the reference.

## Original (checked in Sacrifice.exe)
### Shot execution (order 66, 0x46e3d2–0x46e534; SacrificeModern `execute_spell_order`)
1. `range = SPEL.range · 0.75` (0x4bf8a0 = 0.75). Distance = hitbox gap (0x4027c0) to the target.
2. If the gap > 0.75·range **or terrain blocks the line** (0x48ec70 & 2) → approach the target (0x45b620, radius 0.5), end of tick.
3. Turn to the target: if the angle > 5° (0x4c0680 = 0.0873 rad) → turn only, end of tick.
4. Object test on the line of fire (0x48edb0 → code of the first object hit):
   - 4 = nothing, 8 = object with kind flag 0x10, 0x10 = ally (vfunc +0x70), 0x20 = other (enemy that is not the target).
   - Execution checks **only 0x10**: an ally on the line → `return 0` with no movement at all. Controls are zeroed every tick, so the creature **stands and waits**. It does not approach and does not walk around.
   - An enemy or an object of type 8 on the line does not block: the creature shoots (the projectile hits whatever is in the way).
5. Then cast_check and the shot.

### Target selection (0x48ef50, called from 0x494d60 and, for order 6, via 0x495430)
- For spells with flag 0x30000: `bits = terrain (0x48ec70)`; if `bits & 0x3e == 0` → `bits |= objects (0x48ed80)`; if `bits & 0x3a` → target **rejected** (terrain, type-8 object, ally, enemy that is not the target).
- Each think (at least every 17 ticks) discards the temporary order. A creature whose line to its target is blocked picks another target with a clear line at the next think; if there is none (melee disabled by mana), it picks nothing.
- When nothing is picked, the base order runs: with no order the creature stands; with order 7 (attack-move) it keeps walking towards the order's point, not towards the target.

### Order 6 (explicit attack) with a blocked line (checked in the exe)
- Target selection, order 6 branch (0x493fb7): sets reach 10 (0x41200000), range 150, threshold 0. For a creature with +0xb34 & 0x18 (ranged/support) it **scores only abilities** on the ordered target (0x4940da–0x4941f7); melee on that target is not scored (that is a separate branch, 0x494210, for other creatures). Then `and al,0xf3` (no scan of other enemies).
- For each ability: skips flag 0x800000; `d² ≤ (10 + SPEL.range)²` (positions); cast_check ≤ 1; then 0x495430 with context flags 0x380.
- 0x495430 calls 0x48ef50. Flag 0x100 skips only the evaluator (returns 1.0, 0x4bf628); flag 0x200 skips the SPEL range test inside 0x48ef50. **The line-of-fire test (mask 0x3a) stays.** Blocked line → the ability is not picked.
- After 0x49445f: a creature with mana does not score melee (0x494466), so nothing is picked and there is no temporary order 66.
- The base order 6 then runs (0x46e0b3–0x46e1c9). It is the same code as for melee creatures, with no separate branch for ranged ones:
  - if the position distance > sum of radii (+0x2e4) → approach the target's position (0x45b620, radius 0.1 = 0x3dcccccd);
  - in contact: turn (threshold 0x4bfc58), `melee_target(0.25, mask 0x15)` must return the ordered target, then FSM event 4 (melee blow).
- Result: under an explicit attack, a ranged creature with a blocked line **walks towards the target**. The line is rechecked at every think (≥ 17 ticks); once it clears and the target is within 10 + range, the shot is picked (order 66) and the creature stops to shoot (0.75·range). If it reaches contact before the line clears, it attacks in melee.
- Whether every ranged creature's FSM handles event 4 depends on the game data; I don't have that information.

### Overall result
- With no order, when guarding, and under attack-move, the back row of shooters behind their own creatures does not push towards the target. Whoever has a clear line shoots; the rest stand (or follow the base order) until the line clears or another target comes into range.
- Only an explicit attack on a target makes a blocked shooter walk towards the target.
- Creatures beyond 0.75·range, or with the line blocked by terrain, also approach.
- In addition (aggression-ranges.md): an idle ranged creature takes any enemy within SPEL range and does not chase it out of range beyond the next think, so a group shoots from a distance instead of converging to ~74 units.

## Sacengine
- `aim` (state.d ~14019): when `!hasClearShot` → `moveTowards(targetPosition, 0.0f, ...)`, i.e. walks straight at the target with radius 0 until the line clears. Same path for automatically picked targets and for explicit attacks (`attack` ~14320 → `aim`).
- `hasLineOfSightTo` (~31089): blocked by any projectile obstacle other than the shooter and the target: allies, enemies, buildings.
- `hasClearShot` is used only in `aim`; target selection (`updateTarget`) does not check the line of fire, so a target with a blocked line is not changed.
- In addition: ranged creatures pick targets like melee ones (~74 units) and chase without limit (aggression-ranges.md), and the collision push moves the position instead of the velocity (creature-collision-push.md).
- Effect: every shooter behind an ally walks towards the target, runs into the front row and compresses it. The bigger the group, the worse.

## Differences (causes of bunching, most important first)
1. Ally on the line, automatically picked target (idle, guard, attack-move): the original stands and waits or follows the base order; sacengine walks at the target.
2. Enemy/building on the line: the original shoots (in shot execution); sacengine walks at the target.
3. Target selection: the original rejects targets with a blocked line (switches target or picks nothing); sacengine keeps the target.
4. Target selection range and chasing (see aggression-ranges.md).
5. Explicit attack: both walk towards the target. The original checks the line only at thinks (≥ 17 ticks) and approaches to contact (radius 0.1, then a melee blow); sacengine checks the line every frame. The difference in bunching is small here.

## Proposed fix in sacengine
- `aim` for automatically picked targets: separate the reason the line is blocked. Terrain → approach (as now). Ally first on the line → `stop()` and turn, no movement. Enemy/building → shoot.
- Explicit attack: keep the approach towards the target (matches the original).
- `updateTarget` for ranged creatures with mana: reject targets with a blocked line (terrain, ally, non-target enemy, building), so at the next think the creature switches target or returns to its base order.

## Not checked
- Hit order in 0x48edb0 (whether it is always the object nearest to the shooter). According to SacrificeModern, objects closer than 10 units have their box shrunk by 0.1 − d²·0.001.
- Handling of FSM event 4 (melee blow) by individual ranged creatures (game data).
- Nothing tested in a running game.
