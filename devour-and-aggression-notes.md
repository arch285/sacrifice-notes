# Devour + aggression ranges: original vs sacengine

Sources: `Sacrifice.exe` (addresses below), SacrificeModern (andrei-cb, 24fc0e5) as a map into the exe, sacengine 62c1b23 (`source/state.d` line numbers).
The original is the reference. Anything strange it does is listed as behaviour to copy, not as a bug.
Items marked *(Andrei)* come from SacrificeModern and were not re-checked in the exe. Nothing here was tested in a running game.
Time: 60 ticks/s.

---

## 1. Aggression ranges (passive / active), with focus on ranged creatures

### How target selection runs in the original
- Selection: `0x493b70` (SacrificeModern `CreatureActor::select_attack`, `creature_attack.cpp`). Andrei matches the exe at every point checked.
- Called from `think` through a per-side work queue (`0x473130`). The counter at +0x364 is set to 16 after thinking and decremented by elapsed ticks. So there are **at least 17 ticks** between two thinks of one creature, and at most one creature thinks per side per tick.
- Every tick the creature's controls are zeroed (`0x45a39c`) and then the current order runs (`0x46de80`). A creature with no order stands still.
- At the start of `think` the previous *temporary* order (auto-picked attack or shot) is always discarded, and everything is recomputed against the base order. "Active" range is the same formula with a ×1.3 bonus for the current target. Between thinks the temporary order runs normally (walking to the target).
- Candidates: the side's `known` list (side+0xc), i.e. everything any creature of the side has seen. An entry expires 80 periods of 32 ticks after its last refresh. A visible object is refreshed about every 8 periods, so memory after losing sight is 72–80 periods = 2304–2560 ticks (~38–43 s). Buildings and souls get +1000000 and never expire. Selection does **not** check current visibility or sight range.

### Formulas (`0x493b70`)
- Shared: `centre`, `range` (default 100), `threshold` (default 0.25), `reach` (default 0).
- Melee: `d = |centre − target|`; `near = d<range ? 1 − 0.75·d/range : 0.25·range/d`; `value = near² · aggr · bias · crowd`.
  - `aggr` = 1 if CRE8 aggressiveness ≠ 0, else 0.
  - `bias` = 1.2 if the target is the last attacker (+0x430), ×1.3 if it was the previous melee target (order 6).
  - crowd: path cost at the target ≥ 9 → ×(1 − cost·0.00625).
- Ranking uses `value · scale` (scale 1.25 for creatures, 1 for buildings). **The threshold is compared against `value` without the 1.25** (`0x494b22` stores both; `0x494bd2` compares the unscaled one).
- Hostile abilities (`0x494d60`, creatures with flag 8 = ranged): only considered when `|centre − target|² ≤ (reach + SPEL.range)²` (positions, not hitboxes).
  - `value = (1 − d²/limit² · 0.5) · bias`, i.e. 0.5..1 · bias. bias 1.2 for the attacker, ×1.3 if the previous order was 66 on this target. Scale 2 for creatures, 1 for buildings.
  - Requires: cast_check ≤ 1 (cooldown passes, missing mana does not), line of fire (0x30000), SPEL evaluator > 0 (skipped for order 6, `0x495430` flag 0x100).
  - Bonuses do not widen the range limit (it is checked before value is computed).
  - Spells with property flag 0x800000 are never auto-selected (`0x494e62`).
- A creature with flag 8 and current mana > 0 (+0xb20) **does not score melee at all** (`0x494466`). If mana is > 0 but below the shot cost, it selects nothing.
- Shot execution (`0x46e3db`): the creature walks while the hitbox gap is > **0.75·range**, then shoots.
- Guard-point threshold (`0x493dd2`): `d_c ≤ 0 → 0`; `0 < d_c < 100 → d_c/100`; `d_c ≥ 100 → 1`.

### Modes (by base order)
| Order | centre | range | reach | threshold |
|---|---|---|---|---|
| none (idle, also after a plain move) | creature | 100 | 0 | 0.25 |
| 1–3, 34 (move, follow) | — | — | — | attacks off (`capabilities &= 0xc3`) |
| 4 guard point | the point | 50 if the owner is a human wizard (+0xb80 = 0), else 100 | 10 | d_c/100 (d_c = creature's distance from the point, max 1) |
| 5 guard object | guarded object's position | as above | 10 | 1 − n², n = d_c<100 ? 1−0.0075·d_c : 25/d_c; ×1/3 if the side has an AI controller (side+0xd8) |
| 6 attack target | that target only | 150 | 10 | 0 |
| 7 attack-move (advance) | creature + min(15, distance) towards goal | 150 | 25 | 0.25 |

Execution table `0x46e2b8`: orders 1, 2, 34 → reach within 0.5, then forced pop. Orders 4 and 7 → reach within 0.5, then pop(0), which does not remove the last order, so they stay after arrival. sacengine already does the same (`unqueueOrder` with an empty queue is a no-op).

### Resulting distances
- Idle melee (from the creature): new target < **66.7**, last attacker 72.5, current target 74.9, both 80.0. Buildings the same (66.7 / 74.9).
- **Idle ranged (with mana): any known enemy within SPEL range of the creature** (archer fbow 110, Squall abma 150, flak 100–200, claws 150, zappers 50…). Walks to 0.75·range (hitbox gap) and shoots.
- Advance melee (from the point 15 ahead): 100 / 112.3 / 119.9. Ranged: range + 25 from that point.
- Guard point, melee, target's distance from the point (R50 human / R100 AI) at creature distance d_c: 5 → 55.9 / 111.8; 10 → 45.6 / 91.2; 20 → 36.9 / 73.7; 30 → 30.2 / 60.3; 50 → 19.5 / 39.1. At d_c ≤ 0.5 (standing on the point): 177 / 354.
- Guard point, ranged: any target within range+10 of the point, while d_c < 50 (new) / 65 (continuing, ×1.3).
- Guard object, melee R50: d_c 10 → 41.3, 20 → 31.5, 30 → 24.5, 40 → 19.1. AI side (R100, threshold/3): 10 → 114, 20 → 92.8, 40 → 78.4. Ranged: any target within range+10 of the guarded object while threshold < 0.5 (d_c < ~39 new, < ~55 continuing).

### Behaviour worth copying
- **Guard point "yo-yo".** After arriving, the creature stays within 0.5 of the point (`0x46dee8`), so the threshold is ≤ 0.005. Melee takes any known, reachable enemy up to 177 (human, R50) / 354 (AI, R100) from the point and walks to it (temporary order 6) until the next think (≥ 17 ticks). Then threshold = d_c/100 and the target is kept only if `1.3·near(D)² > d_c/100` (D = target's distance from the point). For a standing target at R50: D=100 → dropped after ~2 units of walking; D=60 → ~5.6; D=40 → ~21; D=30 → ~39. The base order 4 walks it back to the point, the threshold drops to ~0, the target is picked again. A far, standing enemy causes a few-unit yo-yo around the point; an approaching enemy is fought normally.
- **Idle ranged creature and a target leaving range.** Between thinks order 66 keeps approaching (`0x46e3db`), also when the target runs, so the creature follows **at most until the next think** (≥ 17 ticks). At the think: temporary order dropped, out-of-range target not considered (limit before bonuses), melee disabled by mana → nothing chosen → no order → controls zeroed every tick → the creature stops where it is and does not walk back. It keeps choosing nothing until the target returns into range or another one appears.
- Without mana (mana = 0) a ranged creature scores melee like any other (idle 66.7 / 74.9) and chases in melee.

### sacengine differences (`updateTarget` ~14433, `patrolAround` ~14511, `guard` ~14522, `patrol` ~14538, `advance` ~14547, `attack` ~14320, `aim` ~14019)
1. One formula for everyone, ranged included (scarab excepted): `targetScore = near² · dangerFactor · 1.25 (non-buildings) · 1.2 · 1.3`, compared to the threshold. The 1.25 enters the threshold; in the original it only ranks. → idle 73.7 / 78.9 / 81.0 / 85.6 instead of 66.7 / 72.5 / 74.9 / 80.0; advance 110.6 instead of 100.
2. No SPEL range and no `reach` in selection: ranged creatures pick targets like melee (~74 from themselves when idle) instead of anything in shooting range. Once chosen, they chase without limit; the original lets go at the next think when the target is out of range.
3. Shooting distance: `max(0.8·r, r−30)` from the centre (`aim`/`shootRange`) vs 0.75·r hitbox gap.
4. Guard point: hard leash 50 (`guardRange`) + threshold min(100,d)/100 with refRange always 100. Original: no hard leash, R = 50 for creatures of a human player. On the point sacengine reaches ~395 for a human player instead of 177.
5. Guard object: same formula as guard point, around the formation-offset position. Original: separate formula 1 − n² around the guarded object itself, ×1/3 for AI sides.
6. Pacifist: aggressiveness 0 disables everything in sacengine; in the original it only zeroes melee (no practical difference in the data).
7. Visibility: sacengine requires the side to have seen the target in the last 6.4 s (`isVisibleToSide`, 32/5 s). Original: `known` list, ~38–43 s.
8. Ranged creature in close combat: sacengine switches to melee (`rangedMeleeAttackDistance` 0.5); the original keeps shooting while it has mana.

---

## 2. Devour (`ifna` / SPEL `anfi`, run function `SoulEaterRun` 0x41e1f0)

### Data
- SPEL `anfi` (Spells.wad, `@ABI`): range **2.0**, targets 0x1002 (souls | only ally), property flags 0x840000, cooldown 0, mana 0, cast time 0, amount3 1. Both sound slots are `ltss`: event 2 (start) and event 8 (repeating, see below).
  - Property bit 0x800000 = never auto-selected by creature target selection (`0x494e62`). sacengine calls bit 23 `aiCastable`, but in the exe it excludes the spell from automatic use.
  - Property bit 0x40000 = second cast animation (FSM event 54 after 13).
- Two CRE8s have it:
  - `dprc` = **Netherfiend**: `ability0` = `anfi`, so Devour is its only ability (melee creature → sacengine uses `abilities[0]`). HP 1800, regeneration 0, resistances melee/direct spell/splash spell/direct ranged/splash ranged = 750/2000/2500/900/900.
  - `dmon` = **Hellmouth**: `ability0` = `abhm` (its ranged attack), `ability1` = `anfi` (ranged creature → sacengine uses `abilities[1]`). HP 5400, regeneration 0, resistances 1000/500/500/750/750.
  - The effect code is the same for both; only the starting stats and the creature size (approach distance, bulk visual) differ.

### Who can be devoured
- A soul is a valid target when the caster's side is in the soul's `eligible_sides` and the soul is not already being collected (soul flags 0xc0000000):
  - soul with no anchor (free) → every side;
  - soul still on a corpse → the corpse owner's side (+ allies when allied soul pickup is on).
  - In other words: blue souls only. Red souls must be converted first.
- Eligibility is checked again when the effect starts (stage 1, `0x41e2f9`); if it fails, the effect ends.
- Never used automatically (bit 0x800000 above, and souls are not attack candidates anyway). Only on an explicit order. I found no devour use in Andrei's computer-player code either *(Andrei, not checked in exe)*.
- sacengine: order targeting matches (`isApplicable` with targetSouls|onlyAlly → blue souls). At shot time `devourSoul` (~14650) only checks soul state normal/emerging, not colour.

### Approach and cast *(Andrei, `execute_spell_order`, creature_order_run.cpp)*
- The creature walks until the squared gap between its bounding box and the soul is ≤ (0.75 · range)² = **1.5²**. The soul's own box is empty in Andrei, so in practice: soul point within 1.5 of the creature's box, in 3D. Terrain visibility to the soul is also required.
- Then it turns to face the soul within 5° (0.0873 rad), then casts: FSM event 13, then 54 (second cast animation). The order is popped.
- sacengine (`useAbilityDistance` ~13982, `aim` ~14034): 2D distance from the creature's position (feet) to the soul centre ≤ hitbox diagonal length − speed/frame. For a creature as big as the Hellmouth this is much farther than "box within 1.5".

### Effect (`SoulEaterRun` 0x41e1f0, checked in the exe)
1. Start: sound event 2 (`ltss`). If the soul is eligible, it is collected by the caster (`0x4765f0`), detaching it from its corpse. Otherwise the effect ends.
2. Next tick: if the soul's collector is the caster → stage 16, else the effect ends.
3. Stage 16, once, `n` = number of souls in the soul object. Repeated `n` times:
   - max HP (base) += 500. Written as a 16-bit value with no clamp in the exe (Andrei clamps to 32767; only matters after ~55 souls).
   - current and target HP are rescaled proportionally: `trunc(newMax · current / oldMax)`. HP **percentage** is kept, not +500 absolute.
   - regeneration = `(newMax + 3 · regeneration) / 4` (integer, truncated toward zero); current/target regeneration rescaled the same way.
   - all five resistances (stat slots 9–13) = `trunc(0.9 · value)`, current/target rescaled.
   - heal with amount = new max HP, rate 256 (`0x43f610`): one healing effect that heals 64 HP every 16 ticks (**240 HP/s**) until full.
   - Then `remaining = 100 · (3n + 9)` ticks (n = 1 → 1200 ticks = 20 s; n = 2 → 1500).
4. Stage 17: every tick the soul is pulled to the caster's animation reference point 0 (mouth, `0x44e660` / `0x476730`). Each tick sound event 8 (`ltss`) plays with probability ticks/64 (≈ once per second). When the soul is gone → stage 18.
   - Soul flight *(Andrei, soul_actor.cpp)*: after collection the soul interpolates from where it was to the destination while its growth drops 0.05/tick → ~20 ticks.
5. From stage 16 until the effect ends, every tick: bone scale request (1.5, 1.5, 1) on every bone (`0x459380`).
   - *(Andrei, creature_deformation.cpp)*: scale approaches the request at 0.01/tick (~50 ticks to reach 1.5), and returns to 1 at 0.05/tick (~10 ticks) once requests stop. Several overlapping devours average their requests.
   - **The bulk-up is temporary**: it lasts `100 · (3n + 9)` ticks and then shrinks back. The stat gains are permanent.
6. The effect ends when `remaining` ≤ 0 or when the caster is dead.

Regeneration units in the original: HP per minute. Per tick the creature gains `regen · ticks · manaRatio / 3600` HP in 16.16 fixed point (`0x45a210`), where manaRatio = current/max mana (1 without mana).

### Netherfiend numbers (original vs sacengine)
| souls | max HP | regen (HP/min) orig | regen sacengine | melee / d.spell / s.spell / d.ranged / s.ranged orig | sacengine |
|---|---|---|---|---|---|
| 0 | 1800 | 0 | 0 | 750/2000/2500/900/900 | same |
| 1 | 2300 | 575 | 575.00 | 675/1800/2250/810/810 | same |
| 2 | 2800 | 1131 | 1131.25 | 607/1620/2025/729/729 | 607.5/1620/2025/729/729 |
| 3 | 3300 | 1673 | 1673.44 | 546/1458/1822/656/656 | 546.8/1458/1822.5/656.1/656.1 |
| 4 | 3800 | 2204 | 2205.08 | 491/1312/1639/590/590 | 492.1/1312.2/1640.2/590.5/590.5 |
| 5 | 4300 | 2728 | 2728.81 | 441/1180/1475/531/531 | 442.9/1181/1476.2/531.4/531.4 |

(575 HP/min ≈ 9.6 HP/s. Resistances are damage multipliers in ‰, so the Netherfiend starts at 2×/2.5× spell damage and each soul cuts every multiplier by 10 %.)

### Hellmouth numbers (original vs sacengine)
| souls | max HP | regen (HP/min) orig | regen sacengine | melee / d.spell / s.spell / d.ranged / s.ranged orig | sacengine |
|---|---|---|---|---|---|
| 0 | 5400 | 0 | 0 | 1000/500/500/750/750 | same |
| 1 | 5900 | 1475 | 1475.0 | 900/450/450/675/675 | same |
| 2 | 6400 | 2706 | 2706.25 | 810/405/405/607/607 | 810/405/405/607.5/607.5 |
| 3 | 6900 | 3754 | 3754.69 | 729/364/364/546/546 | 729/364.5/364.5/546.8/546.8 |
| 4 | 7400 | 4665 | 4666.02 | 656/327/327/491/491 | 656.1/328.1/328.1/492.1/492.1 |
| 5 | 7900 | 5473 | 5474.51 | 590/294/294/441/441 | 590.5/295.2/295.2/442.9/442.9 |

(1475 HP/min ≈ 24.6 HP/s.)

### sacengine differences (`devourSoul` ~14650, bulk ~15794)
1. **Visual: permanent vs temporary.** sacengine grows the creature permanently with `bulk → 2 − 0.75^numBulks` (shader inflate). The original scales every bone by (1.5, 1.5, 1) for `100 · (3n + 9)` ticks only, then returns to normal size. This is the most visible difference.
2. Current HP: sacengine adds +500 to current HP; the original keeps the HP percentage (2700/5400 → 2950/5900, sacengine 3200/5900). Both then heal to full at 240 HP/s, so this only matters while the heal runs.
3. Regeneration: sacengine's increment recursion (`0.25·maxHealth`, then `0.75·inc + 125`) gives the same result only because both devouring creatures have base regeneration 0 and nothing else touches it. Simpler and exact: `regen = (maxHealth + 3·regen) / 4` in retail units (HP/min), i.e. `regen_s = (maxHealth/60 + 3·regen_s) / 4` in sacengine's HP/s. The `devourRegenerationIncrement` field is then not needed (it carries a `// TODO: is this needed?` already).
4. Rounding: the original truncates regeneration and resistances to integers at every step; sacengine keeps floats (differences ≤ ~2 after 5 souls).
5. Sound: sacengine plays `ltss` once. The original plays it at the start and then randomly ≈ once per second while the soul is being eaten (until the soul is gone).
6. Approach distance: see "Approach and cast" above.
7. Timing: sacengine applies the stats at the shot frame; the original one tick later (after collecting). Negligible.
8. Colour recheck at shot time: the original rechecks eligibility when the effect starts; sacengine only checks soul state.
9. Heal: same speed (240 HP/s, rainbow heal with infinite amount = full). sacengine's `canHeal` and the original both skip when already at full HP; no practical difference found.
