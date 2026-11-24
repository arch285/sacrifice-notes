# Melee combat: SacrificeModern vs sacengine vs the original (findings, 2026-11-19 to 2026-11-21)

## Sources
- **ANDREI** = `andrei-cb/SacrificeModern` (C++20, Windows), commit 2691e10 (2026-11-15). Melee code: `engine/creature_melee.cpp`, `engine/spatial.cpp`, `engine/creature_combat.cpp`, `engine/creature_actor.cpp`, `engine/creature_attack.cpp`, `engine/creature_order_run.cpp`, `engine/player.cpp`.
- **TG** = `tg-2/sacengine` (D, multi-platform, multiplayer), commit 8fe0753 (2026-11-11). Melee code: `source/state.d`, `source/sacobject.d`.
- **Original**: `Sacrifice.exe` (884,736 bytes, Polish install with Patch 3). All function addresses from ANDREI's comments (0x493b70, 0x4596e0, 0x46a940, 0x494fd0, 0x4951b0, 0x46da50, 0x45bd20, 0x45be60) land on function prologues in this exe, so it is the same build ANDREI ported from.

## How the two engines work
- **ANDREI** ports functions from the original exe instruction by instruction, with binary addresses in comments (e.g. `melee_hit` = 0x4596e0, target sweep = 0x46a940). It reproduces fixed-point arithmetic and integer overflow quirks.
- **TG** reconstructs behaviour from observation, in floating point. The melee hitbox is marked by the author as a guess: *"TODO: this is a guess. what does the game actually do?"*
- Both arrived at the same hit and damage formula.

## The blow, step by step

| Step | ANDREI | TG |
|---|---|---|
| Starting the attack | The state script from the game data calls `CreatureChooseAttackAnim`, then `CreatureDoAttack` | `startMeleeAttacking` sets mode `meleeMoving`, then `meleeAttacking` on the attack-tick frame |
| Choosing the animation | 50/50 `atk1`/`atk2`; `atkl` if the target is clearly lower (`dz·5 < −horizontal distance`) | Random `attack0`/`attack1`; `attack2` if the target is 0.25 below the creature's centre; `flyAttack` in the air. Fixes an original bug for must-fly creatures |
| Finding the target | Segment sweep from the creature's centre forward: length = 1 + half the body, width = ¼ of the box diagonal, heights must overlap. Takes the **nearest** object (mask 0x15), allies included | Axis-aligned hitbox in front: offset 1.3× size, height ×1.5. Hand-tuned exceptions: Netherfiend ×3, Boulderdash ×2, Phoenix/Hellmouth ×1.5, Druid and peasant wider. Picks by stance rank (**enemy** first), then distance |
| No target in reach | If the FSM state allows it, strikes the designated target anyway | No hit |
| Ally in reach | Hit sound plays, 0 damage (unless it is the designated target) | Enemies take priority, so an ally is rarely chosen |

## Damage formula (identical in both)
- **Attack time** = animation length ÷ number of `atck` ticks in the animation. ANDREI: computed once at load (`attack_seconds`). TG: per blow (`pendingDamage`).
- **Facing multiplier** = `1.5 + 0.5 · dot(attacker facing, target facing)`. Hit in the back ×2, face to face ×1.
- **Hit chance** = accuracy ÷ (accuracy + evasion ÷ facing).
- **A miss is not zero**: damage is scaled by `(1 − roll) / (1 − p)`.
- **Melee resistance** is multiplied by 0.4.

| | ANDREI | TG |
|---|---|---|
| Numbers | Integers, ×256, 15-bit random draw | Floating point |
| Base damage | `melee_damage` from creature data; wizard hard-coded 120; damage type from data | `meleeStrength` from `cre8` |
| Bonuses (Lightning Charge, Firefist, Guardian, slime) | Generic, via stats and damage watchers (e.g. Firefist is a separate effect changing the multiplier) | Hard-coded in `attackDamageFactor` / `dealMeleeDamage`: Guardian ×1.5, Lightning ×1.5 (damage and accuracy), Firefist up to ×2.5 at low HP, slime ×0.2^n |
| Target protections | Generic resistance multiplier by damage type | Explicit: Life Shield ×0.5, Rock Form ×0.05, Petrify ×0.2 |
| Buildings | Same function, no facing, always a full hit | Separate function: no facing or accuracy, `hitWall` sound |

## Reaction of the struck creature
- **Stun**
  - ANDREI: FSM event 7 ("heavy hit") when damage is untyped and above ¼ of max HP, otherwise event 6 (flinch). Blood count depends on damage.
  - TG: stun when damage ≥ ¼ of max HP or the creature is "stunned by every blow"; then 4 s stun immunity (marked TODO).
- **On-hit specials (Freeze, Grasping Vines)**: ANDREI as separate creature abilities (`spell_melee.cpp`); TG inline in the blow code (Freeze with cooldown, Grasping Vines).
- **Life steal and damage credit**: ANDREI in `damage_dealt` (stat 7, ritual altar ×1.5); TG in `healFromDrain` and kill statistics.

## Target selection layer (checked in the original exe)

Question from tg: does the original have a rule where melee attackers temporarily change target if another enemy blocks their way?

**Answer: yes, but only for attack-ground orders (kind 7), plus guard orders and no order. Not for attacking a specific target (kind 6).**

### Order kinds (table 0x4cff08, ported in ANDREI `player.cpp`)
- 1, 2, 3, 34: move-type orders (grey/blue markers).
- 4, 5: guard (blue markers; 5 = guard a unit).
- 6: attack a specific target (red marker, flag 0x200).
- 7: attack ground / attack-move (red marker).

### 0x493b70, AI target choice (ANDREI `select_attack`, TG `updateTarget`)
- Jump table at 0x494c4c / 0x494c64, indexed by order kind − 1:
  - kinds 1–3 and 34 → 0x493dc3: `and al,0xc3` (melee and enemy-ability scan off);
  - kind 4 → 0x493dd2; kind 5 → 0x493eb4;
  - kind 6 → 0x493fb7: only the ordered target is scored, then `and al,0xf3` (other enemies not scanned) at 0x494205, 0x494387, 0x49439e;
  - kind 7 → 0x4943a9;
  - no order and other kinds → 0x49445f (full scan).
- Kind 7 constants in the exe: look-ahead point **15** units from the creature toward the destination (0x4bfa98, 0x4bf948), reach **25** (0x41c80000), range **150** (0x43160000), threshold 0.25.
- Melee score: `(1 − 0.75·d/range)²`, ×1.25 for creatures vs buildings, ×1.2 for the last attacker, ×1.3 for the current target (stickiness). ANDREI also applies a crowd (path cost) penalty; TG uses a danger factor.
- The best choice is queued as a **temporary** order (0x46da50, flag 0x20). It is discarded and re-evaluated every think (0x46da80). After the temporary target dies, the creature resumes its original order.
- Effect: under attack-ground, guard or no order, an enemy standing in the way scores high and becomes a temporary target. This matches what tg remembered.

### Kind 6 executor (around 0x46e0b3)
- At 0x46e15a: `melee_target(0.25, mask 0x15)` (0x46a940) must return the ordered target. Otherwise it keeps approaching (0x45b620, radius 0.1). No retargeting.
- The movement blocker field +0xb70 is referenced only at 0x4592da/0x4592e2 (cleared on reference events), 0x45b6d7/0x45b729 (approach cache) and 0x45ba2d/0x45bbe1 (passed to obstacle avoidance). It is never used to pick a new target.

### The blow itself
- `CreatureDoAttack` (0x455bb0) calls 0x46a940 with length 1.0 and mask 0x15: the blow hits whatever is nearest in the swing at that moment, not necessarily the ordered target. An enemy stepping in front takes the hit. ANDREI reproduces this; TG's `meleeAttackTarget` has a similar effect.

### Group attack on a creature (0x46d180, ANDREI `player.cpp`)
- For kind 6 with flag 0x200, each formation slot is re-targeted to the nearest hostile creature in the same 20-unit grid cell as the slot point. This spreads a group over nearby enemies when the order is given. It is not a blocking rule.

### Status in the engines
- ANDREI: kind 7 / guard / idle logic ported in `creature_attack.cpp` (`select_attack`), kind 6 executor in `creature_order_run.cpp`, both matching the exe.
- TG: `updateTarget!true` (`advance`) uses the same numbers (15 ahead, refRange 150, threshold 0.25, ×1.2 last attacker, ×1.3 current target); explicit attack goes through `attack(targetId)` without retargeting. Nothing missing.

## Conclusions
- ANDREI is bit-for-bit closer to the original (segment sweep, integer math, logic driven by the FSM script from game data); its code is harder to read.
- TG is a more readable reconstruction with hand-picked exceptions. The melee hitbox is guessed, and some mechanics (stun cooldown, bonuses) are hard-coded and marked TODO.
- If a creature in TG hits from a different distance than in the original, the guessed hitbox is the most likely cause; ANDREI ports the original sweep (0x46a940) there.

## Not checked
- Whether TG's hitbox reach matches 0x46a940 numerically for each creature.
- TG's stun immunity duration vs the original FSM data.
- Callers of 0x46a940 at 0x42fce0, 0x430269 (length 2.0, mask 4) and 0x449df4 (probably spells).
