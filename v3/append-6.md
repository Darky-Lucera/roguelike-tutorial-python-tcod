# Appendix 6 Draft: Designing Equipment with Personality

!!! note "Draft notes"

    This document is a parking place for future ideas. It is not part of the main tutorial flow yet, and the details below should be revisited and balanced before being turned into a finished appendix.

[Part 13](part-13.md) closes with a claim that mirrors Part 12's: the boring way to grow the player's power is to scale numbers (a sword that is a bigger dagger), and the interesting way is to give each piece a *personality*, a distinct decision it changes for the player. Part 13 proves the point by *not* doing it: it ships a clean stat ladder on purpose, because its real subject is the equipment *system*, not the arsenal.

This appendix is where that arsenal can grow without bloating the chapter. The main tutorial stays a thin trunk that teaches the *system* (`Equippable` data, two slots, bonuses through the `Fighter` properties); here we collect *content*: weapons and armor whose value is a behavior, not a number. These cross the line the chapter drew deliberately: a piece earns its own code when it needs different *behavior*, not different *numbers*.

## The design lens

Before adding a weapon, finish this sentence: *this weapon makes the player...* If the only honest ending is "...hit for more", it is a number, and it belongs in `factories.py` as one more template, not here. A good entry forces a *decision*: fight in the open or in a corridor, close the gap or keep distance, spend a turn to control or to damage. "Stronger" is the boring axis; "what new decision does this create?" is the one that adds depth.

Armor is the deliberate exception. As the chapter argues, personality lives in *behavior*, and wearing armor is passive: it changes a number and does nothing else. Most armor should stay that way, the steady baseline that makes the weapons' tricks legible. Armor earns an entry here only when it becomes partly *active* (it reacts when you are hit), and even then it is borrowing the weapon's idea.

## Reusable patterns

Most behaviors are one of a handful of techniques, reused from systems the tutorial (or these appendices) already built:

- **An action hook on the piece.** Today `Equippable` is pure data. A behavioral piece gains an optional method that runs at a moment in combat: `on_attack` for a weapon (after its wielder lands a hit), `on_hit` for armor (when its wearer takes one). This is the same split the consumables use: plain data for the common case, a method for the few that *act*. Whether that method lives on an `Equippable` subclass or on a separate effect object is the main open question below.
- **Area effects (cleave).** Reuse the area logic from the Fireball scroll (Part 9): the actors within a radius of a center. An axe applies its damage to every actor adjacent to the wielder, not just the target.
- **Status effects (stun).** Reuse the confusion machinery (Part 9) and the [Combat Effects appendix](append-2.md): a "stunned" status that consumes the target's next turn or two. A war hammer applies it on a hit, probably with a chance rather than every swing.
- **Ranged attack and line of sight.** The same genuinely new system the archers need in the [enemies appendix](append-5.md): a `ranged_attack` path on `Fighter` plus a line-of-sight check. A bow turns the *player* into that archer. Build it once, share it both ways.
- **On-hit reaction (active armor).** The defensive mirror of `on_attack`, applied where `Fighter` resolves incoming damage. Thorns reflect a fraction back at the attacker; regeneration reuses the troll's `hp`-over-time plumbing (Part 12 Exercise 3); resistance needs a damage-type system, the one real new dependency in the list.

## The cast (current and proposed)

Already in Part 13 (do not duplicate here when this is finished):

- **Dagger, Sword** (main): the weapon ladder, attack bonus only. The honest baseline; keep them plain.
- **Leather Armor, Chain Mail** (main): the armor ladder, defense bonus only. The steady passive axis everything else is measured against.

Proposed entries (to design and balance later):

### Axe (cleave)

Hits every actor adjacent to the wielder, not only the chosen target. Forces a positioning decision the dagger never did: a corridor (one enemy at a time) now *wastes* the axe, while standing in the open among three foes is where it shines. Pairs naturally against the swarming early monsters. Pattern: area effect + `on_attack` hook.

### War Hammer (stun)

Trades raw damage for control: a chance to stun on hit, costing the target its next turn. Forces "do I need to buy a turn?", to break off, to reposition, to let a scroll come off cooldown. The defensive weapon for a melee build. Pattern: status effect + `on_attack` hook. **Caveat:** stun is powerful and swingy; lean on a modest chance and a one-turn duration before tuning up.

### Bow (ranged)

Strikes at a distance, turning the player into the archer from Appendix 5. Forces "can I fight before they reach me?" and rewards line-of-sight play. The first weapon whose question is about *distance* rather than position or tempo. Pattern: ranged attack + line of sight. This is the heaviest entry, because the system does not exist yet.

### Spiked Armor (thorns), active

The exception that proves the rule: armor with a behavior. Reflects a fraction of melee damage back at the attacker. Forces nothing on the player directly, but rewards a melee, take-the-hits build and quietly punishes the enemies that swarm. Pattern: `on_hit` reaction. Keep it rare; most armor stays passive.

### Seeds (undefined)

- **Two-handed weapon**: occupies a slot but forbids a second (no shield while wielding it); a bigger bonus bought with flexibility. Needs the slot model to say "this blocks that".
- **Shield**: a third slot rather than a weapon, a small defense bonus, maybe a block chance. Tests whether the two-slot model generalizes (the `getattr`/`setattr` design from the chapter was built for exactly this).
- **Cursed gear**: cannot be unequipped once worn, or carries a hidden penalty revealed on equip. Needs an identify/curse flag.
- **Throwing weapon / elemental ward (resistance)**: both wait on systems that do not exist yet (consumable-like equipment, damage types).

## Open questions for later

- **Subclass or effect object?** Does a behavioral piece subclass `Equippable` (a `class Axe(Equippable)` with an `on_attack`), or does `Equippable` keep being data and carry an optional effect object (mirroring how `Item` carries a `Consumable`)? The second keeps the data/behavior split the rest of the codebase uses; decide before writing any of the above.
- **Where the ranged system lives.** Shared with the archers in Appendix 5, so it is not really an equipment topic. It may deserve its own short section, or a place in the main tutorial, before either the bow or the archer can exist.
- **Armor balance vs the chapter's stance.** Active armor contradicts "armor is the honest baseline" if overused. Keep it to one or two exceptional pieces; the bulk of armor should stay a number.
- **Balance:** every entry here needs the usual "play it ten times per floor" tuning before it is more than a sketch.
- Once this appendix has real content, add a pointer to it from the end of Part 13's "A toolkit, not a power ladder" section. Holding that link until then, so readers are not sent to a draft.
