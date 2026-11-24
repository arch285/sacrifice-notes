# Creature aggression ranges: original vs sacengine (from Sacrifice.exe, sacengine 62c1b23)

Target selection in the original is 0x493b70 (andrei's `CreatureActor::select_attack` matches the exe on everything I checked). There are no fixed aggression ranges. On every think, the creature drops its temporary attack order and rescores all candidates against its base order. "Active" range = the same formula with the 1.3 current-target bonus.

## Timing / candidates
- Think: countdown +0x364 is set to 16 after a think and decremented by ticks; the creature thinks when it goes below 0, so at least 17 ticks between thinks, and at most one creature per side per tick.
- Every tick, controls are zeroed (0x45a39c) before the current order runs (0x46de80). No order → the creature stands still.
- Candidates come from the side's known list (side+0xc): anything any unit of the side has seen. An entry is dropped 80 periods × 32 ticks after its last refresh (0x466cc0 → 0x4939e0(clock-80, id, 72)), so it stays ~38–43 s after the object leaves view. Buildings/souls never expire. No sight range or current visibility check in target selection. sacengine: isVisibleToSide, 6.4 s.

## Formulas
- Melee: near = d<R ? 1-0.75d/R : 0.25R/d; value = near² · (aggressiveness≠0) · bias · crowd. bias: 1.2 last attacker, ×1.3 previous melee target; crowd = 1-cost·0.00625 if path cost ≥ 9.
- The 1.25 creature factor (vs 1 for buildings) is only used to rank candidates. **The threshold test uses value without it** (0x494b22 / 0x494bd2). sacengine puts 1.25 into the score compared with the threshold, so its ranges are too large (idle 73.7/81.0 instead of 66.7/74.9; advance 110.6 instead of 100).
- Hostile abilities (ranged creatures, 0x494d60): only considered if |centre-target|² ≤ (reach + SPEL.range)² (positions, not hitboxes). value = (1 - d²/limit²·0.5)·bias, i.e. 0.5..1. Bonuses don't extend the limit. Needs cast_check ≤ 1 (cooldown OK, missing mana not), line of fire, evaluator > 0 (skipped for an explicit attack order).
- A ranged creature with current mana > 0 (+0xb20) does not score melee at all (0x494466). If 0 < mana < cost it picks nothing.
- Shooting (0x46e3db): approaches while hitbox separation > 0.75·range, then fires. sacengine: max(0.8r, r-30), centre distance.

## Per base order (centre / R / reach / threshold)
- none (idle, also after a plain move): self / 100 / 0 / 0.25
- move, follow (1–3, 34): no attacks
- guard location (4): point / 50 if the owner is a human wizard (+0xb80 == 0), else 100 / 10 / d_c/100 (d_c ≤ 0 → 0, ≥ 100 → 1). No hard leash.
- guard object (5): guarded object's position / same R / 10 / 1-n², n = d_c<100 ? 1-0.0075d_c : 25/d_c; ×1/3 if the side has an AI controller (side+0xd8)
- attack target (6): that target only / 150 / 10 / 0
- attack location (7): self + min(15, dist) towards dest / 150 / 25 / 0.25
- Orders 4 and 7 stay after arrival (arrive radius 0.5, pop(0)), same as sacengine.

## Resulting numbers
- Idle melee: new 66.7, last attacker 72.5, current target 74.9, both 80.0.
- Idle ranged: any known enemy within SPEL range of itself (archer 110, squall 150, flak 100–200...).
- Advance melee: 100 / 112.3 / 119.9 from the shifted point; ranged: range + 25.
- Guard location melee (R50/R100), target distance from the point at d_c 5/10/20/50: 55.9/111.8, 45.6/91.2, 36.9/73.7, 19.5/39.1. Ranged: anything within range+10 of the point while d_c < 50 (65 when continuing).
- Guard object melee R50 at d_c 10/20/40: 41.3/31.5/19.1; AI side: 114/92.8/78.4.

## Two consequences (this is how the original behaves)
1. A creature standing on its guard point has d_c ≤ 0.5, so the threshold is ≤ 0.005 and it goes after any known, reachable enemy up to 177 (human, R50) / 354 (AI) from the point. At the next think it keeps the target only if 1.3·near(D)² > d_c/100. Against a stationary enemy at D=100 it gives up after ~2 units, at D=60 ~5.6, D=40 ~21. Then order 4 walks it back, and it picks the target again → a few-unit yo-yo. sacengine has the same threshold but R=100 for everyone plus the 1.25 factor (~395 on the point) and a hard 50 leash.
2. Idle ranged, target leaves range: between thinks order 66 keeps approaching (to 0.75·range), so it follows for at most ~17 ticks. At the next think the target is out of range, melee is off because of mana, nothing is chosen → no order → it stops where it is (doesn't return). With 0 mana it scores melee like a normal creature and chases. sacengine picks ranged targets like melee (~74) and then chases without limit.

## Other differences
- Guard object: sacengine reuses the guard-location formula around the guarded position + formation offset.
- aggressiveness 0: sacengine disables everything, the original only zeroes the melee score (no practical difference in the data).
- Ranged in melee contact: sacengine switches to melee (rangedMeleeAttackDistance 0.5), the original keeps shooting while it has mana.

All of this is from the exe code; I haven't observed it in a running game.
