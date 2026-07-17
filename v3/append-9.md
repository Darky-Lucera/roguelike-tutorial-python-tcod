# Appendix 9 Draft: A Camera for Bigger Maps (WIP)

!!! note "Draft notes"

    This document is a parking place for future ideas. It is not part of the main tutorial flow yet, and the details below should be revisited and balanced before being turned into a finished appendix.

    The camera itself only needs the map from [Part 2](part-2.md) and the renderer as it stands after [Part 6](part-6.md). The mouse section assumes [Part 7](part-7.md) (`Engine.mouse_location`, the HUD) and the targeting section assumes [Part 9](part-9.md). The code is written against the finished game as the chapters leave it, without the optional exercises; the reference repository carries those on top (the fading memory, for example), and the notes below call them out where they matter. If you are earlier in the tutorial, adapt the module paths and skip the sections about systems you do not have yet.

Every map in this tutorial is exactly as big as the window that shows it. `MAP_WIDTH` is 80, the same as `SCREEN_WIDTH`. `MAP_HEIGHT` is 44: the 50 screen rows minus the 6 that the HUD panel keeps. That is not a coincidence. It is a silent assumption that a lot of code depends on: *the position of a tile on the map and its position on the screen are the same pair of numbers*.

Try to break it. Set `MAP_HEIGHT = 100` and the first frame crashes: `GameMap.render` asks numpy to copy a block of 100 rows into a console that only has 50. The renderer does not know the difference between "where a tile is" and "where a tile is drawn", because so far there has never been one.

This appendix introduces that difference on purpose. The tool that manages it is a **camera**: a small object that decides which part of the map is on screen and translates positions between the two spaces. With one in place, nothing stops a floor from being 120x80, or 300x300, with the view scrolling over it as the player walks.

!!! info "Why classic levels fit on one screen"

    Rogue (1980) ran on the serial terminals of its time: screens of 80 columns by 24 rows. A level had to fit in one screen because there was nowhere else to put it. The constraint became a genre convention that outlived the hardware by decades, and plenty of roguelikes still ship screen-sized levels on purpose: "you can see the whole battlefield" is a real tactical virtue, not just nostalgia.

    Moria (1983) had already outgrown the screen: its dungeon was far wider than any terminal, and the view moved in **panels**, half-screen jumps that triggered when you walked near the edge. Angband (1990) inherited that scheme. Smooth per-tile scrolling, the kind this appendix builds, became the default later, when redrawing the whole screen every turn stopped being expensive.

## Two coordinate systems

From now on, a position can live in one of two spaces:

- **Map coordinates**: where something *is*. `entity.x`, room centers, FOV arrays, pathfinding: everything the game simulates.
- **Screen coordinates**: where something is *drawn*. Console cells, mouse events: everything the window sees.

The camera is the bridge. Its position `(camera.x, camera.y)` is the map coordinate of the screen's top-left tile, and the translation is one subtraction each way:

```text
screen = map - camera        map = screen + camera
```

We will also give a name to the rectangle of the screen that shows the map: the **viewport**. In our layout it is the top 80x44 cells; the bottom 6 rows belong to the HUD and are not the camera's business.

Before writing the class, it helps to see where the old identity hides. Four places treat one space as the other, and nothing breaks today:

- `GameMap.render` writes tiles and entities at their map position, straight into console cells.
- The `MouseMotion` case in `GameState.handle_events` stores `event.integer_position` (screen) in `Engine.mouse_location`.
- The HUD reads `mouse_location` back as a map position and checks it against `game_map.visible`.
- The Part 9 targeting states draw the cursor at `mouse_location` (map) but read clicks from `event.integer_position` (screen).

Each of those lines will keep working or break depending on one question: which space does this value live in? Keeping that answer straight is the whole discipline of camera code.

## A `Camera` class

Create `game/camera.py`:

```python
from __future__ import annotations

import numpy as np


class Camera:
    """The window through which the screen looks at the map."""

    def __init__(self, width: int, height: int) -> None:
        self.width = width
        self.height = height
        self.x = 0
        self.y = 0

    def follow(
        self,
        target_x: int,
        target_y: int,
        map_width: int,
        map_height: int,
    ) -> bool:
        """Center the viewport on the target without leaving the map.

        Returns True if the camera moved.
        """
        x = max(0, min(target_x - self.width // 2, map_width - self.width))
        y = max(0, min(target_y - self.height // 2, map_height - self.height))

        moved = (x, y) != (self.x, self.y)
        self.x = x
        self.y = y
        return moved
```

`follow` centers the view on the target and then clamps: `min(...)` stops the right and bottom edges, `max(0, ...)` stops the left and top ones. Near a map edge the camera stands still and the player walks toward the border of the screen, which is exactly what you want. It also reports whether it actually moved; keep that detail in mind, the mouse handling further down depends on it.

Notice what the class does *not* have: no velocity, no history, no update order to get wrong. In a turn-based game a centered camera is a pure computation from three things (target, viewport size, map size), so `follow` can rebuild it from scratch every frame and it can never go stale. It also takes the map size as two plain integers instead of a `GameMap`, so the class knows nothing about the rest of the game: it could follow a confused enemy or a cutscene just as happily, and you can test it in isolation with nothing but numbers.

The rest of the class is the translation service: the two conversions, a bounds check, and the slices the renderer will use.

```python
    def screen_to_map(self, x: int, y: int) -> tuple[int, int]:
        """Where a screen position lands on the map."""
        return x + self.x, y + self.y

    def map_to_screen(self, x: int, y: int) -> tuple[int, int]:
        """Where a map position lands on the screen."""
        return x - self.x, y - self.y

    def in_view(self, x: int, y: int) -> bool:
        """True if the map position is inside the viewport."""
        return (
            self.x <= x < self.x + self.width
            and self.y <= y < self.y + self.height
        )

    @property
    def view(self) -> tuple[slice, slice]:
        """The viewport as slices, ready to index the map arrays."""
        return np.s_[self.x : self.x + self.width, self.y : self.y + self.height]
```

One assumption to keep in mind: `follow` and `view` expect the map to be at least as big as the viewport. A smaller map would pin the camera to `(0, 0)` and shrink the `view` slices, and the renderer below would need extra care to center the leftover space. The tutorial's maps only grow, so the main path ignores that case; centering small maps is one of the exercise seeds at the end.

!!! tip "`np.s_` builds slice objects"

    [Part 3](part-3.md) introduced slice objects, the things Python creates from the `a:b` syntax inside square brackets. That syntax is only legal inside brackets, which makes slices awkward to store in a variable.

    `np.s_` is numpy's tiny helper for exactly that: `np.s_[2:5]` is `slice(2, 5)`, and `np.s_[2:5, 0:44]` is the tuple `(slice(2, 5), slice(0, 44))`. The `view` property returns that tuple, so `self.visible[camera.view]` means the same as `self.visible[camera.x : camera.x + camera.width, camera.y : camera.y + camera.height]`, with one important bonus: a slice of a numpy array is a *view*, not a copy, so indexing this way costs almost nothing.

## Rendering through the viewport

`GameMap.render` currently copies the whole map into the console. Now it copies the slice the camera selects, and converts every entity position on the way out. Add `Camera` to the `TYPE_CHECKING` imports of `game/map/game_map.py`, then:

```diff
-    def render(self, console: Console) -> None:
-        console.rgb[0 : self.width, 0 : self.height] = np.select(
-            condlist   = [self.visible, self.explored],
-            choicelist = [self.tiles["in_fov"], self.tiles["out_of_fov"]],
-            default    = tile_types.UNSEEN,
-        )
+    def render(self, console: Console, camera: Camera) -> None:
+        view = camera.view
+        console.rgb[0 : camera.width, 0 : camera.height] = np.select(
+            condlist   = [self.visible[view], self.explored[view]],
+            choicelist = [self.tiles["in_fov"][view], self.tiles["out_of_fov"][view]],
+            default    = tile_types.UNSEEN,
+        )

         for entity in sorted(self.entities, key=lambda e: e.render_order.value):
             stays_visible = entity.stays_visible and self.explored[entity.x, entity.y]
-            if self.visible[entity.x, entity.y] or stays_visible:
-                console.print(entity.x, entity.y, entity.char, fg=entity.color)
+            if not (self.visible[entity.x, entity.y] or stays_visible):
+                continue
+
+            if not camera.in_view(entity.x, entity.y):
+                continue
+
+            screen_x, screen_y = camera.map_to_screen(entity.x, entity.y)
+            console.print(screen_x, screen_y, entity.char, fg=entity.color)
```

The tile half is the same `np.select` as always; the only change is that every input array is sliced through the same window before the comparison happens. If you carried exercise code in your condition list (the fading memory from Part 4, the mapping scroll from Part 9), slice those arrays the same way; the fading memory also replaces `self.explored` in the `stays_visible` line.

The entity half gains a guard, and it is not paranoia. For *visible* entities you could argue it away: the camera follows the player, the FOV radius is 8, and half a viewport is at least 22 tiles, so anything in view of the player is comfortably on screen. But the loop has drawn `stays_visible` entities in explored tiles since Part 6, and the moment something has that flag ([Part 11](part-11.md) gives it to the stairs), an entity can be explored, far outside the viewport, and still asked to draw.

!!! warning "Clamp it yourself"

    What happens without the guard? On current tcod, mostly nothing: `Console.print` silently clips text that falls outside the console (verified on tcod 21.2.1: printing at a negative position draws only the characters that reach the console, and a row below the console draws nothing at all). The game would look correct by courtesy of one method's undocumented behavior.

    Do not rely on that. Nothing in `print`'s contract promises clipping, and the numpy writes coming later in this appendix (`console.bg` and `console.fg`, for the targeting cursor and the area preview) follow harsher rules: a negative index wraps around to the opposite edge, silently drawing a ghost on the wrong side of the screen, and an index past the edge raises `IndexError`. Guarding and converting at every drawing site is always more correct than trusting another layer to save you.

## Wiring it into the `Engine`

The camera needs a size before anything else, and constants must exist before the code that uses them. Name the viewport in `game/data/config.py`:

```diff
 # Screen / window
 SCREEN_WIDTH = 80
 SCREEN_HEIGHT = 50
+
+# Viewport: the part of the screen that shows the map
+VIEWPORT_WIDTH = SCREEN_WIDTH
+VIEWPORT_HEIGHT = SCREEN_HEIGHT - 6  # the HUD panel keeps its 6 rows
 TITLE = "Roguelike Tutorial"
```

The 6 comes from the HUD panel, and note what this diff does *not* fix: the HUD's own row numbers (44 to 49, spread across `hud.py` and `Engine.render`) stay hardcoded, and they only agree with `VIEWPORT_HEIGHT` by construction. That is acceptable while the viewport height never changes; deriving the HUD layout from `VIEWPORT_HEIGHT` is the first of the exercise seeds.

Now the camera needs a home, somewhere both the renderer and the event handlers can reach: the `Engine`. In `game/engine.py` (this file already imports the config module):

```diff
 from game import hud
+from game.camera import Camera
 from game.data import colors
```

```diff
     ) -> None:
         self.mouse_location: tuple[int, int] = (0, 0)
+        self.camera = Camera(config.VIEWPORT_WIDTH, config.VIEWPORT_HEIGHT)
         self.player = player
```

And in `Engine.render`, aim the camera before drawing the map (ignore the returned flag for now; the mouse section uses it):

```diff
     def render(self, console: Console) -> None:
-        self.game_map.render(console)
+        self.camera.follow(
+            self.player.x,
+            self.player.y,
+            self.game_map.width,
+            self.game_map.height,
+        )
+        self.game_map.render(console, self.camera)
```

Because `follow` runs at the start of every frame, there is nothing to update anywhere else: not after moving, not after taking stairs, not after loading a save. Whatever happened to the player since the last frame, the camera is correct by construction.

!!! warning "Old saves and the new attributes"

    [Part 10](part-10.md) pickles the whole `Engine`, and this appendix adds attributes to it: the camera here, and a raw mouse position in the next section. A save written before this change unpickles into an `Engine` without them, and the first render raises `AttributeError`.

    Either delete old saves, as Part 11 and Part 13 already made you do, or supply the missing attributes with the `__getattr__` migration trick from Part 10.

Finally, the payoff. Grow the map in `game/data/config.py`:

```diff
 # Map generation
-MAP_WIDTH = 80
-MAP_HEIGHT = 44
-MAX_ROOMS = 30
+MAP_WIDTH = 120
+MAP_HEIGHT = 80
+MAX_ROOMS = 80
 ROOM_MIN_SIZE = 6
```

`MAX_ROOMS` grows with the map: 30 rooms in almost three times the area feels like a desert. Run the game. The dungeon no longer fits the window; the `@` stays centered while the world slides under it, and near a map edge the camera stops and lets the `@` walk to the border. The HUD has not moved, the message log still works, combat and FOV are untouched. Then move the mouse over a monster, and meet the one system we broke.

## The mouse crosses the border

Part 7 stores the mouse position in the `match` block of `GameState.handle_events`:

```python
case tcod.event.MouseMotion():
    self.engine.mouse_location = event.integer_position
```

That line was correct by accident. `integer_position` is a screen position; everything that reads `mouse_location` (the HUD's names-under-mouse, the targeting states) treats it as a map position. The two spaces coincided, so nobody noticed, and the `in_bounds` check inside the HUD quietly rejected the panel rows because map row 46 did not exist.

With a camera, the accident turns into two real bugs. Point at a troll on screen and the HUD looks up whatever tile happens to share those numbers, up and left of what you see. Hover the HUD panel and screen row 46 now converts to a tile that *does* exist, hidden under the panel; if it is in your FOV, the HUD names an entity you are not pointing at.

The fix states the rule this appendix runs on: **convert at the boundary**. The moment a position enters the game from the outside world, translate it, and let everything downstream keep speaking map coordinates. Give the conversion to the `Engine`, next to the attributes it feeds, and keep the raw screen position too (you will see why in a moment):

```python
def update_mouse_location(self, screen_x: int, screen_y: int) -> None:
    """Translate a raw screen position into a map position for mouse_location."""
    self.mouse_screen_location = (screen_x, screen_y)

    if 0 <= screen_x < self.camera.width and 0 <= screen_y < self.camera.height:
        self.mouse_location = self.camera.screen_to_map(screen_x, screen_y)
    else:
        self.mouse_location = (-1, -1)
```

Add `self.mouse_screen_location: tuple[int, int] = (0, 0)` next to `mouse_location` in `Engine.__init__`, and route the event through the new method:

```diff
             case tcod.event.MouseMotion():
-                self.engine.mouse_location = event.integer_position
+                self.engine.update_mouse_location(*event.integer_position)
```

Positions outside the viewport (the HUD rows) become `(-1, -1)`, a map position that fails every `in_bounds` check downstream, which is exactly the behavior the old code got for free. And because `handle_events` lives in `GameState`, the base class of every in-game state, this one line is the single point that updates `mouse_location`, for the main game, the game over screen, and the targeting states alike. Mouse *clicks* are a second, separate doorway: the targeting section below converts them in their own handler.

One subtlety remains, and it is the reason the method stores the raw screen position. Events only fire when the mouse *moves*, but the camera can move under a mouse that stays still: every step the player takes scrolls the world beneath the pointer, no `MouseMotion` arrives, and `mouse_location` quietly points one tile behind what the player sees. A camera move is a border crossing too: the screen position did not change, but its meaning did. So repeat the conversion whenever `follow` reports movement; this is what its return value is for:

```diff
     def render(self, console: Console) -> None:
-        self.camera.follow(
-            self.player.x,
-            self.player.y,
-            self.game_map.width,
-            self.game_map.height,
-        )
+        if self.camera.follow(
+            self.player.x,
+            self.player.y,
+            self.game_map.width,
+            self.game_map.height,
+        ):
+            self.update_mouse_location(*self.mouse_screen_location)
         self.game_map.render(console, self.camera)
```

This never fights the keyboard-driven cursor of the targeting states: while the player is aiming, the player is not walking, so the camera reports no movement and `mouse_location` stays wherever the keys put it.

And here is the pleasant surprise: `hud.py` does not change at all. `render_names_at_mouse_location` always thought in map coordinates; it was the data feeding it that was mislabeled. When you keep each function inside one coordinate space, a change like this stays local to the border.

## Targeting through the camera

The Part 9 targeting states mix the two spaces in four places, and each one follows a pattern you have already seen.

**Entering the state.** `engine.mouse_location = player.x, player.y` is already a map position. Unchanged.

**Drawing the cursor.** `SelectIndexState.on_render` writes `console.bg` at a map position: the entity bug again. Guard with the camera and convert. `in_view` is stricter than the old `in_bounds` (the viewport always lies inside the map) and it also rejects the `(-1, -1)` sentinel:

```diff
         x, y = self.engine.mouse_location
-        if self.engine.game_map.in_bounds(x, y):
-            console.bg[x, y] = colors.WHITE
-            console.fg[x, y] = colors.BLACK
+        camera = self.engine.camera
+
+        if camera.in_view(x, y):
+            screen_x, screen_y = camera.map_to_screen(x, y)
+            console.bg[screen_x, screen_y] = colors.WHITE
+            console.fg[screen_x, screen_y] = colors.BLACK
```

**Moving the cursor with the keyboard.** The old code clamps the cursor to the *map*, which used to be the same as keeping it on screen. On a big map it is not: the cursor could walk off the viewport and keep going, selecting tiles the player cannot see. Clamp to the viewport instead, in map space. Nothing targetable is lost: everything you can aim at must be visible, and with the centered camera everything visible is on screen (the margins section revisits that promise).

```diff
             x, y = self.engine.mouse_location
             dx, dy = keys.MOVE_KEYS[key]
-            x = max(0, min(x + dx * modifier, self.engine.game_map.width - 1))
-            y = max(0, min(y + dy * modifier, self.engine.game_map.height - 1))
+            camera = self.engine.camera
+            x = max(camera.x, min(x + dx * modifier, camera.x + camera.width - 1))
+            y = max(camera.y, min(y + dy * modifier, camera.y + camera.height - 1))
             self.engine.mouse_location = x, y
```

**Clicking.** `event.integer_position` is a screen position entering the game: guard and convert, the same shape as `update_mouse_location`. The viewport always lies inside the map, so the old `in_bounds` check is subsumed by the new guard:

```diff
     def event_mousebuttondown(self, event: tcod.event.MouseButtonDown) -> Action | None:
-        x, y = event.integer_position
-        if self.engine.game_map.in_bounds(x, y):
+        screen_x, screen_y = event.integer_position
+        camera = self.engine.camera
+
+        if 0 <= screen_x < camera.width and 0 <= screen_y < camera.height:
+            x, y = camera.screen_to_map(screen_x, screen_y)
             if event.button == 1:
                 return self.on_index_selected(x, y)
```

**The area preview.** `AreaRangedAttackState.on_render` walks a bounding box around the cursor and tints `console.bg` tile by tile: the entity loop again, so guard, convert, write. The guard genuinely matters here: the box extends `radius + 1` beyond a cursor that can sit at the edge of the viewport, and as the earlier admonition warned, `console.bg` with an index past the edge raises `IndexError` while a negative one wraps to the opposite side of the screen.

```diff
+        camera = self.engine.camera
+
         for grid_y in range(min_y, max_y):
             for grid_x in range(min_x, max_x):
                 alpha = weights[grid_x, grid_y]
-                if alpha > 0:
-                    console.bg[grid_x, grid_y] = self.color.scale(alpha)
+                if alpha > 0 and camera.in_view(grid_x, grid_y):
+                    screen_x, screen_y = camera.map_to_screen(grid_x, grid_y)
+                    console.bg[screen_x, screen_y] = self.color.scale(alpha)
```

## Scroll margins: a camera with memory

The centered camera has one aesthetic flaw: it moves every single step, so the world never holds still and the `@` never visibly walks. Many games prefer **scroll margins** (also called a dead zone): the camera stays put while the player moves inside a comfortable inner rectangle, and only scrolls when the player pushes into the margin near an edge.

Add the margin to `game/data/config.py`:

```diff
 # Field of view
 FOV_RADIUS = 8
+
+# Camera
+SCROLL_MARGIN = 10
```

Then replace `follow` in `game/camera.py`. The file now needs the config import the rest of the project uses (`from game.data import config`); configuration is not game state, so the class stays easy to test:

```python
    def follow(
        self,
        target_x: int,
        target_y: int,
        map_width: int,
        map_height: int,
    ) -> bool:
        """Scroll only when the target pushes into a margin, then clamp.

        Returns True if the camera moved.
        """
        x = self._follow_axis(self.x, target_x, self.width, map_width)
        y = self._follow_axis(self.y, target_y, self.height, map_height)

        moved = (x, y) != (self.x, self.y)
        self.x = x
        self.y = y
        return moved

    @staticmethod
    def _follow_axis(position: int, target: int, viewport: int, map_size: int) -> int:
        margin = min(config.SCROLL_MARGIN, (viewport - 1) // 2)

        if target < position + margin:
            position = target - margin
        elif target >= position + viewport - margin:
            position = target - viewport + 1 + margin

        return max(0, min(position, map_size - viewport))
```

Two details in there do real work:

- **The margin is capped** at half the viewport. Without the cap, a viewport smaller than two margins would make the bands overlap, and every step would push the camera back and forth between them.
- **The clamp runs unconditionally**, outside the `if`. The margin logic only moves the camera when the player pushes it, so without the final clamp, a stale position could survive events that teleport the player: taking stairs to a floor with a different size, or loading a save. Recompute the bounds every frame and those cases fix themselves.

One more consequence: with margins the player can now stand as close as `margin` tiles from the viewport edge, so "everything visible is on screen" holds only while `FOV_RADIUS` stays at or below `SCROLL_MARGIN`. The defaults keep the promise (radius 8, margin 10); if you made the torch radius variable in the Part 4 exercises, cap it, or grow the margin with it.

Note what the margins cost you: the camera is no longer a pure function of the current state. Where it points now depends on how the player got there, so `self.x` and `self.y` have become real state, saved and restored with the `Engine` like everything else. That is the general trade: a nicer feel in exchange for state you must keep valid. The `moved` flag keeps working unchanged, and with it the mouse re-conversion from the previous section.

!!! info "Cameras are a craft of their own"

    Side-scrolling games spent the 1980s and 1990s refining exactly this problem. Super Mario Bros. famously only scrolled forward; later platformers added dead zones, look-ahead, and separate vertical rules that only scroll after you land.

    Itay Keren's GDC talk and article "Scroll Back: The Theory and Practice of Cameras in Side-Scrollers" catalogs dozens of these techniques, with animations, and is the standard reference. Almost everything in it translates directly to tile grids.

## What does not change

It is worth listing what this appendix never touched, because the list is the design working as intended. FOV runs on map arrays. Pathfinding costs, AI decisions, actions, combat, spawning, the message log, dungeon generation: all of them live purely in map coordinates and never noticed the camera. The camera is a lens between the game and the screen, not a change to the world, and the only code that had to move was code that touches the screen (rendering) or comes from the screen (the mouse).

## Try it

The changes touch rendering, the mouse, and targeting, so walk each border once:

- [ ] Walk toward each map edge: the camera stops, the `@` reaches the border, and no tiles appear on the opposite side of the screen.
- [ ] Cross the whole map: the HUD never moves and the message log keeps working.
- [ ] Hover a monster: the HUD names what is under the pointer, not a tile up and left of it.
- [ ] Hover the HUD panel: no names appear.
- [ ] Walk while the mouse rests over the map: the highlighted name updates as the world scrolls under the pointer.
- [ ] Aim a confusion scroll with the keyboard: the cursor cannot leave the screen.
- [ ] Aim a fireball at a tile near the viewport edge: the preview is cut at the border instead of crashing or painting the far side.
- [ ] Click a target: the spell hits the tile you clicked, not an offset one.
- [ ] Take the stairs and load a save: the view is correct on the first frame.
- [ ] With scroll margins on, walk small circles at the center of the screen: the camera holds still; push toward an edge: it follows.

## Summary

- The tutorial silently assumes map coordinates and screen coordinates are the same numbers; bigger maps require splitting the two spaces.
- A camera is the translation between them: one subtraction each way, plus a policy for where it points.
- The centered, clamped camera is stateless: recomputed from scratch each frame, it can never be wrong after stairs or loads.
- Rendering slices the map arrays through `camera.view`; entities convert per position, with an on-screen guard. Guard every drawing site yourself: `print` happens to clip, but numpy writes wrap or raise.
- Input converts at the boundary: one `Engine` method translates the mouse, rejects positions off the viewport, and runs again when the camera moves under a motionless pointer.
- Scroll margins buy a calmer view at the price of real state: cap the margin, clamp unconditionally, and save the camera with the engine.

## Exercise seeds

Ideas to grow this draft, roughly in order of effort:

1. **Derive the HUD from the viewport.** The HUD rows are still hardcoded (44 to 49, across `hud.py` and `Engine.render`). Introduce `HUD_PANEL_Y = VIEWPORT_HEIGHT` in `config.py` and turn every row into an offset from it, so changing the viewport height cannot desynchronize the map area and the HUD.
2. **Center small maps.** Let a floor be smaller than the viewport (a cramped vault level) and center it with a pair of offsets instead of pinning it to the top-left corner.
3. **A look mode.** A state that scrolls the camera freely with the movement keys, without moving the player, and snaps back on exit. The Part 9 `SelectIndexState` machinery is most of the work already done.
4. **Angband panels.** Change `follow` so the camera jumps by half a viewport instead of scrolling per tile. One conditional, a very different feel, and a taste of why terminals liked it: far fewer full redraws.
