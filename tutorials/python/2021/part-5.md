---
toc: true
---

# Part 5 - Placing enemies and kicking them (harmlessly)

Add the `name` and `blocks_movement` attributes to the `Entity` class in `/game/entity.py`:
```diff
         y: int,
         char: str,
         color: Tuple[int, int, int],
+        name: str,
+        blocks_movement: bool = True,
     ):
 ...
         self.char = char
         self.color = color
+        self.name = name
+        self.blocks_movement = blocks_movement
```

Since the added `name` parameter is not optional the current instantiations of `Entity` will break.
[Mypy](https://mypy.readthedocs.io/en/stable/) can detect all places where a created `Entity` must have a `name` parameter added.

Add a `name` to the player entity in `/main.py`:
```diff
         engine=engine,
     )
-    engine.player = game.entity.Entity(engine.game_map, *engine.game_map.enter_xy, "@", (255, 255, 255))
+    engine.player = game.entity.Entity(
+        engine.game_map, *engine.game_map.enter_xy, char="@", color=(255, 255, 255), name="Player"
+    )
     engine.update_fov()
```

It will be common to search for if an entity is blocking a specific position.
We add a method to do this to `GameMap` in `/game/game_map.py`:
```diff
 from __future__ import annotations

-from typing import Set
+from typing import Optional, Set

 import numpy as np

 ...
         self.visible = np.full((width, height), fill_value=False, order="F")  # Tiles the player can currently see
         self.explored = np.full((width, height), fill_value=False, order="F")  # Tiles the player has seen before
{{' '}}
+    def get_blocking_entity_at(self, x: int, y: int) -> Optional[game.entity.Entity]:
+        """Returns an entity that blocks the position at x,y if one exists, otherwise returns None."""
+        for entity in self.entities:
+            if entity.blocks_movement and entity.x == x and entity.y == y:
+                return entity
+
+        return None
+
     def in_bounds(self, x: int, y: int) -> bool:
```

This is a basic implementation to check if a position is occupied by a blocking entity.
A [Spatial Partition](https://gameprogrammingpatterns.com/spatial-partition.html) would scale better, but for now the tutorial will not be using an excessive amount of entities.

Next a function is added to `/game/procgen.py` to handle the placement of entities.
```diff
def intersects(self, other: RectangularRoom) -> bool:
         return self.x1 <= other.x2 and self.x2 >= other.x1 and self.y1 <= other.y2 and self.y2 >= other.y1


+def place_entities(room: RectangularRoom, dungeon: game.game_map.GameMap, maximum_monsters: int) -> None:
+    rng = dungeon.engine.rng
+    number_of_monsters = rng.randint(0, maximum_monsters)
+
+    for _ in range(number_of_monsters):
+        x = rng.randint(room.x1 + 1, room.x2 - 1)
+        y = rng.randint(room.y1 + 1, room.y2 - 1)
+
+        if dungeon.get_blocking_entity_at(x, y):
+            continue
+        if (x, y) == dungeon.enter_xy:
+            continue
+
+        if rng.random() < 0.8:
+            game.entity.Entity(dungeon, x, y, char="o", color=(63, 127, 63), name="Orc")
+        else:
+            game.entity.Entity(dungeon, x, y, char="T", color=(0, 127, 0), name="Troll")
+
+
 def tunnel_between(
```

This attempts to place a random monster but will give up if the location would overlap with the dungeon entrance or another monster.

`place_entities` must be called from the `generate_dungeon` function in `/game/procgen.py`:
```diff
     room_max_size: int,
     map_width: int,
     map_height: int,
+    max_monsters_per_room: int,
     engine: game.engine.Engine,
 ) -> game.game_map.GameMap:
 ...
             for x, y in tunnel_between(engine, rooms[-1].center, new_room.center):
                 dungeon.tiles[x, y] = FLOOR
{{' '}}
+        place_entities(new_room, dungeon, max_monsters_per_room)
+
         # Finally, append the new room to the list.
         rooms.append(new_room)
```

We then add a `max_monsters_per_room` config value in `/main.py`:
```diff
     room_max_size = 10
     room_min_size = 6
     max_rooms = 30
+    max_monsters_per_room = 2

     tileset = tcod.tileset.load_tilesheet("data/dejavu16x16_gs_tc.png", 32, 8, tcod.tileset.CHARMAP_TCOD)
 ...
         room_max_size=room_max_size,
         map_width=map_width,
         map_height=map_height,
+        max_monsters_per_room=max_monsters_per_room,
         engine=engine,
     )
```

Finally we can have entities block movement in the `Move` action at `/game/actions.py` by adding a condition:
```diff
         if not self.engine.game_map.tiles[dest_x, dest_y]:
             return  # Destination is blocked by a tile.
+        if self.engine.game_map.get_blocking_entity_at(dest_x, dest_y):
+            return  # Destination is blocked by an entity.

         self.entity.x, self.entity.y = dest_x, dest_y
```

At this point enemies should spawn in the dungeon which can block the players movement.

To make it easier to add more actions an `ActionWithDirection` class is added.
This takes the `__init__` method from `Move` so that all subclasses of `ActionWithDirection` can have a direction like `Move` has.
`Move` has its `__init__` removed and becomes a subclass of `ActionWithDirection`.
`Move` keeps its `perform` method.
Add these changes to `/game/actions.py`:
```diff
-class Move(Action):
+class ActionWithDirection(Action):
     def __init__(self, entity: game.entity.Entity, dx: int, dy: int):
         super().__init__(entity)

         self.dx = dx
         self.dy = dy
{{' '}}
+    def perform(self) -> None:
+        raise NotImplementedError()
+
+
+class Move(ActionWithDirection):
     def perform(self) -> None:
```

We now use the `ActionWithDirection` class to add two actions to the end of `/game/actions.py`:
```python
...  # /game/actions.py (continued)

class Melee(ActionWithDirection):
    def perform(self) -> None:
        dest_x = self.entity.x + self.dx
        dest_y = self.entity.y + self.dy
        target = self.engine.game_map.get_blocking_entity_at(dest_x, dest_y)
        if not target:
            return  # No entity to attack.

        print(f"You kick the {target.name}, much to its annoyance!")


class Bump(ActionWithDirection):
    def perform(self) -> None:
        dest_x = self.entity.x + self.dx
        dest_y = self.entity.y + self.dy

        if self.engine.game_map.get_blocking_entity_at(dest_x, dest_y):
            return Melee(self.entity, self.dx, self.dy).perform()
        else:
            return Move(self.entity, self.dx, self.dy).perform()
```

In `/game/input_handlers.py` the `Move` action is replaced with `Bump`:
```diff
         if key in MOVE_KEYS:
             dx, dy = MOVE_KEYS[key]
-            return game.actions.Move(self.engine.player, dx=dx, dy=dy)
+            return game.actions.Bump(self.engine.player, dx=dx, dy=dy)
         elif key == tcod.event.K_ESCAPE:
             raise SystemExit(0)
```

Now you can interact with enemies in a simple extendable way.

Instead of using `print` everywhere the [standard logging module](https://docs.python.org/3/library/logging.html) will be used for output that is not in-game.
Enable logging at the debug level in `/main.py`:
```diff
 #!/usr/bin/env python3
+import logging
 import random

 ...
 if __name__ == "__main__":
+    if __debug__:
+        logging.basicConfig(level=logging.DEBUG)
     main()
```

Then add a logger to `/game/engine.py`.
We also add a `handle_enemy_turns` function to `Engine` and log the results.
```diff
 from __future__ import annotations
{{' '}}
+import logging
 import random

 import tcod
 ...
 import game.entity
 import game.game_map
{{' '}}
+logger = logging.getLogger(__name__)
+

 class Engine:
     game_map: game.game_map.GameMap
     player: game.entity.Entity
     rng: random.Random
{{' '}}
+    def handle_enemy_turns(self) -> None:
+        logger.info("Enemy turn.")
+        for entity in self.game_map.entities - {self.player}:
+            logger.info(f"The {entity.name} wonders when it will get to take a real turn.")
+
     def update_fov(self) -> None:
```

You add `logger = logging.getLogger(__name__)` to every module which you plan on logging.

`self.game_map.entities - {self.player}` makes a copy of the `entities` set with the player removed.
It is important that a copy is iterated over, since later steps will modify the set during the loop.

Add a call to `handle_enemy_turns` in `/game/input_handlers.py`.
```diff
     def handle_action(self, action: game.actions.Action) -> EventHandler:
         """Handle actions returned from event methods."""
         action.perform()
+        self.engine.handle_enemy_turns()
         self.engine.update_fov()
         return self
```

The order of these functions is important.
`action.perform` always has to be first, and `self.engine.update_fov` has to be right before we return control over to the player.

You can see the current progress of this code in its entirety [here](https://github.com/TStand90/tcod_tutorial_v2/tree/2021/part-5).

Part 6 is still in development.

[Return to the hub](.).
