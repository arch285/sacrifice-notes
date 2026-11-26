# Creature HP regeneration: original vs sacengine (petrification and freeze, guardian, poison, Rings of Fire / Slime / Vines)

Sources: `Sacrifice.exe` (addresses below), game WADs (`Fsm.wad`: FSM state flags; `Spells.wad`: CRE8 and SPEL), SacrificeModern (andrei-cb) as a map into the exe, sacengine 056e674 (`source/state.d`, line numbers).
The original is the reference. Nothing here was tested in a running game.
Time: 60 ticks/s. Regeneration in the original and in CRE8 is in HP/min, in sacengine in HP/s.

---

## Original

### Regeneration (0x45a210, called every tick from 0x45a300)
Each tick the creature gains `regen · ticks · manaRatio / 3600` HP (16.16 fixed point, fraction at +0xb30), where `regen` = current stat 6 (+0x8d0) and `manaRatio` = mana/maxMana (+0xb20/+0xb24; 1 without mana).

All of these must hold:
1. HP ≠ max (+0x8ce vs +0x8fe) and HP ≠ 0.
2. The creature is alive (vfunc +0x34 = 0x46abe0: FSM state without flags 0x44000).
3. **One of three**:
   - the current FSM state has flag **0x800**,
   - CRE8 `behaviorFlags` has bit **0x2000** (+0xb38 = copy of `behaviorFlags`, set at 0x45917d),
   - the creature is a **guardian** of a building (+0x5c8 ≠ 0).

When the condition fails, the fraction at +0xb30 is reset to 0.

### Creatures with CRE8 0x2000 (read from `Spells.wad`, CRE8 `behaviorFlags`)
| creature | tag | HP | regeneration (HP/min) | mana | flags |
|---|---|---|---|---|---|
| Troll | `trol` | 1600 | 1750 (≈ 29.2 HP/s) | 0 | 0x2000 |
| Sac Doctor | `sacd` | 400 | 5000 (≈ 83.3 HP/s) | 0 | 0x2100 |
| Familiar | `fami` | 1000 | 1000 | 2000 | 0x23a0 |
| Zyzyx (mentor) | `ment` | 1000 | 1000 | 2000 | 0x23a0 |

None of the other 71 CRE8 files has this bit. Matches the comment in sacengine `spells.d`.

### States with flag 0x800 (Fsm.wad, "Normal creature FSM")
`STAN main` (standing), `CHAR char`, `ANIM norm/flyn`, `INVI stan`, `PDED play/hide`, `FLYN main` (flying). Without 0x800, among others: `MOVE` (running), `ATTK`, `DAMG`, `LDMG`, `CAST`, `ABIL`, `FALL`.

### Petrification (Turn to Stone, `TurnToStoneRun` 0x4478b0)
- The creature enters FSM group `PARA`. State flags: `stan`, `_ko_`, `hitf`, `getu` = 0x20; `fall`, `fdth` = 0x24; `deth` = 0x5020 (death). Same in "Just flying creature FSM" (0x20/0x24/0x5020/0x5024). Sac doctor: `PARA` 0x20002020.
- **No `PARA` state has 0x800.** A petrified creature does not regenerate on its own.
- Turn to Stone does not change stats (checked calls 0x4215b0, 0x421630, 0x446990, 0x446900: only the `PARA` effect name, damage/death watchers, particles, sound). Damage to a petrified creature ×0.2 (0x447880).
- Freeze uses the same `PARA` group and the same hold function 0x421630 (per SacrificeModern `spell_control.cpp`), so it behaves the same for regeneration.
- Exceptions that regenerate despite the stone: CRE8 0x2000 creatures (table above) and guardians.

### Guardian (`GuardianSpellRun` 0x45ef10, update 0x45e980)
- 0x45e980 is called every tick from 0x45a300 (0x45a368) when +0x5c8 ≠ 0. Requires the same side as the building and a living creature (vfunc +0x34). **It does not check the FSM state**, so it also works in stone.
- `f = n > 3 ? 3/n : 1`, n = number of guardians of the building (building +0x450).
- Target regeneration (stat 6, +0x930) += `1000 · f`. Other stats: +0x932 (stat 7) ×(1+f), +0x934 (stat 8) ×(1−f), ranged resistances ×(1−0.75f), spell resistances ×(1−0.5f).
- The current stat approaches the target by **10 per tick** (0x45a670), and the target is then reset to base. So the +1000 bonus builds up over ~100 ticks (~1.7 s) and decays the same way after the guardian is removed.
- "Guardian healing" in the original is an **addition to normal regeneration**, not a separate heal. It is subject to the mana multiplier and the conditions of 0x45a210, and the guardian has the regeneration gate open in every living state.

### Poison and guardian: order
- Poison (`POIS`, procedure with `pois` particles and the `POIS` name at 0x4401c9–0x44023e, stat writes at 0x4402a6–0x4403e0): every tick while active, sets target regeneration (+0x930) **to 0** and target stat 20 (+0x94c, mana gain from manaliths, below) to 0. Unconditionally: the `freg` argument of POIS is not used for stats.
- Order within a tick (0x45a620): 0x45a300 (containing 0x45e980 = guardian bonus) → … → 0x45a670 (current stats approach targets, targets reset to base). Poison is a separate spell object and is not run inside the creature update (0x46a6c0 only counters).
- Whether the poison object runs before or after the creature in the same tick, its zero reaches the target **before** the guardian bonus is added, and the bonus is added before the stats approach. Result: a poisoned guardian has target regeneration `0 + 1000·f`. **Poison removes the guardian's own regeneration, but not the bonus.**
- Same with Ikarus (`abil`, 0x412e22: target regeneration ×`freg`, `freg` = 0 in the data): the multiplication happens before +1000 is added.
- `freg` in the data (Spells.wad): Plague `plag` 0, Necryl attack `anec` 0, `@nec` 0, Ikarus `abil` 0 (fspd 0.75).

### Mana while petrified
- Creatures get mana only from a manalith/altar. Callback 0x465880 only checks that the creature is not dead (vfunc +0x38 = 0x46abd0: state flag 0x4000), then calls 0x4650d0: `mana_rate · ticks · stat20 · 0.001` (16.16, fraction +0xb2c), clamped to maxMana.
- `PARA` has no 0x4000, so **a petrified creature gets mana from a nearby manalith** as usual. Range 50 (3D) and side conditions per SacrificeModern (`building_actor.cpp` `scan_mana`), not checked in the exe.
- Poison zeroes target stat 20, so mana gain from manaliths fades over ~100 ticks (current value drops by 10/tick from 1000).

### Rings of Fire, Slime, Vines (checked in the exe)
sacengine names → original procedures (addresses from the name table in the exe, e.g. 0x4cdc24): Rings of Fire `rapf` = **ChainsOfFireRun** 0x4258a0 (SPEL `fpar`), Slime `htel` = **LethargyRun** 0x432550 (SPEL `leth`), Vines `vine` = **GraspingVinesRun** 0x42c2f0 → **VinesHoldRun** 0x42c090.

All three only register the **effect name** `PARA` on the target (0x4215b0 → 0x4692a0) and remove it at the end (0x469380 / 0x469300). The `PARA` name is used for spell-cast exclusions; 0x45a210 does not read it. Unlike Turn to Stone and Freeze, none of them sends FSM event 24 (no calls to 0x454d90, no 0x421630), so **the creature does not enter the `PARA` state**.
- **Rings of Fire (ChainsOfFireRun):** only continuous (accumulated) damage and visuals. Accumulated damage (vfunc +0xf0 = 0x459830 → 0x454870 → 0x454a10) subtracts HP without an FSM event, so it does not put the creature into `DAMG`/`LDMG`. No stat changes.
- **Slime (LethargyRun, stats at 0x4321f0):** target stats 0, 1, 3 ×0.2 (slow) and resistances 9–13 (+0x936…+0x93e). Stat 6 (regeneration, +0x930) is **not touched**. No FSM events.
- **Vines (VinesHoldRun):** every tick sets the target's position and flag +0x234 |= 0x80000 (pinned). This flag only disables movement: 0x45b620 (move to point, called from order execution) exits early and sets no controls; 0x45a8f2 in the movement code checks it too. No stat changes and no FSM events of its own.
  - Running → standing transition (checked): movement controller 0x45a8b0 computes `controlled = (FSM state flags & 1) && !(+0x234 & 0x80000)` (0x45a8ed–0x45a905). At the end (0x45b22c–0x45b275), when the current state has flag 1: if walk speed (+0x1f0 or +0x1f4) ≠ 0 **and** `controlled` → FSM event 3 (`STAN main` → `MOVE init`), otherwise event 20. In `MOVE init` (flags 0x600001) event 20 leads to `STAN init` (stance selector), which leads to `STAN main` (0x600801).
  - For a pinned creature `controlled` = 0, so every tick sends event 20 and never event 3. A running creature switches to `STAN main` on the tick after being pinned and stays there (0x800 → regeneration). Attacking, casting and abilities still work (`ATTK`, `CAST`, `ABIL` have no 0x800, so there is no regeneration during them, as always).

Conclusion: **in the original, Rings of Fire, Slime and Vines disable neither regeneration nor the guardian bonus.** Own regeneration follows the normal rules (FSM state with 0x800, CRE8 0x2000 or guardian), and the guardian bonus (+1000·f to stat 6) is unchanged.

### Result in the original
| situation | healing |
|---|---|
| normal creature, petrified | 0 |
| troll / sac doctor, petrified | `regen · manaRatio` (troll 29.2 HP/s, sac doctor 83.3 HP/s; mana 0 → ratio 1) |
| guardian, petrified | `(regen + 1000·f) · manaRatio` |
| guardian, poisoned (also in stone) | `1000·f · manaRatio` |
| guardian running / attacking / hit | `(regen + 1000·f) · manaRatio` |
| normal creature running / attacking | 0 |
| creature under Rings of Fire / Slime / Vines | as without the effect (e.g. standing: `regen · manaRatio`) |
| guardian under Rings of Fire / Slime / Vines | `(regen + 1000·f) · manaRatio` |

Example: a guardian with no own regeneration, full mana, ≤ 3 guardians → 1000 HP/min ≈ 16.7 HP/s, also in stone. 4 guardians → 750 HP/min (12.5 HP/s), 6 → 500 HP/min (8.3 HP/s). A guardian with regeneration 1750 (e.g. Ent) with ≤ 3 guardians: 2750 HP/min ≈ 45.8 HP/s.
Note: almost all creatures have mana in CRE8 (e.g. Ent 500), so `manaRatio` also applies to melee creatures once they have spent mana on an ability.

---

## Sacengine (056e674)

- `petrify` (~13026): `creatureState.mode = stunned`, `effects.petrified = true`. `setCreatureState` (~8781) keeps `stunned` while the stone lasts.
- `isRegenerating` (~1486): modes `idle`, `cower`, `playingDead`, `rockForm`, flying while moving, no hit animation; **or** `continuousRegeneration && canHeal(mode)` (troll, sac doctor in `nttData.d`). All of that **and** `!regenerationBlocked`.
- `regenerationBlocked` (`stats.d:88`) = `immobilized || ringsOfFire || slimed || vined`, and `immobilized` = `petrified || frozen`. **So stone blocks regeneration for everyone, troll and sac doctor included.**
- `updateCreatureStats` (~16267):
  - normal regeneration: `manaRatio · regeneration · poisonRegeneration / 60` per frame while `isRegenerating`;
  - guardian: separately `heal(1000/(60·60))` per frame = 16.7 HP/s, **with no conditions at all** (mode, stone, poison, mana, number of guardians).
- Poison: `poisonRegeneration *= freg` (`freg` from data = 0, so own regeneration 0). Mana from manaliths blocked after `manaBlockDelay`. Does not affect the guardian heal.
- Mana while petrified: `canRegenerateMana(stunned)` = true, so a petrified creature gets mana from a manalith.
- A guardian stays a guardian in stone (`isValidGuard` ~17270 only excludes dying, dead and `thrashing`).

### Result in sacengine
| situation | healing |
|---|---|
| normal creature, petrified | 0 ✔ |
| troll / sac doctor, petrified | **0** ✘ (original: full regeneration) |
| guardian, petrified | 16.7 HP/s (no own regeneration, no mana factor, no `f`) |
| guardian, poisoned | 16.7 HP/s (+ own × 0) |
| guardian standing | `regen · manaRatio` + 16.7 HP/s |
| guardian running / attacking / hit | 16.7 HP/s (no own regeneration) |
| creature under Rings of Fire / Slime / Vines | **0** ✘ (`regenerationBlocked`; also troll and sac doctor) |
| guardian under Rings of Fire / Slime / Vines | 16.7 HP/s (no own regeneration) |

---

## Differences
1. **Normal creature in stone: matches.** No regeneration (no 0x800 in `PARA` ↔ `stunned` not in the list, plus `regenerationBlocked`).
2. **Troll and sac doctor in stone: differs.** The original keeps regenerating them (CRE8 0x2000 bypasses the FSM gate). sacengine blocks it via `regenerationBlocked` (`immobilized`). Same under Freeze.
3. **Guardian: own regeneration.** The original adds it in every living state (also in stone, running, attacking, when hit). sacengine only in `isRegenerating` states.
4. **Guardian: mana multiplier.** The original multiplies the whole amount (including +1000) by mana/maxMana. sacengine heals the guardian a flat 16.7 HP/s.
5. **Guardian: number of guardians.** Original `1000 · min(1, 3/n)`. sacengine always 1000.
6. **Guardian: build-up.** The original reaches +1000 at 10/tick (~1.7 s) and decays the same way. sacengine is instant.
7. **Poison on a guardian: matches in practice.** In both the guardian bonus stays and own regeneration is gone (only points 4–6 differ).
8. **Mana in stone: matches.** Both give a petrified creature mana from a manalith.
9. **Rings of Fire, Slime, Vines: differs.** The original does not block regeneration (none of these procedures changes stat 6 or the FSM state). sacengine blocks all own regeneration via `regenerationBlocked` (`ringsOfFire || slimed || vined`). Guardian bonus: kept in both.
10. Familiar and Zyzyx have 0x2000 in the original, but no `continuousRegeneration` in sacengine. Whether they can be petrified was not checked.

---

## What it should be (per the original)
- Stone gives **no** regeneration to normal creatures. No change.
- Troll, sac doctor (and familiar, Zyzyx) regenerate in stone and ice: `regen · manaRatio`.
- A guardian heals in stone at `(regen + 1000·min(1, 3/n)) · manaRatio` HP/min. Poisoned: without `regen`.
- Rings of Fire, Slime and Vines do not affect regeneration. Only the creature's state (standing / running / attacking) decides, as without the effect.

Sketch of a sacengine change (not compiled, not tested):

```d
// stats.d: regenerationBlocked is no longer needed (its only use is isRegenerating).
// Stone and ice disable regeneration through the stunned mode (not in the isRegenerating list);
// Rings of Fire, Slime and Vines do not disable regeneration in the original.

bool isRegenerating(B)(ref MovingObject!B object){
	return (object.creatureState.mode.among(CreatureMode.idle,CreatureMode.cower,CreatureMode.playingDead,CreatureMode.rockForm)
	        ||object.creatureState.mode==CreatureMode.moving&&object.creatureState.movement==CreatureMovement.flying)
	       &&!isBeingAttacked(object)
	       ||(object.sacObject.continuousRegeneration||object.isGuardian)&&object.creatureState.mode.canHeal; // guardian: every living mode
}

// updateCreatureStats:
auto manaFactor=object.creatureStats.maxMana==0.0f?1.0f:object.creatureStats.mana/object.creatureStats.maxMana;
if(object.isRegenerating) with(object.creatureStats)
	object.heal(manaFactor*max(0.0f,regeneration*effects.poisonRegeneration)/updateFPS,state);
if(object.isGuardian&&object.creatureState.mode.canHeal){
	auto n=/* number of guardians of the building (building.guardianIds.length) */;
	object.heal(manaFactor*1000.0f/60.0f*min(1.0f,3.0f/n)/updateFPS,state); // bonus is not affected by poison
}
// remove: if(object.isGuardian) object.heal(1000.0f/(60*updateFPS),state);
```

Optional (accuracy): guardian bonus building up/decaying by 10 HP/min per tick (≈ 1000 over 100 frames) instead of a jump.
Creature mode under Vines in sacengine (checked): `graspingVines` (~12191) and `updateGraspingVines` (~19779) do not change the mode, they only increase/decrease `numVines` (`fixed`). In `updateCreatureState` (~15817), for idle/moving modes: `newMode = (no movement direction and speed == 0) || fixed ? idle : moving`, switching at the end of the animation loop or after ≥ 0.1 s (6 frames). So a vined creature is in `idle`, which is in the `isRegenerating` list, and after removing `regenerationBlocked` it will regenerate as in the original. Casting while moving turns into stationary casting (~16087), also without regeneration, as in the original. The only difference: the original switches to standing on the next tick, sacengine after ≥ 6 frames (0.1 s).

## Off topic (noticed along the way)
- sacengine `updatePlague` (~23006) poisons with `regenerationFactor` hard-coded to 1.0, so Plague does not disable regeneration. In the original every `POIS` poison zeroes regeneration (and `plag` has `freg` 0 anyway).

## Not checked
- Manalith mana range and side conditions in the exe (taken from SacrificeModern).
- Whether familiar and Zyzyx can be petrified.
