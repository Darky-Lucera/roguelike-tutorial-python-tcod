# Appendix 5 Draft: Designing Enemies with Personality (WIP)

!!! note "Draft notes"

    This document is a parking place for future ideas. It is not part of the main tutorial flow yet, and the details below should be revisited and balanced before being turned into a finished appendix.

[Part 12](part-12.md) closes with a claim: the boring way to make a dungeon harder is to scale numbers (more hit points, more damage, more defense), and the interesting way is to give each monster a *personality*, a distinct decision it forces on the player. Part 12 proves the point with a small cast and then stops, on purpose: its real subject is the spawn system, not the bestiary.

This appendix is where that bestiary can grow without bloating the chapter. The main tutorial stays a thin trunk that teaches the *system*; here we collect *content*: enemy archetypes, each one a variation on patterns the tutorial already taught. New monsters rarely introduce new concepts, so they belong here as a recipe book rather than in the main path.

## The design lens

Before adding an enemy, finish this sentence: *this enemy makes the player...* If the only honest ending is "...lose more hit points", it is a number, not a personality. A good entry forces a *decision*: chase or let go, close or keep distance, burst it or whittle it, fight it or avoid it. "Harder" is the boring axis; "what decision does this force?" is the one that creates depth.

A healthy cast also *contrasts*. Two healers are interesting only if they heal in opposite ways (the troll runs and waits, the ghoul closes and bites). Two ranged enemies are interesting only if one holds its ground and the other backs away. Design enemies in pairs of opposites rather than in a single line from weak to strong.

## Reusable patterns

Most personalities are one of a handful of techniques, reused:

- **Threshold on `Fighter`.** A behavior that switches on below a fraction of max HP. `flee_threshold` (Part 6 Exercise 3) makes the troll run; the same shape with the sign flipped makes a **berserker** hit harder. A `should_X()` helper mirroring `should_flee()` keeps it readable.
- **Self-healing.** Two flavors on the same `hp` plumbing: **lifesteal** heals on a landed hit, inside `Fighter.melee_attack` (the ghoul); **regeneration** heals over time, ticked from the turn loop (the troll, only when idle). Both are capped by `max_hp` via the `hp` setter.
- **Temporary AI swap with revert.** `ConfusedEnemy` (confusion scroll) and the reversible `CowardEnemy` (Part 12 Exercise 3) both replace the AI, remember the previous one, and restore it on a condition. Any "the creature changes how it acts for a while" behavior fits here.
- **Ranged attack and line of sight.** Not yet in the tutorial as an enemy capability. An archer needs a way to strike at range (a `ranged_attack` path on `Fighter`) and a line-of-sight check, so it only fires when it can actually see the player. This is the one genuinely new system in the list below.
- **Distance-keeping AI (kiting).** Move *away* toward a preferred range and attack from there, the opposite of the chase-and-bump `HostileEnemy`.

## The cast (current and proposed)

Already in Part 12 (do not duplicate here when this is finished):

- **Orc** (main): the plain baseline, no twist, the honest fight everything else is measured against. Deliberately has no gimmick; keep it that way.
- **Troll** (main, plus Part 12 Exercise 3): weak, flees at low HP, regenerates while standing still, returns to the fight when fully healed or when cornered and provoked.
- **Ogre** (main): a wall. High HP, heavy hit, no trick. Forces "do not trade blows, use range or control".
- **Ghoul** (Part 12 Exercise 2): frail but high lifesteal. Forces "burst it down or stay out of reach".

Proposed entries (to design and balance later):

### Goblin (early)

The simplest coward: weak, flees at low HP, no regeneration. It is the pure example of the `flee_threshold` pattern, with the troll as its evolved cousin. **Caveat:** a plainly cowardly goblin poses almost the same question as the troll ("chase or let go?"), and two fleers on floor 1 make the early game samey. To give it its own question, make it a **thief**: it steals an item and bolts, so the decision becomes "chase it down to get your item back?" (the "thief" seed from Part 12's closing paragraph).

### Orc Berserker (deep)

The mirror of the troll. Where the troll flees when wounded, the berserker *rages*: below a `rage_threshold` it gains an attack bonus, so hurting it makes it more dangerous. Implementation: a `rage_threshold: float` on `Fighter`, a `should_rage()` helper mirroring `should_flee()`, and the bonus applied in `Fighter.melee_attack`. **Important:** this is a *new, deeper creature* (an enraged cousin of the orc), not a change to the plain orc. The plain orc must stay the baseline, or Part 12's closing section stops being true. Stats a notch above the orc; spawns on deep floors only. The pair reads as "two reactions to a wound: one runs, the other doubles down".

### Dark Elf / archer (the turret)

A stationary ranged attacker: it does not move, but it fires at the player on sight. Forces a positioning decision: close the gap under fire, or break line of sight. Needs the ranged-attack and line-of-sight pattern above. The first enemy that punishes walking in a straight line.

### Kiter (the turret, but worse)

An archer that *retreats* when the player approaches: ranged attack plus distance-keeping AI. You cannot simply walk it down; you must corner it, out-range it, or break line of sight. Effectively "ranged plus flee", and a natural capstone for this appendix because it combines two patterns.

### Seeds (undefined)

- **Skeleton**: undead filler waiting for a hook (reassembles? immune to something? rises once more when killed?).
- **Demon**: a deep, high-threat personality to be defined.
- Others from Part 12's closing paragraph: a monster harmless until you turn your back; one that splits in two when hit; one that does nothing but heal the monsters around it.

## Open questions for later

- Where to introduce the ranged-attack and line-of-sight system, since it is a real new capability, not a variation. It may deserve its own short section, or a place in the main tutorial before the archers can exist.
- Goblin: pure coward or thief? (Leaning thief, to avoid overlapping the troll.)
- Balance: every entry here needs the usual "play it ten times per floor" tuning before it is more than a sketch.
- Once this appendix has real content, add a pointer to it from the end of Part 12's "A cast, not a difficulty curve" section. Holding that link until then, so readers are not sent to a draft.
