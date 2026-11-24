# Melee attacks on wizards: is the wizard pushed? sacengine vs original (2026-11-24)

Sources: sacengine (tg-2) 62c1b23, SacrificeModern (andrei-cb) d0f826b, `Sacrifice.exe` (884,736 bytes, Patch 3), recordings `melee_push_ne.mp4` (sacengine) and `melee_push_original_1.mp4` (original).

## Short answer
- **Original:** a wizard standing still and being hit in melee does not move at all.
- **sacengine:** the wizard gets a short jolt away from the attacker on every swing.
- Cause in sacengine: the collision fixup pushes the standing wizard whenever the attacker's animated hitbox (its arm during the swing) overlaps the wizard's hitbox.
- Why the original does not push is not confirmed yet (see "Open question").

## What the recordings show
Method: background motion measured with phase correlation at 30 fps on a patch of ground outside the HUD and the creatures (frames scaled to 960 px width, sensitivity ~0.15 px/frame). In both games the camera keeps the wizard at the same screen position, so background motion = wizard motion.

**sacengine (`melee_push_ne.mp4`)**
- A jolt on every swing: 1.7 s, 3.3 s, 4.95 s, 6.5 s, 8.2 s (every ~1.6 s, the attack rhythm).
- Each jolt lasts ~0.1 s (3–5 frames) and moves ~0.7–1 px; ~5 px in total over 8 s (≈5% of the wizard's on-screen height).
- Direction: away from the attacker.
- Frames at 3.3–3.4 s show the attacker's extended arm entering the wizard exactly when the jolt happens.

**Original (`melee_push_original_1.mp4`)**
- During the fight (2.5–11 s) the background does not move (0.0–0.1 px).
- Only two movements:
  - 2.37–2.40 s: ~1.3 px, when the creature walks up to the wizard. This fits a single collision push at first contact.
  - 5.0–5.2 s: camera rotated by the player (the wizard stays at the same screen spot, the background rotates). Not a push.

Caveat: it is not certain that both recordings use the same creature and the same wizard (sacengine lighting is very dark). A 1:1 recording of the same pair in both games would make the comparison stronger.

## Original (verified in Sacrifice.exe)
- **The melee blow itself never pushes.** CreatureDoAttack 0x455bb0 → target sweep 0x46a940 → melee_hit 0x4596e0: no calls to push functions and no velocity writes.
- The only push that can affect a wizard in melee is the generic collision push in the motion integrator:
  - Integrator 0x469760 (called from the creature update at 0x45b288; also a creature vtable entry, slot 0xbc). It has no early exit for zero velocity, so it also runs for a standing wizard.
  - Collision 0x469e20 (vtable +0xc0): candidate list from 0x468cf0, then a pair test for each candidate.
  - Pair test 0x469ff0 (vtable +0xc4): uses the object's collision boxes (+0x270 count, +0x278 records of 0x1c bytes), Minkowski sum via 0x401d70; a hit counts when the fraction is in [0, 1].
  - If the mask has 0x40000000 (creatures and wizards: 0xc0004015) and the fraction == 0.0 (already overlapping), push_from 0x44dc00 is called with target speed 1/12 units/tick (5 units/s), horizontal only, away from the other object's position. Skipped when motion flag 0x10000 is set.
  - The push is a velocity (v += (target − v)·0.5), with ground friction; a single event on the ground moves ~0.065 units.
- Wizards use the same motion model as creatures (in SacrificeModern the same CreatureActor class).
- **Collision boxes follow the animation** (Game3d.dll SAXSINSTANCE::UpdateBBox 0x160287c0): with dynamic bounds on (+0x178 & 1), the box is rebuilt from bones flagged in the SAXS descriptor (bone flag bit 1). For the earth orc and the goblin these are both feet, the head and both forearms. Each flagged bone contributes 6 points. SacrificeModern notes that the original reads the bone matrix by columns (m[axis], m[axis+3], m[axis+6]).

## sacengine (state.d `positionAfterCollision` ~16313)
- Fixup phase: the hitbox is grown by 0.1; for every obstacle the overlap along the axis of smallest overlap is summed, capped at `collisionFixupSpeed` = 5 units/s (0.083/frame), and applied **directly to the position** (moveOnGround on the ground). No velocity, no inertia.
- Runs for every creature in `isReactingToObstacles` modes, which include `idle`, `casting` and `stationaryCasting`, so it also moves a wizard who is standing still.
- Obstacle boxes in the proximity structure come from `relativeHitbox`, i.e. the current animation frame (sacobject.d `hitbox`: small/large/largeZ/largeZbot built from `hitboxBones`).
- Result: when the attacker swings, its arm extends its hitbox into the wizard's box, and the fixup pushes the wizard for as long as the boxes overlap, i.e. the ~0.1 s jolts seen in the recording.

## Open question: why the original does not push
From the code alone the original should push too (the integrator runs for a standing wizard, the boxes are animated, overlap gives fraction 0, and that calls push_from). The recording shows it does not, so during the swing the boxes apparently never overlap in the original. Candidate reasons (not verified):
1. The original box does not reach as far with the arm: only 6 points per bone, and the column-order matrix read may point the extent along a different axis than sacengine's large hitbox, which includes the whole fist.
2. The collision records at +0x270/+0x278 are not refreshed from the pose on every simulation tick (e.g. only when the model is rendered).
3. Something else.

Next step: find in the exe what writes the creatures' +0x270/+0x278 collision records and when.

## Fix direction for sacengine
Target behaviour: a melee swing must not move the wizard. A single small bump on first contact (when the attacker walks into the wizard) is fine. Possible approaches, depending on which candidate above turns out true:
- do not apply the fixup to an object that is not moving itself (closer to the original, where only the moving object is pushed), or
- use a hitbox for collision that does not grow with the attack animation.

## Not verified
- The reason above.
- The hitbox type (small/large) of the attacking creature in sacengine.
- Collision bones checked only for the earth orc and the goblin.
- Whether the heavy hit (≥ ¼ max HP, FSM event 7, DAMG state) moves the target through its animation.
