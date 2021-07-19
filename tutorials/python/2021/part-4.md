---
toc: true
---

# Part 4 - Field of view

Currently the dungeon is fully visible at all times.
To add a sense of exploration we will begin tracking which tiles are explored or in view at all times.

First is to add two new arrays to `GameMap`.
`visible` is a boolean array of all currently visible tiles.
`explored` is a boolean array of all tiles that have ever been seen.
Add these changes to `/game/game_map.py`:
```diff
         self.entities: Set[game.entity.Entity] = set()
         self.enter_xy = (width // 2, height // 2)  # Entrance coordinates.
{{' '}}
+        self.visible = np.full((width, height), fill_value=False, order="F")  # Tiles the player can currently see
+        self.explored = np.full((width, height), fill_value=False, order="F")  # Tiles the player has seen before
+
     def in_bounds(self, x: int, y: int) -> bool:
```
An alternative way to create the above arrays is with `np.zeros((width, height), dtype=bool, order="F")`.

Python-tcod ports Libtcod's FOV algorithms with the [tcod.map.compute_fov](https://python-tcod.readthedocs.io/en/latest/tcod/map.html#tcod.map.compute_fov) function.
This function takes a 2D array where the zero values are opaque walls, and all non-zero tiles are transparent.
Because we've already used 0 for walls and 1 for floors the tiles of `GameMap.tiles` can be passed directory to this function.
The function returns which tiles are visible as a boolean array.

We now add a method to `Engine` to handle updating the FOV.
Make the following changes to `/game/engine.py`:
```diff
 import random
{{' '}}
+import tcod
+
 import game.entity
 ...
     player: game.entity.Entity
     rng: random.Random
+
+    def update_fov(self) -> None:
+        """Recompute the visible area based on the players point of view."""
+        self.game_map.visible[:] = tcod.map.compute_fov(
+            self.game_map.tiles,
+            (self.player.x, self.player.y),
+            radius=8,
+            algorithm=tcod.FOV_SYMMETRIC_SHADOWCAST,
+        )
+        # If a tile is currently "visible" it will also be marked as "explored".
+        self.game_map.explored |= self.game_map.visible
```


We must call this function after every action is performed in `/game/input_handlers.py`:
```diff
     def handle_action(self, action: game.actions.Action) -> EventHandler:
         """Handle actions returned from event methods."""
         action.perform()
+        self.engine.update_fov()
         return self
```
The above will only update the visible area after the player moves.
Because of that we also need to update it before the first turn, so another call gets added to `/main.py` after the map and player is setup:
```diff
     engine.player = game.entity.Entity(engine.game_map, *engine.game_map.enter_xy, "@", (255, 255, 255))
+    engine.update_fov()

     event_handler = game.input_handlers.EventHandler(engine)
```

Now it is time to modify the rendering function.
We need a default graphic for the unexplored area so a NumpPy scalar called `SHROUD` (a thing that obscures) is made, which is a blank black tile.
Tiles in the current view will have the graphics stay the same as before.
We use `tile_graphics[gamemap.tiles]` as before to generate the entire array but will render only the visible area by the end, this will be called `light`.
A copy of `light` is made called `dark` and this array modified to be darker with the foreground at half brightness and the background at 1/8th brightness.

Now [np.select](https://numpy.org/doc/stable/reference/generated/numpy.select.html) is used to determine which
If a value on `gamemap.visible` is True then the value at `light` will be shown, then `gamemap.explored` is checked and if that is True then `dark` is shown, otherwise `SHROUD` is used.
Because `SHROUD` is a scalar it is [broadcast](https://numpy.org/doc/stable/user/basics.broadcasting.html) along the entire array.
The `default` parameter could have been another 2D array, or the arrays in `condlist` could have been scalars like `SHROUD`, the important part is that these fit the shape of `console.rgb[0 : gamemap.width, 0 : gamemap.height]` acorind to the [broadcasting rules](https://numpy.org/doc/stable/user/basics.broadcasting.html).

The final step is the entities, any entity that is not on a visible tile is not printed to the screen.

With this in mind we can edit `/game/rendering.py`:
```diff
+# SHROUD represents unexplored, unseen tiles
+SHROUD = np.array((ord(" "), (255, 255, 255), (0, 0, 0)), dtype=tcod.console.rgb_graphic)
+

 def render_map(console: tcod.Console, gamemap: game.game_map.GameMap) -> None:
-    console.rgb[0 : gamemap.width, 0 : gamemap.height] = tile_graphics[gamemap.tiles]
+    # The default graphics are of tiles that are visible.
+    light = tile_graphics[gamemap.tiles]
+
+    # Apply effects to create a darkened map of tile graphics.
+    dark = light.copy()
+    dark["fg"] //= 2
+    dark["bg"] //= 8
+
+    # If a tile is in the "visible" array, then draw it with the "light" colors.
+    # If it isn't, but it's in the "explored" array, then draw it with the "dark" colors.
+    # Otherwise, the default graphic is "SHROUD".
+    console.rgb[0 : gamemap.width, 0 : gamemap.height] = np.select(
+        condlist=[gamemap.visible, gamemap.explored],
+        choicelist=[light, dark],
+        default=SHROUD,
+    )

     for entity in gamemap.entities:
+        if not gamemap.visible[entity.x, entity.y]:
+            continue  # Skip entities that are not in the FOV.
         console.print(entity.x, entity.y, entity.char, fg=entity.color)
```

You can see the current progress of this code in its entirety [here](https://github.com/TStand90/tcod_tutorial_v2/tree/2021/part-4).

[Continue to part 5](part-5).

[Return to the hub](.).
