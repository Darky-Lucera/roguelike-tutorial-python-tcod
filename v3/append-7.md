# Appendix 7 Draft: Using Graphics Instead of Characters (WIP)

!!! note "Draft notes"

    This document is a parking place for future ideas. It is not part of the main tutorial flow yet, and the details below should be revisited and balanced before being turned into a finished appendix.

Part 1 ends with a quick trick: point a free codepoint at a tile in the sheet and draw `chr(500)` instead of `"@"`. That is enough to see a sprite, but it does not scale to a full game with dozens of entities, monsters that face left and right, and a switch to flip the whole game back to ASCII.

This appendix turns that trick into a small, switchable system.

---

## How a tilesheet maps to the screen

A tileset is an image cut into a grid. Each cell is one tile. tcod never thinks in "letters": it thinks in **codepoints**, and a codepoint is just an index into the tileset. `CHARMAP_TCOD` is the table that says "codepoint 64 (`@`) lives at this cell, codepoint 35 (`#`) at that cell", and so on, for codepoints `0`-`255`.

`tileset.remap(codepoint, column, row)` rewrites one entry of that table: it makes a codepoint draw the tile at `(column, row)` of the sheet, counted from the top-left and zero-based. Because the character map only fills `0`-`255`, any codepoint at `256` or above is free real estate we can assign to sprites.

!!! info "Filling free cells vs. extending the sheet"
    `CHARMAP_TCOD` does not fill all 256 cells: the lower rows of the original DejaVu sheet are empty. The example sheet in Part 1 paints its sprites into those free cells, so nothing from the base font is lost. The `32` and `8` passed to `load_tilesheet` are the tile counts across and down. When you need more sprites than the free cells can hold, make the sheet *taller*: keep the original rows untouched, add new rows below, and raise the row count to match.

---

## A `sprites` module

Collect every sprite decision in one place: `game/constants/sprites.py`. Two pieces live here, a flag and a set of named codepoints, plus a function that wires them to the sheet.

```python
from __future__ import annotations

import tcod

# Flip the whole game between sprites and characters.
USE_SPRITES: bool = True

# Named codepoints. Values stay above 255 so they never collide with the
# TCOD character map. Group them by kind to keep the sheet layout readable.
PLAYER = 500
ORC    = 501
TROLL  = 502

DAGGER = 700
SWORD  = 701
POTION = 702


def init_sprites(tileset: tcod.tileset.Tileset) -> None:
    """Point each sprite codepoint at its tile (column, row) in the sheet."""
    tileset.remap(PLAYER, 0, 5)
    tileset.remap(ORC,    1, 5)
    tileset.remap(TROLL,  2, 5)

    tileset.remap(DAGGER, 0, 7)
    tileset.remap(SWORD,  1, 7)
    tileset.remap(POTION, 2, 7)
```

!!! tip "Define the id first, then remap"
    Declare the named codepoint (`PLAYER = 500`) before you remap it. It is safer than scattering magic numbers across `init_sprites`: the constant is the single source of truth, the remap only says *where* on the sheet that tile sits, and entity definitions refer to `sprites.PLAYER` instead of a bare `500`.

<!-- TODO: full codepoint table (all monsters, items, equipment, stairs, corpse) once the final sheet layout is frozen. -->

---

## Wiring it into `main.py`

Load the extended sheet and call `init_sprites` once, right after the tileset is created and before the context opens:

```diff
     tileset = tcod.tileset.load_tilesheet(
-        Path(__file__).parent / "res" / "dejavu12x12_gs_tc.png",
+        Path(__file__).parent / "res" / "dejavu12x12_gs_tc_ex.png",
         32,
         8,
         tcod.tileset.CHARMAP_TCOD,
     )
+    init_sprites(tileset)
```

The remap mutates the tileset in place, so every console that uses this tileset draws the sprites from now on.

---

## Do not tint sprites

`console.print(..., fg=color)` multiplies each tile by the foreground color. For characters that is a feature: one white `o` glyph becomes a green orc or a red one just by changing `fg`. For a full-color sprite it is a bug: the art gets stained.

When sprites are on, force the entity color to white (white is the neutral multiplier, so the tile shows its own colors). In `Entity.__init__`:

```diff
-        self.color = color
+        self.color = colors.WHITE if sprites.USE_SPRITES else color
```

!!! question "Why white and not 'no color'?"
    `console.print` always multiplies by *some* `fg`. There is no "skip tinting" flag, so the way to get the original pixels back is to multiply by white, which is `(255, 255, 255)` and leaves every channel unchanged. This is the same reason white tiles in a font can be recolored to anything, while a colored tile can only be darkened.

---

## Telling a codepoint from a character

Once you mix the two worlds, an entity's glyph is either a real character (`"#"`, a `str`) or a sprite codepoint (`500`, an `int`). `console.print` wants a one-character string, so convert the integer before drawing. In `GameMap.render`:

```diff
-                console.print(entity.x, entity.y, entity.char, fg=entity.color)
+                glyph = chr(entity.char) if isinstance(entity.char, int) else entity.char
+                console.print(entity.x, entity.y, glyph, fg=entity.color)
```

<!-- TODO: decide whether the int/str split stays in render or moves to a property on Entity (e.g. entity.glyph). A property keeps render clean and centralizes the rule. -->

---

## Facing: the flip problem

tcod copies each tile to the screen exactly as it sits in the sheet. It never rotates or mirrors. A hero facing left and the same hero facing right are therefore *two different tiles*, and you draw whichever matches the current facing.

The plan:

- Reserve a second block of codepoints for the mirrored art (`PLAYER = 500`, `PLAYER_FLIPPED = 600`), painted into their own rows of the sheet.
- Store a `flip: bool` on the actor: `True` means "draw the mirrored tile". We track *flip*, not facing, because we cannot assume which way the base art looks. The artist decides that; `flip` just toggles whenever a horizontal move goes against the base orientation.
- At render time, pick the base or flipped codepoint based on `flip`.

```python
# sketch, not final
PLAYER         = 500
PLAYER_FLIPPED = 600

def sprite_for(entity) -> int:
    if entity.flip:
        return entity.char + 100   # 500 -> 600
    return entity.char
```

!!! info "Why duplicate instead of mirror, even when you could"
    Most 2D games keep separate left and right art on purpose. Light hits a character from one side, so highlights and shadows sit on fixed pixels. Mirroring would make the shine jump to the other side every time the character turns, which reads as wrong. Duplicating the art (or hand-tuning each direction) keeps the lighting consistent.

<!-- TODO: the +100 offset is fragile. Replace with an explicit flipped-codepoint map, or a pair dataclass, so the relationship is data, not arithmetic. -->

---

## Constraints and what comes next

- **One tile per cell.** Every sprite must fit a single grid cell, same size as the font. Multi-tile creatures and large sprites need extra work and are out of scope here.
- **Layering two graphics is the hard part.** Drawing a sprite *on top of another sprite* (a floor tile under a creature, an item glow behind an icon) is not straightforward in tcod, because a cell holds one tile. It is possible, and it gets its own appendix (advanced tcod techniques), with worked examples.
- **Animation** (cycling tiles per frame), **blending**, and **partial transparency** also belong in that advanced appendix.

---

## Choosing characters or sprites

There are two reasonable designs, depending on whether you ever need to switch at runtime.

**Option A: one set, no switch.** Keep the single `sprites` module from above. Each constant holds the sprite codepoint, with the character it replaces written as a comment:

```python
PLAYER = 500  # "@"
ORC    = 501  # "o"
DAGGER = 700  # "/"
```

This is the simplest design. The game is always graphical, and `chr(500)` is what reaches the screen instead of `"@"`. Going back to characters means editing the constants by hand.

**Option B: two sets, switchable.** When you want a single flag to flip the whole game, use two sets of constants with the same names: one with characters, one with sprites. The sprite set *inherits* from the character set and overrides only the names that actually have art, so anything without a sprite falls back to its character automatically.

```python
class CharGlyphs:
    PLAYER = "@"
    ORC    = "o"
    DAGGER = "/"

class SpriteGlyphs(CharGlyphs):
    PLAYER = 500
    ORC    = 501
    # DAGGER is not overridden, so it falls back to "/"

glyphs = SpriteGlyphs if USE_SPRITES else CharGlyphs
```

Entity definitions always read `glyphs.PLAYER`, never a bare value, so the same code works in both modes. With `USE_SPRITES = True` the player is codepoint `500`; with `False`, the whole game drops back to characters, and any entity still missing a sprite shows its letter either way.

---

## Summary

Graphics in tcod are not a different rendering path: they are the same tile system with the codepoints remapped. A single `sprites` module (a `USE_SPRITES` flag, named codepoints, and `init_sprites`) plus two tiny render rules (do not tint sprites, convert `int` codepoints with `chr`) is the whole base system. Facing needs duplicated art, and anything beyond one-tile-per-cell (layering, animation) is the subject of the advanced tcod appendix.

<!--
Open decision: pick one of the two designs in "Choosing characters or sprites".
- Option A: a single constant set holding sprite codepoints (no runtime switch).
- Option B: two inheriting glyph sets (CharGlyphs / SpriteGlyphs) toggled by USE_SPRITES.
Once chosen, make the whole appendix consistent with it (the earlier sprites module
and the tint/render rules assume the constants live in one place).
-->
