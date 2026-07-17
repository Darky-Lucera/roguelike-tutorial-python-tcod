# Appendix 8 Draft: Animations and Visual Effects (WIP)

!!! note "Draft notes"

    This document is a parking place for future ideas. It is not part of the main tutorial flow yet, and the details below should be revisited and balanced before being turned into a finished appendix.

    It builds directly on [Appendix 7](append-7.md) (sprite codepoints, the `Graphic` strategy) and edits the main loop as [Part 10](part-10.md) leaves it. The trigger examples assume the consumables of [Part 9](part-9.md). All code below is concept level: checked against the current game layout, but not implemented in the reference repository yet.

Appendix 7 gave every entity its art, and the whole dungeon still stands perfectly still. Nothing blinks, nothing flies, an exploding fireball is a log line and some instant hit point math. This appendix maps what it takes to add motion: a flash when lightning strikes, an arrow that crosses the room, sprites with an idle animation.

The surprise is where the difficulty lives. It is not in the sprites, not in the tileset, not in the `Graphic` system. It is in the main loop.

---

## A loop that waits

The heart of `run()` is one call:

```python
for event in tcod.event.wait():
```

`wait()` **blocks**. Between one keypress and the next, the program is asleep inside that line: no code runs, no frame is drawn. The screen you see between turns is just the last frame, sitting unchanged in the window. A two-frame sprite has no chance, because nobody is redrawing anything while the player thinks.

This is a feature, not a bug. A turn-based game is idle almost all the time, and a loop that only wakes for input spends zero CPU while the player stares at the map. Any animation, though, needs the opposite: frames drawn *without* input, plus a measure of how much time passed between them. In concept:

```python
# Concept, not final code
while True:
    delta_time = clock.tick()          # seconds since the previous frame
    effects.update(delta_time)         # advance whatever is animating

    console.clear()
    state.on_render(console)
    effects.render(console)            # draw animations on top of the game
    context.present(console)

    for event in tcod.event.get():     # non-blocking
        state = state.handle_events(event)
```

Running that at 60 frames per second, forever, would burn CPU and battery to redraw an identical frame all night. The reasonable design for a roguelike is a **hybrid**: `tcod.event.wait(timeout=...)` while something is animating, so the loop wakes on its own to draw the next frame, and plain `wait()` when nothing is, so the game goes back to costing nothing. `wait(timeout=n)` returns after `n` seconds even if no event arrived, which is exactly the wake-up call an animation needs.

The rest of this appendix builds the pieces that hybrid loop needs, then wires them in.

---

## An `Effects` manager

An effect is a short-lived visual: it appears, it changes with time, it removes itself. That is a small interface, and a manager that owns the list. Create `game/effects.py`:

```python
from __future__ import annotations

from tcod.console import Console


class Effect:
    """One short-lived visual. It advances with time and draws over the map."""

    done: bool = False

    def update(self, delta_time: float) -> None:
        raise NotImplementedError

    def render(self, console: Console) -> None:
        raise NotImplementedError


class Effects:
    """Every effect currently playing."""

    _playing: list[Effect] = []

    @classmethod
    def add(cls, effect: Effect) -> None:
        cls._playing.append(effect)

    @classmethod
    def any_playing(cls) -> bool:
        return bool(cls._playing)

    @classmethod
    def update(cls, delta_time: float) -> None:
        for effect in cls._playing:
            effect.update(delta_time)

        cls._playing = [effect for effect in cls._playing if not effect.done]

    @classmethod
    def render(cls, console: Console) -> None:
        for effect in cls._playing:
            effect.render(console)
```

!!! info "The `MessageLog` pattern, and why saves do not care"
    `Effects` keeps its state at class level and is called from anywhere as `Effects.add(...)`, exactly like `MessageLog.add_message(...)` since Part 7. The pattern has a second payoff here: the manager lives *outside* the `Engine`, and Part 10 pickles the `Engine`. Effects are transient by nature, so keeping them out of the object graph that gets saved means a save file never contains half-played animations, with no `__getstate__` tricks needed.

---

## Wiring the hybrid loop

Name the frame budget in `game/data/config.py`:

```python
FRAME_TIME = 1 / 60   # seconds per frame while something is animating
```

Then `run()` in `main.py` learns about time. `time.monotonic()` is the standard clock for this: it only moves forward, immune to the system clock being adjusted under you:

```diff
+import time
+
+from game.effects import Effects
 ...
     try:
+        previous = time.monotonic()
         while True:
+            now        = time.monotonic()
+            delta_time = min(now - previous, config.FRAME_TIME)
+            previous   = now
+
+            Effects.update(delta_time)
+
             console.clear()
             state.on_render(console=console)
+            Effects.render(console)
             context.present(console)

-            for event in tcod.event.wait():
+            timeout = config.FRAME_TIME if Effects.any_playing() else None
+            for event in tcod.event.wait(timeout=timeout):
                 converted_event = context.convert_event(event)
                 state = state.handle_events(converted_event)
```

With no effects playing, `timeout` is `None` and the loop behaves exactly as before: it sleeps in `wait()` until the player acts. The moment an effect exists, the loop wakes 60 times per second, advances it, draws it, and goes back to sleeping the instant the list is empty.

`delta_time` is the number of seconds the previous frame took, and every effect advances by that amount instead of by "one tick". This is **frame-rate independence**: a projectile at 30 cells per second crosses the same map in the same real time whether the loop managed 60 frames or 13 that second. Game code usually shortens the name to `dt`; the tutorial keeps it spelled out.

!!! warning "The first frame after a long wait"
    Note the `min(...)` clamp on `delta_time`, and what it is guarding. The player thinks for two minutes, presses a key, the scroll fires and adds an effect. On the very next loop pass, `now - previous` is *two minutes*, because `previous` was stamped before the long sleep. Without the clamp, the brand-new effect would receive those 120 seconds in a single `update()` and finish instantly, skipping its whole animation. Clamping to one frame budget means a frame is never asked to represent more time than a frame.

---

## Three cheap effects

### Flash

The simplest effect: tint one cell and fade the tint away. It is the same weighted color mix the area targeting preview draws in Part 9, only decaying over time instead of sitting static:

```python
class Flash(Effect):
    """Tint one cell with a color that fades away."""

    def __init__(self, x: int, y: int, color: Color, duration: float = 0.25) -> None:
        self.x        = x
        self.y        = y
        self.color    = color
        self.duration = duration
        self.elapsed  = 0.0

    def update(self, delta_time: float) -> None:
        self.elapsed += delta_time
        self.done = self.elapsed >= self.duration

    def render(self, console: Console) -> None:
        strength = max(0.0, 1.0 - self.elapsed / self.duration)
        bg = console.rgb[self.x, self.y]["bg"]
        console.rgb[self.x, self.y]["bg"] = tuple(
            int(base * (1.0 - strength) + tint * strength)
            for base, tint in zip(bg, self.color, strict=True)
        )
```

### Projectile

One glyph traveling the straight line between two map positions. `tcod.los.bresenham` returns every cell on that line; the effect just picks which one matches the elapsed time:

```python
import tcod


class Projectile(Effect):
    """One glyph traveling the straight line between two positions."""

    def __init__(
        self,
        start: tuple[int, int],
        end:   tuple[int, int],
        glyph: str,
        color: Color,
        speed: float = 30.0,   # cells per second
    ) -> None:
        self.path    = tcod.los.bresenham(start, end).tolist()
        self.glyph   = glyph
        self.color   = color
        self.speed   = speed
        self.elapsed = 0.0

    def update(self, delta_time: float) -> None:
        self.elapsed += delta_time
        self.done = self.elapsed * self.speed >= len(self.path)

    def render(self, console: Console) -> None:
        index = min(int(self.elapsed * self.speed), len(self.path) - 1)
        x, y  = self.path[index]
        console.print(x, y, self.glyph, fg=self.color)
```

!!! info "Bresenham, 1962"
    The line algorithm behind `tcod.los.bresenham` was designed by Jack Bresenham at IBM in 1962, to drive pen plotters: machines that drew with a physical arm and could only step one unit at a time. It picks which cells best approximate a straight line using only integer additions, no floating point, which made it fast enough for 1960s hardware and keeps it everywhere today, from FOV code to your arrow.

### Explosion

A disc of `Flash`-style tints instead of a single cell: every cell within the fireball's radius gets the fading overlay, optionally delayed a little more the farther it sits from the center, so the blast reads as expanding. No new machinery, just a loop over the same cells the fireball already damages.

### Firing them

The trigger points already exist. Every consumable calls `MessageLog.add_message` at the moment of impact; the effect is one more line beside it. In `LightningDamageConsumable.activate()`:

```diff
             target.fighter.take_damage(self.damage, attacker=consumer)
+            Effects.add(Flash(target.x, target.y, colors.LIGHTNING, duration=0.2))
```

One line per effect, per trigger. The fireball adds its explosion, a future bow adds a `Projectile` from archer to victim.

---

## Cosmetic or blocking?

There is a design fork here, and it decides how much of the game the effects touch.

**Cosmetic (fire and forget).** The damage applies instantly, exactly as today, and the animation plays *on top* while the game keeps accepting input. The lightning already killed the orc; the flash is just telling you about it. Nothing in the turn flow, the states, or the actions changes. This is the version everything above builds, and the right place to start.

**Blocking.** The arrow flies first, *then* the hit resolves, and the player cannot act while it is in the air. This is a different animal: it means deferring the resolution of an action until an animation completes, an `AnimationPlayingState` that swallows input until done, and changes to the turn flow in `GameState.handle_events`. It is the classic roguelike dilemma: veterans hold a key to descend ten floors and resent every enforced delay. If you go this way, keep the animations fast and add a setting to skip them. Out of scope for a first iteration.

---

## Animating the sprites themselves

Appendix 7 already sketched the shape: `AnimatedGraphic`, a `Graphic` that steps through frames. What was missing was time, and the loop now provides it. Two routes, by cost.

**The shared clock (cheap).** One class-level clock, advanced by the loop, read by every animated glyph:

```python
class AnimatedGraphic(Graphic):
    """Cycles through frames on a clock shared by every animated entity."""

    clock: float = 0.0             # advanced once per frame, in run()
    frames_per_second: float = 2.0

    def __init__(self, frames: list[int]) -> None:
        self.frames = frames

    @property
    def glyph(self) -> str:
        index = int(AnimatedGraphic.clock * self.frames_per_second) % len(self.frames)
        return chr(self.frames[index])
```

with one line in `run()`, next to `Effects.update`:

```python
AnimatedGraphic.clock += delta_time
```

`glyph` stays a pure read, honoring the Appendix 7 rule: the loop advances the clock, the property only looks at it. There is no per-entity state, so nothing new enters the savegame, and every orc on screen animates in step, which for a roguelike reads as tidy rather than cheap.

Note one interaction with the hybrid loop: idle animation means the game is *always* animating while any animated entity is on screen, so the loop never gets to sleep fully. A two-frame idle cycle does not need 60 frames per second, though; a second, slower timeout (`1 / frames_per_second`) while only idle animation is running keeps the cost negligible.

!!! info "The remap trick: animating the tileset instead of the game"
    There is a zero-render-changes alternative. The tileset is a lookup table, and `tileset.remap(SpriteGlyphs.PLAYER, column, row)` rewrites one entry of it at any time, not just at startup. Remap the same codepoint to a different sheet cell on a clock, and every `@` on screen changes costume at once, without touching `Graphic`, `render`, or the entities. It is the sprite equivalent of editing the font while the text stays put.

**Per-entity state (complete).** Walk cycles that start when the entity moves, animations out of phase, one-shot animations like an attack swing: these need a current frame and an accumulated time *per entity*, and a `tick(delta_time)` call reaching each visible entity every frame. It also puts animation state inside the pickled object graph, so a save written mid-cycle carries those floats; harmless in size, but worth excluding via `__getstate__` if you want saves byte-identical across runs. Defer this route until a concrete animation demands it.

---

## The expensive part is the art

Everything above is one or two afternoons of code. The real cost is pixels, and there is a trap waiting in the asset packs.

Every frame must fit one cell, and the font decides the cell: 12x12 in this tutorial. The animated packs you will actually find are mostly 16x16 art, often delivered on 32x32 canvases so that swords, bows, and capes can overhang the character's own tile during a swing. Neither survives the trip: downscaling 16x16 pixel art to 12x12 destroys it (pixel art has no fractional pixels to spare), and overhang simply cannot exist when a sprite *is* one cell.

The realistic options:

- **Art made for your grid.** Draw or commission at exactly the cell size. This is what the extended sheet from Part 1 does.
- **Move the game to the art's grid.** The font sets the cell, so switching to a 16x16 font and a 16x16 sheet realigns everything; window columns and rows stay the same, the window just grows. Everything in this appendix is size-agnostic.
- **Keep entities static, animate with effects.** Flashes, projectiles, and tint pulses need no new art at all. This is the cheapest route and, combined with facing from Appendix 7, already reads as a living game.

---

## The ceiling: cells, not pixels

Everything here animates *by cell*. The projectile jumps from square to square; nothing slides smoothly between them, because a tcod console is a grid of tiles, not a canvas of pixels. That is the medium, and per-cell motion with color pulses is aesthetically coherent with it.

Going below the cell, to sub-pixel movement, particles, or sprites gliding between tiles, means leaving the console: rendering it to a texture via the SDL layer (`context.sdl_renderer` plus `tcod.render.SDLConsoleRender`) and drawing free-floating sprites on top. That is a real architecture jump, and a different appendix, if it ever earns one.

---

## Where to start

The order that keeps the game runnable at every step:

1. The hybrid loop in `main.py`: time infrastructure only, no animations yet. The game must behave identically.
2. `Effects` manager plus `Flash`, wired to the lightning scroll. First visible payoff.
3. `Projectile` for a thrown or shot attack.
4. The explosion for the fireball.
5. Extra frames in the tilesheet plus the shared-clock idle animation.
6. Only if the game asks for it: blocking effects, per-entity animation state, or the SDL layer.

---

## Summary

The obstacle to animation was never the graphics, it was `tcod.event.wait()`: a loop that only draws when the player acts cannot show motion. The fix is a hybrid loop that wakes on a timeout while something is playing and sleeps like before when nothing is, plus a `delta_time` so animations advance by real seconds, clamped so the first frame after a long think does not swallow the whole show.

On top of that clock, effects are small self-discarding objects in a `MessageLog`-style manager that lives outside the save file: a fading `Flash`, a `Projectile` walking a Bresenham line, an explosion that is a disc of flashes. They fire from one-line hooks in the consumables and play over the game without touching the turn flow, as long as they stay cosmetic. Sprite idle animation rides the same clock through `AnimatedGraphic`, shared by all entities in the cheap version. The genuinely expensive ingredient is art that fits the cell, and the honest ceiling is the cell itself: smooth sub-tile motion belongs to the SDL layer, another world away.
