# Oil ignition and damage vs. the original game (Troggs, fire attacks)

2026-11-24 · sacengine commit 62c1b23

## Summary

Oil damage against Troggs in sacengine (commit 62c1b23) matches the original game. Two fire attacks ignite oil in the original but not in sacengine: the Warmonger's gatling shots (`fhga`) and the Volcano's initial ground-zero blast (`clov`). The fix is likely one `DamageMod.ignite` flag in each call.

Sources: `Sacrifice.exe` and `Spells.wad` (retail install with Patch 3), cross-checked with SacrificeModern (andrei-cb). Not tested in game.

## Data (Spells.wad)

| Record | Fields |
| --- | --- |
| Oil, `@ABI.SCAT/afck.SPEL` (tag `kcfa`, proc ViscousOilRun) | amount 500, damage type 0x24, damageRange 5, effectRange 15, duration 15 s |
| Trogg, `Cear.SCAT/orcy.CRE8` (tag `ycro`) | HP 960; resistances: melee 1000, direct spell 0, splash spell 0, direct ranged 900, splash ranged 900 |

Damage type 0x24: bits 0x70 = 0x20 is the fire element, low 3 bits = 4 selects the splash ranged resistance. Troggs have spell resistance 0, so spells do no damage to them, but oil hits them for 90%.

## How oil works in the original exe

1. **Resistance first.** 0x454950 calls the mitigation (creature vtable +0xf4 = 0x459890), then fires the slot 0 damage watchers. There is no check that the amount is above 0. Mitigation: amount · res[type & 7] · (0.4 for melee, else 1) · 0.001, with indices 0 melee, 1 direct spell, 2 splash spell, 3 direct ranged, 4 splash ranged.
2. **Ignition.** The oil watcher 0x4131d0 checks (type & 0x70) == 0x20 and (type & 7) != 0, i.e. fire and not melee. It then doubles the already mitigated amount of that hit and moves the oil effect to stage 16.
3. **Burst.** At stage 16 the oil effect calls 0x44d7e0 → 0x44d810 (0x4134ff) with source = the oil effect at the victim's center, direct target = the victim, type 0x24, amount 500, radius = damageRange 5. The victim takes the full 500 before resistance; others within 5 take int(500 · (1 − d/5)). These are fire hits too, so they ignite other oiled units, and those hits are doubled.
4. **Two damage paths.** Creature vtable +0x7c = 0x459950 is a normal hit and runs the watchers; 0x44d7e0 / 0x44d810 go through it. Vtable +0x78 = 0x454810 is damage over time: it applies resistance and adds to an accumulator at +0x43c, with no watchers, so it never ignites oil.

## Oil vs Troggs: matches

sacengine gives the same numbers as the original. `igniteOil` calls `dealSplashRangedDamageAt(0, ability, damageRange, center of the victim, DamageMod.ignite)`, so the splash ranged resistance applies and neighbours within 5 take linear falloff, with the same ignition chain and ×2. The ignition check in `dealRawDamage` runs after resistance, so a fire spell that does 0 damage to a Trogg still ignites the oil. The burst does not double itself, because the status is already `ignited`.

| Case (oiled Trogg, 960 HP) | Original | sacengine |
| --- | --- | --- |
| Igniting hit from a fire spell (resistance 0) | 0 × 2 = 0, oil ignites | same |
| Oil burst on the victim | 500 · 0.9 = 450 | same |
| HP left after a Fireball | 510 | same |

The only difference is rounding: the original uses integers, sacengine floats (at most 1 HP).

## Which fire attacks ignite oil

All 19 SPEL records with a fire, non-melee damage type were checked: which damage path the exe uses, and whether sacengine passes `DamageMod.ignite`. Two differ; the rest match.

| Attack (SPEL, type) | Original exe | Ignites (orig.) | Ignites (sacengine) |
| --- | --- | --- | --- |
| **Warmonger gatling** (`fhga`, 0x23, 80) | 0x416c30 → 0x4176f0 → shot 0x417590 → 0x44d7e0 | **yes** | **no**: `warmongerShootSingle` uses `DamageMod.none` |
| **Volcano ground-zero blast** (`clov`, 0x22, radius 50) | VolcanoRun 0x42a710 → 0x44d810 at 0x42a94f | **yes** | **no**: `updateVolcano` (state.d ~24379) uses `DamageMod.none` |
| Volcano lava balls (`clov`) | 0x429c30 → 0x429a60 → 0x426e50 → 0x44d7e0 | yes | yes |
| Bombard (`cfba`, 0x24) | impact 0x419050 → 0x44d7e0; ground fire afterwards via +0x78 | yes | yes |
| Spitfire (`fhba`, 0x23) | HailOfEarth callback 0x41a660 → 0x44d7e0 | yes | yes |
| Flame minion (`fkba`, 0x23) | KlapponFireRun 0x41aea0 → 0x44d7e0 | yes | yes |
| Oil (`kcfa`, 0x24) | burst 0x4134ff → 0x44d7e0 | yes | yes |
| Steam cloud (`goc@`, 0x22) | 0x440920 → 0x44d7e0 | yes | yes |
| Explosion (`pxef`, 0x22) | PyroExplosionRun 0x426910 → 0x44d7e0 | yes | yes |
| Fireball (`brif`), Rain of fire (`rifr`) | shared impact 0x426e50 → 0x44d7e0 | yes | yes |
| Dragonfire (`rifd`, 0x22) | pass-through hit 0x425d70 → vtable +0x7c; nearby burn via +0x78 | yes | yes, but via a splash on direction change |
| Tickferno (`sfba`), Phoenix (`rdra`) | `thit` = `heat`: branch 0x411d11 skips 0x44d810, burn over time | no | no (commented in sacengine) |
| Pyromaniac (`lffa`) | 0x415d90 → +0x78 | no | no |
| Rings of fire (`rapf`) | 0x4258a0 → 0x4256b0 → +0x78 | no | no |
| Fireform (`mrff`) | 0x427590 → 0x4272c0 → +0x78 | no | no |
| Firewall (`lwrf`) | 0x4282b0 → 0x428030 → +0x78 | no | no |
| Magnifryer (`yrfm`) | MFryRun 0x43d770 → +0x78 at 0x43df7c | no | no |
| Mini volcano (`clvm`) | 0x43e3e0 deals no damage | no | not applicable |

Example of difference 1 on an oiled Trogg: in the original the first Warmonger bullet does 80 · 0.9 · 2 = 144 and the oil then bursts for 450. In sacengine the Warmonger never sets the oil off.

## Proposed fix and other notes

Add `DamageMod.ignite` in two places in `source/state.d`, as flame minion and fireball already do. Not done yet, not compiled.

```diff
 // warmongerShootSingle, line 13239
-dealRangedDamage(end.id,rangedAttack,attacker,side,direction,DamageMod.none,state);
+dealRangedDamage(end.id,rangedAttack,attacker,side,direction,DamageMod.ignite,state);
 // updateVolcano, line 24379
-dealSplashSpellDamageAt(0,spell,volcanoGroundZeroRadius,wizard,side,position,DamageMod.none,state);
+dealSplashSpellDamageAt(0,spell,volcanoGroundZeroRadius,wizard,side,position,DamageMod.ignite,state);
```

SacrificeModern (andrei-cb) differs from the exe in two places, so it is not a safe reference here:

- `run_oil_affect` calls `spell_damage` with radius 0, so only the victim is hit. The exe passes damageRange 5; sacengine is right.
- `run_mfry` (Magnifryer) uses `spell_damage`, which would ignite oil. The exe uses the over-time path (+0x78), so it does not.

Open points:

- Nothing here was tested in game.
- Tickferno and Phoenix: the `heat` branch in the exe (0x411bd6 → 0x411d11) skips 0x44d810. The +0x78 call at 0x412253 was not traced to the end, but it agrees with SacrificeModern and the existing sacengine comment.
- Outside the Troggs question: with friendly fire off, sacengine returns before the ignition check for allied attackers. The original scales friendly damage (`fdam`, ×0.5) and still runs the watchers. Not checked further.
