# Appendix 7: Using Graphics Instead of Characters

Part 1 links here early, but the code below assumes pieces that do not exist yet at that point: the `Entity`/`GameMap` split (Part 2), the `game/constants/` package (Part 5), and the centralized `RES_DIR` (Part 10). Read it then as a preview of where the Part 1 trick leads, and come back to implement it once those chapters are behind you.

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
    `CHARMAP_TCOD` does not fill all 256 cells: the lower rows of the original DejaVu sheet are empty. The extended sheet from Part 1 paints its first sprites into those free cells, so nothing from the base font is lost. When you need more sprites than the free cells can hold, make the sheet *taller*: keep the original rows untouched, add new rows below, and raise the row count passed to `load_tilesheet` to match.

    Taller, never wider. `load_tilesheet` numbers tiles left to right, top to bottom, and assigns the codepoints in `CHARMAP_TCOD` to the tiles in that same order. `CHARMAP_TCOD` has 160 entries, enough for the first five rows of a 32-column sheet, which is exactly why the three rows below them were free in the first place. Any tile numbered past the end of the list gets no character, so rows added at the bottom leave the whole font intact, and the new tiles sit unassigned until `remap` points something at them. Widen the sheet instead and the numbering shifts under the charmap: with 40 columns, tile 32 is no longer the first cell of the second row (`@` in this layout) but the 33rd cell of the first, so every character past the first row lands on the wrong art.

    The tutorial's sheet already does both safe moves: rows 5 to 7 fill free cells, and five extra rows of directional art sit below the original eight, 13 in total. The facing sections at the end of this appendix put those rows to work.

---

## A `sprites` module

Collect every sprite decision in one place: `game/constants/sprites.py`. Two pieces live here: a set of named codepoints, and a function that wires them to the sheet.

```python
from __future__ import annotations

import tcod

# Sprite ids live in the Unicode Private Use Area (PUA), so they never collide with
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

There are two reasonable designs, depending on whether you want a flag that picks the mode at startup, or you are fine hand-editing constants instead.

**Option A: one set, no switch.** Keep the single `sprites` module from above. Each constant holds the sprite codepoint, with the character it replaces written as a comment:

```python
PLAYER = PUA + 0
ORC    = PUA + 1
DAGGER = PUA + 3
```

This is the simplest design. The game is always graphical, and `chr(0xE000)` is what reaches the screen instead of `"@", "o" or "/"`. Going back to characters means editing the constants by hand.

**Option B: two sets, switchable.** When you want a single flag to flip the whole game, declare two classes with the same names: one with characters, one with sprites.

```python
USE_SPRITES: bool = True

class CharGlyphs:
    PLAYER = "@"
    ORC    = "o"
    DAGGER = "/"

class SpriteGlyphs:
    PLAYER = PUA + 0
    ORC    = PUA + 1
    DAGGER = PUA + 3

CurrentGlyphs: type[SpriteGlyphs] | type[CharGlyphs] = SpriteGlyphs if USE_SPRITES else CharGlyphs
```

!!! question "Why doesn't `SpriteGlyphs` inherit from `CharGlyphs`?"
    A natural first instinct is to make `SpriteGlyphs` a subclass of `CharGlyphs`, overriding only the names that have art, so anything without a sprite falls back to its character automatically:

    ```python
    class SpriteGlyphs(CharGlyphs):
        PLAYER = PUA + 0    # overrides "@"
        # DAGGER is not overridden, so it keeps "/"
    ```

    This runs fine, but a type checker rejects it. `CharGlyphs.PLAYER` is typed as `str`; reassigning `PLAYER` to an `int` in the subclass changes the type of an inherited attribute, which mypy and pyright both flag as an incompatible override. Python itself does not check this, class attributes are just names in a namespace at runtime, but a type checker treats each name as carrying one fixed type across the whole hierarchy, and `str` and `int` are not that.

    The fix costs a few lines: declare every name in both classes, even where the sprite set has no fallback to offer. Each class is then self-contained and honestly typed, at the cost of the automatic fallback the inheritance version promised.

Entity definitions always read `CurrentGlyphs.PLAYER`, never a bare value, so the same code works in both modes. With `USE_SPRITES = True` the player is codepoint `0xE000`; with `False`, the whole game drops back to characters. "Both modes" means two separate runs, not a switch mid-game: `CurrentGlyphs` is decided once, when `sprites.py` is imported, and every factory reads it at that same moment. Flipping `USE_SPRITES` means changing the constant and restarting, not toggling a setting while playing.

!!! tip "Why `CurrentGlyphs` and not `current_glyphs`"
    `CurrentGlyphs` holds a class, not a value, so it plays the role of a type alias. PEP 8 asks for CapWords there, the same convention as a class name, rather than the `snake_case` used for ordinary variables. A naming linter (ruff's pep8-naming rules, for instance) flags a lowercase name in this position.

Pick whichever design fits your game. From here on, this appendix follows Option B, since that is what the reference implementation uses: every later snippet reads `CurrentGlyphs.X`, or `SpriteGlyphs.X` directly where facing specifically needs a sprite codepoint. If you picked Option A instead, there is only one set, so read every `CurrentGlyphs.X` and `SpriteGlyphs.X` below as a bare `sprites.X`.

A factory definition changes by one line:

```diff
 player = Actor(
-    char  = sprites.PLAYER,
+    char  = sprites.CurrentGlyphs.PLAYER,
     ...
 )
```

Every other entity in `game/entities/factories.py` follows the same pattern: wherever a definition reads `sprites.X`, it now reads `sprites.CurrentGlyphs.X`.

---

## Wiring it into `main.py`

Load the right sheet for the current mode, with its matching row count, and call `init_sprites` once, right after the tileset is created and before the context opens:

```diff
     tileset = tcod.tileset.load_tilesheet(
-        constants.RES_DIR / "dejavu12x12_gs_tc.png",
+        constants.RES_DIR / ("dejavu12x12_gs_tc_ex.png" if sprites.USE_SPRITES else "dejavu12x12_gs_tc.png"),
         32,
-        8,
+        (13 if sprites.USE_SPRITES else 8), # 5 Monster rows
         tcod.tileset.CHARMAP_TCOD,
     )
+    sprites.init_sprites(tileset)
```

The row count follows the same flag as the filename: the original sheet is 8 rows tall, the extended one is 13, and passing a count that does not match the image leaves tcod unable to cut it into tiles. The five extra rows hold directional monster art; the facing sections below put it to work.

!!! warning "Parenthesize the ternary"
    `/` binds tighter than `if ... else`. Without the inner parentheses, `RES_DIR / "a" if flag else "b"` parses as `(RES_DIR / "a") if flag else "b"`: the `False` branch would return the bare string `"b"`, never joined to `RES_DIR`. Wrap the whole conditional before joining it to the path.

The remap mutates the tileset in place, so every console that uses this tileset draws the sprites from now on.

---

## Tinting sprites: usually not what you want

`console.print(..., fg=color)` multiplies each tile by the foreground color. For characters that is a feature: one white `o` glyph becomes a green orc or a red one just by changing `fg`. For a full-color sprite it is usually the opposite of what you want: the art gets stained.

So, unless you are tinting on purpose (a hit flash, a status effect), set the entity color to white whenever its glyph is a sprite. White is the neutral multiplier, so the tile shows its own colors. Check what the entity's own `char` actually holds, not a global flag: an entity that deliberately keeps a literal character, a debug marker, say, still needs its own color even while the rest of the game runs on sprites. In `Entity.__init__`:

```diff
-        self.color = color
+        self.color = colors.WHITE if isinstance(char, int) else color
```

!!! question "Why white and not 'no color'?"
    `console.print` always multiplies by *some* `fg`. There is no "skip tinting" flag, so the way to get the original pixels back is to multiply by white, which is `(255, 255, 255)` and leaves every channel unchanged. This is the same reason white tiles in a font can be recolored to anything, while a colored tile can only be darkened.

---

## A `Graphic` for each entity

`render` still needs a one-character string, but an entity's `char` can be a real character (`"#"`, a `str`) or a sprite codepoint (`0xE000`, an `int`). The smallest fix is a property that calls `chr` only for codepoints. That works, yet it is only the *first* rule about how an entity becomes a tile: facing will choose between two tiles, animation will cycle through several. Rather than pile branches onto `Entity`, make "how do I become a character" its own small object, a `Graphic`, and give each entity one.

Create `game/entities/components/graphic.py`:

```python
from __future__ import annotations


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

`Entity` now owns a `Graphic` and exposes `glyph` by delegating to it. Every existing definition passes a bare `char`, so wrap that in the trivial `StaticGraphic` automatically; only special entities pass their own `Graphic`. In `game/entities/entity.py`, add the import:

```diff
+from game.entities.components.graphic import Graphic, StaticGraphic
```

and update `__init__`:

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

`Actor` overrides `__init__` with its own explicit signature instead of inheriting `Entity`'s, so it needs the same two edits, or `Actor(..., graphic=...)` raises `TypeError: unexpected keyword argument 'graphic'`:

```diff
     def __init__(
         self,
         *,
         ...
-        char: str = sprites.UNKNOWN,
+        char: str | int = sprites.UNKNOWN,
+        graphic: Graphic | None = None,
         ...
     ) -> None:
         super().__init__(
             ...
             char            = char,
+            graphic         = graphic,
             ...
         )
```

Passing both `char` and `graphic` is not an error, `graphic` simply wins: `char` is only consulted to build the fallback `StaticGraphic` when `graphic` is `None`.

!!! warning "Old saves and `graphic`"
    [Part 10](part-10.md) pickles every entity, and this section replaces `Entity.char` with `Entity.graphic`. A save written before this change unpickles into an entity that has `char` but no `graphic`, and the first render raises `AttributeError`.

    Either delete old saves, as Part 11 and Part 13 already made you do, or adapt the `__setstate__` migration trick from Part 10: translate the old `char` key into `graphic = StaticGraphic(char)` before the rest of `__dict__` loads.

This also retires the tinting rule from the previous section. `isinstance(char, int)` read `self.char`, which no longer exists: an entity built with `graphic=` directly never had a meaningful `char` to inspect, it keeps whatever default the parameter has, unrelated to what actually gets drawn. Fall back to the mode flag instead:

```diff
-        self.color = colors.WHITE if isinstance(char, int) else color
+        self.color = colors.WHITE if sprites.USE_SPRITES else color
```

The trade-off is real: an entity with a literal character no longer gets to opt out of the white tint just because its own glyph is text. If that precision matters for a particular entity, give `Graphic` an `is_sprite` property and check `self.graphic.is_sprite` instead of the global flag.

`render` changes by one word, and never has to change again whatever glyph an entity uses:

```diff
-                console.print(entity.x, entity.y, entity.char, fg=entity.color)
+                console.print(entity.x, entity.y, entity.glyph, fg=entity.color)
```

`GameMap.render` is not the only place `.char` reaches the screen, the inventory panel prints `item.char` too. Search the codebase for every `.char` read on an entity or item and replace it with `.glyph`. A type checker turns this from a hunt into a checklist: once `Entity` no longer declares `char` as an attribute, mypy or pyright flags every remaining `.char` access as unknown, so running one after the rename surfaces exactly the call sites still to fix.

!!! info "This is the Strategy pattern"
    `Graphic` is a *strategy*: an interchangeable object that captures one decision, here "which character do I draw right now". `render` depends only on the small `Graphic` interface, not on the concrete kind, so new behaviors arrive as new subclasses and the drawing code stays put.

### Swapping a `Graphic` after construction

`self.graphic` is a plain attribute, not read-only, so anything that changes what an entity draws can just assign a new one. `Fighter.die()` does exactly this to turn a fallen actor into a corpse. Add the import to `game/entities/components/fighter.py`:

```diff
+from game.entities.components.graphic import StaticGraphic
```

then, in `die()`:

```python
self.entity.graphic = StaticGraphic(sprites.CurrentGlyphs.CORPSE)
self.entity.color   = colors.WHITE if sprites.USE_SPRITES else colors.CORPSE
```

`render` needs no change: it only ever asks for `entity.glyph`, and `glyph` reads whichever `Graphic` is currently assigned. A corpse has no facing, so a plain `StaticGraphic` is enough either way, and `CurrentGlyphs.CORPSE` still follows the mode flag like any other plain `char`.

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

Composing these axes (animation per direction, blending two tiles in one cell) builds on this same base; animation itself is the subject of [Appendix 8](append-8.md). The next sections wire the first extra axis: facing, first as a left/right mirror, then across the full compass.

---

## Facing: the flip problem

tcod copies each tile to the screen exactly as it sits in the sheet. It never rotates or mirrors, so a hero facing left and the same hero facing right are *two different tiles*. You paint the mirrored art into its own codepoints and choose the right one at draw time.

The naive fix is a `flip` boolean and an `if` at every place that draws. That branch spreads through the code and never grows past two directions. With `Graphic` in place, facing becomes one more kind: a `DirectionalGraphic2` that holds both tiles and decides between them. The `2` in the name counts its directions: four-way and eight-way siblings join it before this appendix ends.

First, give every `Graphic` a way to react to movement. The base does nothing, so static graphics ignore it for free; only graphics that care override it. Passing both `dx` and `dy` keeps the interface uniform for future four or eight-way art, even though a left/right flip reads only `dx`:

```diff
 class Graphic:
     """How an entity turns its drawing state into the character tcod prints."""

+    def face(self, dx: int, dy: int) -> None:
+        pass    # No-op by default; overridden when facing matters.
+
     @property
     def glyph(self) -> str:
         raise NotImplementedError
```

`DirectionalGraphic2` stores the two tiles and flips between them. We store `flip` ("show the mirror"), not a compass facing, because the base art may point either way; the artist decides, and `face` only toggles on horizontal moves:

```python
class DirectionalGraphic2(Graphic):
    """Two tiles: the sheet art, and its mirror."""

    def __init__(self, base: int, flipped: int) -> None:
        self.base    = base
        self.flipped = flipped
        self.flip    = False   # True shows the mirrored tile

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
 class SpriteGlyphs:
     PLAYER = PUA + 0
     ORC    = PUA + 1
     TROLL  = PUA + 2
+
+    PLAYER_FLIPPED = PLAYER + 10
+    ORC_FLIPPED    = ORC    + 10
+    TROLL_FLIPPED  = TROLL  + 10

     DAGGER = PUA + 3
     SWORD  = PUA + 4
     POTION = PUA + 5
```

The gap of `10` leaves room for a few more upright entities before their flipped variants would collide with the next block. A larger game can reserve a whole block per category instead, actors, items, equipment, each starting at its own round number, so the roster has room to grow without renumbering anything.

```diff
     tileset.remap(SpriteGlyphs.PLAYER, 0, 5)
     tileset.remap(SpriteGlyphs.ORC,    1, 5)
     tileset.remap(SpriteGlyphs.TROLL,  2, 5)

+    tileset.remap(SpriteGlyphs.PLAYER_FLIPPED, 0, 6)
+    tileset.remap(SpriteGlyphs.ORC_FLIPPED,    1, 6)
+    tileset.remap(SpriteGlyphs.TROLL_FLIPPED,  2, 6)
+
     tileset.remap(SpriteGlyphs.DAGGER, 0, 7)
```

A flipping actor declares its two tiles instead of a single `char`, but only when there is art to flip between: there is no mirrored version of the character `"@"`. Build `DirectionalGraphic2` unconditionally and the failure is silent in character mode: `main.py` loads the plain sheet, but `init_sprites` still remaps `SpriteGlyphs.PLAYER` and its neighbors onto whatever sheet is loaded, since it does not check the flag either, so the actor draws whatever happens to sit at those cells instead of falling back to `"@"`.

(This is where Option A stops needing a translation note: there is only one glyph set, so skip `_directional2` below and build `DirectionalGraphic2(sprites.PLAYER, sprites.PLAYER_FLIPPED)` directly, with no flag and no fallback to consider.)

A per-actor `if sprites.USE_SPRITES else None` would work, but five monsters mean five chances to forget it. Centralize the decision instead, in `game/entities/factories.py`:

```diff
-from game.entities.components.graphic import DirectionalGraphic2
+from game.entities.components.graphic import DirectionalGraphic2, Graphic, StaticGraphic
```

```python
def _directional2(base: int,
                  flipped:  int,
                  fallback: str | int) -> Graphic:
    return DirectionalGraphic2(base, flipped) if sprites.USE_SPRITES else StaticGraphic(fallback)


player = Actor(
    ...
    graphic = _directional2(sprites.SpriteGlyphs.PLAYER, sprites.SpriteGlyphs.PLAYER_FLIPPED, sprites.CurrentGlyphs.PLAYER),
)
```

`CurrentGlyphs.PLAYER` is the right argument here, not `CharGlyphs.PLAYER`: every other entity already reads its plain glyph through `CurrentGlyphs`, so this keeps the same habit even though, in the branch where `sprites.USE_SPRITES` is true, the value it resolves to (a sprite codepoint) never actually gets used, `fallback` only matters in the other branch, where `CurrentGlyphs` has already collapsed to `CharGlyphs` for you.

!!! info "Why this stays out of `DirectionalGraphic2` itself"
    `StaticGraphic` already resolves `str` or `int` in its own `glyph` (the `isinstance` check from earlier), so reusing it for the fallback avoids duplicating that logic a second time inside `DirectionalGraphic2`. It also keeps every `Graphic` subclass ignorant of `USE_SPRITES`: `StaticGraphic` and `AnimatedGraphic` do not know the flag exists, and `DirectionalGraphic2` should not be the one exception. The mode decision belongs to whoever builds the entity, made once at construction, not to the strategy object itself on every `glyph` read.

    A variant worth knowing about: check `isinstance(fallback, int)` instead of `sprites.USE_SPRITES` directly, and `_directional2` would not need to import `sprites` at all, it would just react to the type of whatever `CurrentGlyphs.PLAYER` resolved to, the same idiom `StaticGraphic.glyph` already uses. Either reads fine; this appendix keeps the explicit flag since it says outright what is being decided.

`char` is no longer needed on this actor: `_directional2` always returns a real `Graphic`, sprite or fallback, so `graphic` is never `None` and `Entity.__init__` never reaches for `char`. Every other actor with its own facing art (the tutorial's monsters, if you gave them one) follows the same `_directional2(...)` call.

Finally, wherever an entity moves, tell its glyph which way it went. Static entities ignore the call, so it is safe for everyone:

```diff
     def move(self, dx: int, dy: int) -> None:
+        self.graphic.face(dx, dy)
         self.x += dx
         self.y += dy
```

!!! info "Why duplicate instead of mirror, even when you could"
    Most 2D games keep separate left and right art on purpose. Light hits a character from one side, so highlights and shadows sit on fixed pixels. Mirroring would make the shine jump to the other side every time the character turns, which reads as wrong. Duplicating the art (or hand-tuning each direction) keeps the lighting consistent.

### Facing in combat

Movement is not the only thing with a direction. A bump attack resolves through `MeleeAction` without ever calling `move()`, so an actor could strike left while its sprite kept looking right, and the victim could take the blow with its back turned. Both read wrong on screen. Fix both sides in `MeleeAction.perform`, just before the attack lands:

```diff
             return  # Both attacker and defender must be Actors to fight

+        entity.graphic.face(self.dx, self.dy)
+        target.graphic.face(-self.dx, -self.dy)
+
         entity.fighter.melee_attack(target)
```

The attacker turns into its own blow, and `(-dx, -dy)` is that same blow seen from the other side, so the target turns to meet it. Because `face` is a no-op on the base `Graphic`, both lines are safe whatever kind of graphic either side carries. Ranged attacks, the scrolls, still turn nobody; the same call fits wherever a target position is known, if your game wants it.

!!! question "Should the victim really turn?"
    Turning the defender is a game-feel choice, not a rule. Two fighters trading blows while facing each other is what the eye expects, and being hit is as good a reason as any to look at your attacker. A stealth game might keep the victim's facing untouched on purpose: a backstab reads as a backstab precisely because the victim never saw it coming. If that is your game, drop the second line.

---

## Facing in four and eight directions

A mirror pair costs two tiles of art and covers most roguelikes. Top-down packs usually offer more: distinct art for the four compass points, or for all eight. The Strategy shape absorbs that upgrade without touching `render`, `move`, or `MeleeAction`, because how many directions exist is a private detail of each `Graphic`. This section adds the four-way and eight-way siblings of `DirectionalGraphic2`, and the bookkeeping the extra art needs.

Directions deserve names before anything else. In `game/entities/components/graphic.py`:

```diff
 from __future__ import annotations
+from enum import Enum
```

```python
class GraphicDirection(Enum):
    DOWN       = 0
    DOWN_RIGHT = 1
    RIGHT      = 2
    UP_RIGHT   = 3
    UP         = 4
    UP_LEFT    = 5
    LEFT       = 6
    DOWN_LEFT  = 7
```

Three reading notes. `DOWN` means "toward the bottom of the screen": the console's `y` axis grows downward, the same convention every `dy` in the game already uses. The code below never reads the numeric values (every lookup goes by member); they exist to fix an order, the same order the sprite sheet's columns will use, so the enum, the constants, and the remaps all read straight across. And `DirectionalGraphic2` keeps its `flip` boolean untouched: two directions do not need the enum.

### Four directions

`DirectionalGraphic4` keeps one tile per compass point and reduces every move to one of them:

```python
class DirectionalGraphic4(Graphic):
    """Four tiles: one per cardinal direction, snapped to whichever axis dominates."""

    def __init__(self,
                 down: int,
                 right: int,
                 up: int,
                 left: int,
                 initial: GraphicDirection = GraphicDirection.DOWN) -> None:
        self.direction = initial
        self.sprites = {
            GraphicDirection.DOWN:  down,
            GraphicDirection.RIGHT: right,
            GraphicDirection.UP:    up,
            GraphicDirection.LEFT:  left,
        }

    def face(self, dx: int, dy: int) -> None:
        if dx == 0 and dy == 0:
            return

        if abs(dx) > abs(dy):
            self.direction = GraphicDirection.LEFT if dx < 0 else GraphicDirection.RIGHT
        else:
            self.direction = GraphicDirection.UP if dy < 0 else GraphicDirection.DOWN

    @property
    def glyph(self) -> str:
        return chr(self.sprites[self.direction])
```

The `face` override is where four tiles meet eight-way movement. A diagonal move has no tile of its own, so the comparison asks which axis dominates: the horizontal art wins only when the move is *more* horizontal than vertical. On a pure diagonal both amounts tie and the vertical art wins, on purpose: front and back views are the most readable poses in most sheets, so ties fall to them rather than to a profile.

`initial` exists because an actor must show some tile before its first move. `DOWN`, the pose that looks at the player, is what most sheets draw as the resting frame.

### Eight directions

`DirectionalGraphic8` is the same idea with the full compass. The one new trick is normalizing the move before the lookup:

```python
class DirectionalGraphic8(Graphic):
    """Eight tiles: one per cardinal and diagonal direction."""

    def __init__(self,
                 down:       int,
                 down_right: int,
                 right:      int,
                 up_right:   int,
                 up:         int,
                 up_left:    int,
                 left:       int,
                 down_left:  int,
                 initial: GraphicDirection = GraphicDirection.DOWN) -> None:
        self.direction = initial
        self.sprites = {
            GraphicDirection.DOWN:       down,
            GraphicDirection.DOWN_RIGHT: down_right,
            GraphicDirection.RIGHT:      right,
            GraphicDirection.UP_RIGHT:   up_right,
            GraphicDirection.UP:         up,
            GraphicDirection.UP_LEFT:    up_left,
            GraphicDirection.LEFT:       left,
            GraphicDirection.DOWN_LEFT:  down_left,
        }

    def face(self, dx: int, dy: int) -> None:
        if dx == 0 and dy == 0:
            return

        x = 0 if dx == 0 else 1 if dx > 0 else -1
        y = 0 if dy == 0 else 1 if dy > 0 else -1
        self.direction = {
            ( 0,  1): GraphicDirection.DOWN,
            ( 1,  1): GraphicDirection.DOWN_RIGHT,
            ( 1,  0): GraphicDirection.RIGHT,
            ( 1, -1): GraphicDirection.UP_RIGHT,
            ( 0, -1): GraphicDirection.UP,
            (-1, -1): GraphicDirection.UP_LEFT,
            (-1,  0): GraphicDirection.LEFT,
            (-1,  1): GraphicDirection.DOWN_LEFT,
        }[(x, y)]

    @property
    def glyph(self) -> str:
        return chr(self.sprites[self.direction])
```

`x` and `y` are the *signs* of `dx` and `dy`: any move, one cell or many, collapses to `-1`, `0`, or `1` on each axis. That leaves nine combinations; `(0, 0)` already returned at the top, and the remaining eight are exactly the dictionary's keys. Looking the answer up in a literal dictionary instead of walking an `if` chain is the same dispatch trick the key maps have used since Part 5.

### One block of ids per actor

Count what an actor owns now: a base tile, its mirror, and eight directions, ten ids in all. The gap of 10 from the mirror section is spent to the last slot, with no room left before the next actor. This is the moment to renumber, and the block idea mentioned back then lands one size finer: a block of 100 ids per actor, and a block of 1000 per category:

```python
class SpriteGlyphs:

    # One block of 100 ids per actor:
    # +0 the base tile, +1 its mirror, +10..+17 the eight directions.
    PLAYER = PUA + 0
    ORC    = PUA + 100
    TROLL  = PUA + 200

    PLAYER_FLIPPED = PLAYER + 1
    ORC_FLIPPED    = ORC    + 1
    TROLL_FLIPPED  = TROLL  + 1

    PLAYER_DOWN       = PLAYER + 10
    PLAYER_DOWN_RIGHT = PLAYER + 11
    PLAYER_RIGHT      = PLAYER + 12
    PLAYER_UP_RIGHT   = PLAYER + 13
    PLAYER_UP         = PLAYER + 14
    PLAYER_UP_LEFT    = PLAYER + 15
    PLAYER_LEFT       = PLAYER + 16
    PLAYER_DOWN_LEFT  = PLAYER + 17

    # ORC_DOWN .. ORC_DOWN_LEFT and TROLL_DOWN .. TROLL_DOWN_LEFT
    # repeat the same +10 .. +17 offsets inside their own blocks.
```

In the full game the corpse moves to `PUA + 1000`, the chest and consumables to the 2000s, weapons and armor to the 3000s, and the stairs to the 4000s. The direction offsets follow the `GraphicDirection` order, and renumbering ids changes nothing on screen: the ids are the game's names for tiles, the cells of the sheet have not moved, and `init_sprites` remains the only bridge between the two.

The sprite sheet grows five new rows for this art, rows 8 through 12: one row per actor (player, orc, troll, ghoul, ogre in the full game), eight columns in `GraphicDirection` order. Those rows are why `load_tilesheet` has been passing `13 if sprites.USE_SPRITES else 8` since the wiring section. `init_sprites` gains one line per tile; the player's row shows the pattern, and each other actor repeats it one row further down:

```python
    tileset.remap(SpriteGlyphs.PLAYER_DOWN,       0, 8)
    tileset.remap(SpriteGlyphs.PLAYER_DOWN_RIGHT, 1, 8)
    tileset.remap(SpriteGlyphs.PLAYER_RIGHT,      2, 8)
    tileset.remap(SpriteGlyphs.PLAYER_UP_RIGHT,   3, 8)
    tileset.remap(SpriteGlyphs.PLAYER_UP,         4, 8)
    tileset.remap(SpriteGlyphs.PLAYER_UP_LEFT,    5, 8)
    tileset.remap(SpriteGlyphs.PLAYER_LEFT,       6, 8)
    tileset.remap(SpriteGlyphs.PLAYER_DOWN_LEFT,  7, 8)
```

!!! warning "Guard `init_sprites` against character mode, not just this block"
    Rows 8 to 12 exist only in the extended, 13-row sheet; the plain one stops at row 7. `init_sprites` runs once, unconditionally, right after `main()` loads whichever sheet `USE_SPRITES` picked, so a run with `USE_SPRITES = False` would still reach these `remap` calls against the 8-row sheet, and `tileset.remap` raises `IndexError: Tile ... is non-existent` the moment it gets to row 8.

    The fix is not to special-case these particular rows: character mode never reads a `SpriteGlyphs` codepoint at all, `CurrentGlyphs` is `CharGlyphs` there, so nothing in `init_sprites` needs to run in the first place. Guard the whole function, once, at the top:

    ```diff
     def init_sprites(tileset: tcod.tileset.Tileset) -> None:
         """Point each sprite codepoint at its tile (column, row) in the sheet."""
    +    if not USE_SPRITES:
    +        return
    +
         tileset.remap(SpriteGlyphs.PLAYER,            0, 5)
    ```

    Add this once you reach this section, not before: everything `init_sprites` remapped up to row 7 already exists on both sheets, so the crash only becomes possible from here, where the function first reaches past row 7.

### Wiring the factories

The helpers follow `_directional2`, one per family, still deciding sprite mode in one place:

```python
def _directional4(down:     int,
                  right:    int,
                  up:       int,
                  left:     int,
                  fallback: str | int,
                  initial:  GraphicDirection = GraphicDirection.DOWN) -> Graphic:
    return DirectionalGraphic4(down, right, up, left, initial) if sprites.USE_SPRITES else StaticGraphic(fallback)

def _directional8(down:       int,
                  down_right: int,
                  right:      int,
                  up_right:   int,
                  up:         int,
                  up_left:    int,
                  left:       int,
                  down_left:  int,
                  fallback:   str | int,
                  initial:    GraphicDirection = GraphicDirection.DOWN) -> Graphic:
    return DirectionalGraphic8(down, down_right, right, up_right, up, up_left, left, down_left, initial) if sprites.USE_SPRITES else StaticGraphic(fallback)
```

with the import line grown to match:

```diff
-from game.entities.components.graphic import DirectionalGraphic2, Graphic, StaticGraphic
+from game.entities.components.graphic import DirectionalGraphic2, DirectionalGraphic4, DirectionalGraphic8, Graphic, GraphicDirection, StaticGraphic
```

An actor upgrades by naming its eight tiles; the fallback works exactly as before:

```python
player = Actor(
    graphic    = _directional8(
        sprites.SpriteGlyphs.PLAYER_DOWN,
        sprites.SpriteGlyphs.PLAYER_DOWN_RIGHT,
        sprites.SpriteGlyphs.PLAYER_RIGHT,
        sprites.SpriteGlyphs.PLAYER_UP_RIGHT,
        sprites.SpriteGlyphs.PLAYER_UP,
        sprites.SpriteGlyphs.PLAYER_UP_LEFT,
        sprites.SpriteGlyphs.PLAYER_LEFT,
        sprites.SpriteGlyphs.PLAYER_DOWN_LEFT,
        sprites.CurrentGlyphs.PLAYER
    ),
    ...
)
```

The orc, the troll, and the rest make the same call with their own blocks.

!!! info "Three strategies, zero new branches"
    Stand back and count what did *not* change. `render` still prints `entity.glyph`. `move` and `MeleeAction` still call `face(dx, dy)` without knowing who is listening. Switching one actor between two, four, and eight directions is one factory line, and a future `DirectionalGraphic16`, or a graphic that turns by swapping palettes, would slot in the same way. This is the Strategy pattern paying rent: every new behavior arrived as a new class plus data, never as a new `if` in the drawing code.

---

## Constraints and what comes next

- **One tile per cell.** Every sprite must fit a single grid cell, same size as the font. Multi-tile creatures and large sprites need extra work and are out of scope here.
- **Layering two graphics is the hard part.** Drawing a sprite *on top of another sprite* (a floor tile under a creature, an item glow behind an icon) is not straightforward in tcod, because a cell holds one tile. It is possible, and it gets its own appendix (advanced tcod techniques), with worked examples.
- **Animation timing and composition** belong in [Appendix 8](append-8.md). The `AnimatedGraphic` sketch above shows the shape; advancing frames on a clock, composing animation with facing, plus **blending** and **partial transparency**, are the deeper work.

---

## Summary

Graphics in tcod are not a different rendering path: they are the same tile system with the codepoints remapped. `tileset.remap` points a free id at a tile in the sheet, and the Unicode Private Use Area is where those ids come from, so they never collide with a real character. A `sprites` module collects them behind a `USE_SPRITES` flag and an `init_sprites()` function, organized either as flat constants or as two switchable sets, and forces `fg` to white whenever the game is in sprite mode, so the art keeps its own colors.

Rather than branch on `int` versus `str` at every draw, each entity delegates to a `Graphic`: a small strategy object whose `glyph` property resolves to the character tcod prints. That one change is what lets facing arrive as new kinds of `Graphic` instead of new special cases in `render`: a mirror pair in `DirectionalGraphic2`, four compass points in `DirectionalGraphic4`, the full eight in `DirectionalGraphic8`, all fed by the same `face()` calls from movement and combat, with `AnimatedGraphic` sketched as the next step. The whole system still assumes one tile per cell; layering two graphics, blending, and full animation timing go beyond that, and animation is the subject of [Appendix 8](append-8.md).
