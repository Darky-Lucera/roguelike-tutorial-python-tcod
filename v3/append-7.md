# Appendix 7 Draft: Using Graphics Instead of Characters (WIP)

!!! note "Draft notes"

    This document is a parking place for future ideas. It is not part of the main tutorial flow yet, and the details below should be revisited and balanced before being turned into a finished appendix.

    Part 1 links here early, but the code below assumes pieces that do not exist yet at that point: the `Entity`/`GameMap` split (Part 2) and the `game/constants/` package (Part 5). Read it then as a preview of where the Part 1 trick leads, and come back to implement it once those chapters are behind you.

Part 1 ends with a quick trick: point a free codepoint at a tile in the sheet and draw `chr(0xE000)` instead of `"@"`. That is enough to see a sprite, but it does not scale to a full game with dozens of entities, monsters that face left and right, and a switch to flip the whole game back to ASCII.

This appendix turns that trick into a small, switchable system.

---

## How a tilesheet maps to the screen

A tileset is an image cut into a grid. Each cell is one tile. tcod never thinks in "letters": it thinks in **codepoints**, and a codepoint is just an index into the tileset. `CHARMAP_TCOD` is the table that says "codepoint 64 (`@`) lives at this cell, codepoint 35 (`#`) at that cell", and so on, for the ASCII characters plus the box-drawing, block, and arrow glyphs tcod's classic layout includes.

`tileset.remap(codepoint, column, row)` rewrites one entry of that table: it makes a codepoint draw the tile at `(column, row)` of the sheet, counted from the top-left and zero-based. Because `CHARMAP_TCOD` only maps the characters a game prints (ASCII plus a handful of box-drawing, block, and arrow glyphs, the highest around `0x2611`), most codepoints are unused.

Rather than hunt for gaps, take the sprite ids from the Unicode **Private Use Area** (`U+E000..U+F8FF`): a block Unicode reserves for custom glyphs and never assigns to real characters, so a sprite there can never collide with `CHARMAP_TCOD` or with text the game prints.

!!! info "Running out of private codepoints"
    The main Private Use Area, in Unicode's BMP (Basic Multilingual Plane), runs `U+E000..U+F8FF` and holds 6400 codepoints, much more than a roguelike usually needs.

    If you ever wanted tens of thousands more, Unicode has two further **Supplementary Private Use Areas**, `U+F0000..U+FFFFD` and `U+100000..U+10FFFD`, 65,534 codepoints each. All three are reserved for private use and never assigned to real characters, and all stay under `chr()`'s ceiling of `0x10FFFF`, so the same `chr(codepoint)` render path keeps working.

    Each range stops at `...FFFD` because its final two codepoints (`...FFFE` and `...FFFF`) are permanent *noncharacters*: reserved forever and never meant for interchange, which is why the private ranges end just before them.

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

# Sprite ids live in the Unicode Private Use Area, so they never collide with
# a real character. PUA marks its start; each sprite is a small offset from it.
PUA = 0xE000

PLAYER = PUA + 0
ORC    = PUA + 1
TROLL  = PUA + 2

DAGGER = PUA + 3
SWORD  = PUA + 4
POTION = PUA + 5


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
    Declare the named codepoint (`PLAYER = PUA + 0`) before you remap it. It is safer than scattering magic numbers across `init_sprites`: the constant is the single source of truth, the remap only says *where* on the sheet that tile sits, and entity definitions refer to `sprites.PLAYER` instead of a bare number.

!!! question "Why not store `chr(0xE000)` directly?"
    It is tempting to define `PLAYER = chr(0xE000)` so the constant already holds a drawable character, like the ASCII glyphs do. The problem shows up in `init_sprites`: `tileset.remap()` expects an **integer** codepoint, not a string.

    Remember that Python has no `char` type: a single character is just a `str` of length 1 (`chr(0xE000)` returns a `str`, and `PLAYER[0]` would be a `str` too). That string is not an `int`, and `int(chr(0xE000))` raises `ValueError` (it only parses digit strings, not "the code of this character").

    You would have to write `remap(ord(PLAYER), ...)` everywhere, converting back to the number you started from. Keeping the constant as a plain `int` is the honest representation: a codepoint *is* a number, `remap` takes it as-is, and the single conversion to a character happens once, at render time, with `chr`.

### One tile can serve many names

Not every entity needs its own art. A scroll looks like a scroll, whatever spell it carries, so a single `SCROLL` tile can back every scroll in the game. Declare the shared codepoint once, then point each item name at it:

```python
POTION = PUA + 5
SCROLL = PUA + 6

# One tile, many names: every scroll reuses the same SCROLL sprite.
HEALTH_POTION    = POTION
MANA_POTION      = POTION

BACKPACK_SCROLL  = SCROLL
CONFUSION_SCROLL = SCROLL
FIREBALL_SCROLL  = SCROLL
LIGHTNING_SCROLL = SCROLL
MAPPING_SCROLL   = SCROLL
DRAIN_SCROLL     = SCROLL
TELEPORT_SCROLL  = SCROLL
```

Because `SCROLL` is one codepoint, `init_sprites` remaps it a single time; the names above are aliases that all draw that tile. Assigning the constant, instead of repeating the raw codepoint on every line, keeps the shared-tile relationship explicit: move where `SCROLL` sits in the sheet and every scroll follows.

---

## Choosing characters or sprites

There are two reasonable designs, depending on whether you ever need to switch at runtime.

**Option A: one set, no switch.** Keep the single `sprites` module from above. Each constant holds the sprite codepoint, with the character it replaces written as a comment:

```python
PLAYER = PUA + 0
ORC    = PUA + 1
DAGGER = PUA + 3
```

This is the simplest design. The game is always graphical, and `chr(0xE000)` is what reaches the screen instead of `"@", "o" or "/"`. Going back to characters means editing the constants by hand.

**Option B: two sets, switchable.** When you want a single flag to flip the whole game, use two sets of constants with the same names: one with characters, one with sprites.

The sprite set *inherits* from the character set and overrides only the names that actually have art, so anything without a sprite falls back to its character automatically.

```python
class CharGlyphs:
    PLAYER = "@"
    ORC    = "o"
    DAGGER = "/"

class SpriteGlyphs(CharGlyphs):
    PLAYER = PUA + 0
    ORC    = PUA + 1
    # DAGGER is not overridden, so it falls back to "/"

current_glyphs = SpriteGlyphs if USE_SPRITES else CharGlyphs
```

Entity definitions always read `current_glyphs.PLAYER`, never a bare value, so the same code works in both modes. With `USE_SPRITES = True` the player is codepoint `0xE000`; with `False`, the whole game drops back to characters, and any entity still missing a sprite shows its letter either way.

Pick whichever design fits your game; the rest of this appendix keeps using the plain `sprites.PLAYER` form (Option A) to keep the listings short. If you go with Option B, read every `sprites.X` in what follows as `current_glyphs.X`.

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

## Tinting sprites: usually not what you want

`console.print(..., fg=color)` multiplies each tile by the foreground color. For characters that is a feature: one white `o` glyph becomes a green orc or a red one just by changing `fg`. For a full-color sprite it is usually the opposite of what you want: the art gets stained.

So, unless you are tinting on purpose (a hit flash, a status effect), set the entity color to white whenever its glyph is a sprite. White is the neutral multiplier, so the tile shows its own colors. Check what the entity's own `char` actually holds, not a global flag: under Option B, a character-mode fallback still needs its color even while sprites are enabled elsewhere. In `Entity.__init__`:

```diff
-        self.color = color
+        self.color = colors.WHITE if isinstance(char, int) else color
```

!!! question "Why white and not 'no color'?"
    `console.print` always multiplies by *some* `fg`. There is no "skip tinting" flag, so the way to get the original pixels back is to multiply by white, which is `(255, 255, 255)` and leaves every channel unchanged. This is the same reason white tiles in a font can be recolored to anything, while a colored tile can only be darkened.

---

## A `Graphic` for each entity

`render` still needs a one-character string, but an entity's `char` can be a real character (`"#"`, a `str`) or a sprite codepoint (`0xE000`, an `int`). The smallest fix is a property that calls `chr` only for codepoints. That works, yet it is only the *first* rule about how an entity becomes a tile: facing will choose between two tiles, animation will cycle through several. Rather than pile branches onto `Entity`, make "how do I become a character" its own small object, a `Graphic`, and give each entity one.

```python
class Graphic:
    """How an entity turns its drawing state into the character tcod prints."""

    @property
    def glyph(self) -> str:
        raise NotImplementedError


class StaticGraphic(Graphic):
    """One fixed tile: a character like "@", or a sprite codepoint like 0xE000."""

    def __init__(self, char: str | int) -> None:
        self.char = char

    @property
    def glyph(self) -> str:
        return chr(self.char) if isinstance(self.char, int) else self.char
```

`Entity` now owns a `Graphic` and exposes `glyph` by delegating to it. Every existing definition passes a bare `char`, so wrap that in the trivial `StaticGraphic` automatically; only special entities pass their own `Graphic`:

```diff
     def __init__(
         self,
         ...
-        char: str = sprites.UNKNOWN,
+        char: str | int = sprites.UNKNOWN,
+        graphic: Graphic | None = None,
         ...
     ) -> None:
         ...
-        self.char = char
+        # A bare char is wrapped in the trivial Graphic, so every existing
+        # entity definition keeps working unchanged.
+        self.graphic = graphic if graphic is not None else StaticGraphic(char)
+
+    @property
+    def glyph(self) -> str:
+        return self.graphic.glyph
```

`render` changes by one word, and never has to change again whatever glyph an entity uses:

```diff
-                console.print(entity.x, entity.y, entity.char, fg=entity.color)
+                console.print(entity.x, entity.y, entity.glyph, fg=entity.color)
```

!!! info "This is the Strategy pattern"
    `Graphic` is a *strategy*: an interchangeable object that captures one decision, here "which character do I draw right now". `render` depends only on the small `Graphic` interface, not on the concrete kind, so new behaviors arrive as new subclasses and the drawing code stays put.

### How the `glyph` scales

`StaticGraphic` covers almost every entity. When one needs more, it becomes a different kind of `Graphic`, and `render` keeps the exact line above, because it only ever asks for `entity.glyph`. Animation, for instance, is a `Graphic` that steps through a list of frames:

```python
class AnimatedGraphic(Graphic):
    def __init__(self, frames: list[int]) -> None:
        self.frames = frames   # one codepoint per animation frame
        self.frame = 0

    @property
    def glyph(self) -> str:
        return chr(self.frames[self.frame])
```

!!! warning "Keep `glyph` pure"
    The `glyph` property must only *read* state, never change it. Advancing `frame` (or the facing in the next section) belongs to update time: a `tick` called once per game turn, or on a clock in the main loop. A property that `render` may call many times, or not at all, is the wrong place for a side effect. Rendering should change nothing.

Composing these axes (animation per direction, eight-way facing, blending two tiles in one cell) builds on this same base and is the subject of the advanced tcod appendix. The next section wires the first extra axis: facing.

---

## Facing: the flip problem

tcod copies each tile to the screen exactly as it sits in the sheet. It never rotates or mirrors, so a hero facing left and the same hero facing right are *two different tiles*. You paint the mirrored art into its own codepoints and choose the right one at draw time.

The naive fix is a `flip` boolean and an `if` at every place that draws. That branch spreads through the code and never grows past two directions. With `Graphic` in place, facing becomes one more kind: a `DirectionalGraphic` that holds both tiles and decides between them.

First, give every `Graphic` a way to react to movement. The base does nothing, so static graphics ignore it for free; only graphics that care override it. Passing both `dx` and `dy` keeps the interface uniform for future four or eight-way art, even though a left/right flip reads only `dx`:

```diff
 class Graphic:
     """How an entity turns its drawing state into the character tcod prints."""

+    def face(self, dx: int, dy: int) -> None:
+        """React to a move. No-op by default; overridden when facing matters."""
+
     @property
     def glyph(self) -> str:
         raise NotImplementedError
```

`DirectionalGraphic` stores the two tiles and flips between them. We store `flip` ("show the mirror"), not a compass facing, because the base art may point either way; the artist decides, and `face` only toggles on horizontal moves:

```python
class DirectionalGraphic(Graphic):
    """Two tiles: the sheet art, and its mirror."""

    def __init__(self, base: int, flipped: int) -> None:
        self.base = base
        self.flipped = flipped
        self.flip = False   # True shows the mirrored tile

    def face(self, dx: int, dy: int) -> None:
        # Assumes the base art faces right; swap the comparison if it faces left.
        if dx < 0:
            self.flip = True
        elif dx > 0:
            self.flip = False

    @property
    def glyph(self) -> str:
        return chr(self.flipped if self.flip else self.base)
```

Before an actor can use it, reserve the mirrored tile in `sprites.py`, in whichever row of the sheet holds the flipped art (row 6, if row 5 holds the upright actors):

```diff
 PLAYER = PUA + 0
 ORC    = PUA + 1
 TROLL  = PUA + 2
+
+PLAYER_FLIPPED = PLAYER + 0x100
+ORC_FLIPPED    = ORC    + 0x100
+TROLL_FLIPPED  = TROLL  + 0x100

 DAGGER = PUA + 3
 SWORD  = PUA + 4
 POTION = PUA + 5
```

```diff
     tileset.remap(PLAYER, 0, 5)
     tileset.remap(ORC,    1, 5)
     tileset.remap(TROLL,  2, 5)

+    tileset.remap(PLAYER_FLIPPED, 0, 6)
+    tileset.remap(ORC_FLIPPED,    1, 6)
+    tileset.remap(TROLL_FLIPPED,  2, 6)
+
     tileset.remap(DAGGER, 0, 7)
```

A flipping actor declares its two tiles instead of a single `char`:

```python
player = Actor(
    ...
    graphic = DirectionalGraphic(sprites.PLAYER, sprites.PLAYER_FLIPPED),
)
```

Finally, wherever an entity moves, tell its glyph which way it went. Static entities ignore the call, so it is safe for everyone:

```diff
     def move(self, dx: int, dy: int) -> None:
+        self.graphic.face(dx, dy)
         self.x += dx
         self.y += dy
```

!!! info "Why duplicate instead of mirror, even when you could"
    Most 2D games keep separate left and right art on purpose. Light hits a character from one side, so highlights and shadows sit on fixed pixels. Mirroring would make the shine jump to the other side every time the character turns, which reads as wrong. Duplicating the art (or hand-tuning each direction) keeps the lighting consistent.

---

## Constraints and what comes next

- **One tile per cell.** Every sprite must fit a single grid cell, same size as the font. Multi-tile creatures and large sprites need extra work and are out of scope here.
- **Layering two graphics is the hard part.** Drawing a sprite *on top of another sprite* (a floor tile under a creature, an item glow behind an icon) is not straightforward in tcod, because a cell holds one tile. It is possible, and it gets its own appendix (advanced tcod techniques), with worked examples.
- **Animation timing and composition** belong in that advanced appendix. The `AnimatedGraphic` sketch above shows the shape; advancing frames on a clock, composing animation with facing, plus **blending** and **partial transparency**, are the deeper work.

---

## Summary

Graphics in tcod are not a different rendering path: they are the same tile system with the codepoints remapped. `tileset.remap` points a free id at a tile in the sheet, and the Unicode Private Use Area is where those ids come from, so they never collide with a real character. A `sprites` module collects them behind a `USE_SPRITES` flag and an `init_sprites()` function, organized either as flat constants or as two switchable sets, and forces `fg` to white so the art keeps its own colors.

Rather than branch on `int` versus `str` at every draw, each entity delegates to a `Graphic`: a small strategy object whose `glyph` property resolves to the character tcod prints. That one change is what lets facing and animation arrive as new kinds of `Graphic` (`DirectionalGraphic`, `AnimatedGraphic`) instead of new special cases in `render`. The whole system still assumes one tile per cell; layering two graphics, blending, and full animation timing go beyond that and are the subject of the advanced tcod appendix.
