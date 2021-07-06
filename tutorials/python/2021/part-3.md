---
toc: true
---

# Part 3 - Generating a dungeon

Let us prepare things so that we can generate a dungeon.

We will add a new variable to GameMap to keep track of the entrance of the dungeon.
This will track the players starting position instead of having to do anything *funny* with generating the player entity.
Add the following to `/game/game_map.py`:

```diff
         self.width, self.height = width, height
         self.tiles = np.zeros((width, height), dtype=np.uint8, order="F")
         self.entities: Set[game.entity.Entity] = set()
+        self.enter_xy = (width // 2, height // 2)  # Entrance coordinates.

     def in_bounds(self, x: int, y: int) -> bool:
         """Return True if x and y are inside of the bounds of this map."""
```

This will be the start of using random numbers.
We will use Python's [random module](https://docs.python.org/3/library/random.html) for this, and we are going to store the random state in `Engine` instead of relying on the random modules internal state.
So add the following changes to `/game/engine.py`:

```diff
 from __future__ import annotations
{{" "}}
+import random
+
 import game.entity
 import game.game_map


 class Engine:
     game_map: game.game_map.GameMap
     player: game.entity.Entity
+    rng: random.Random
```

Now for the random dungeon generator.

```python
# /game/procgen.py
from __future__ import annotations

from typing import Iterator, List, Tuple

import tcod

import game.engine
import game.entity
import game.game_map

WALL = 0
FLOOR = 1


class RectangularRoom:
    def __init__(self, x: int, y: int, width: int, height: int):
        self.x1 = x
        self.y1 = y
        self.x2 = x + width
        self.y2 = y + height

    @property
    def center(self) -> Tuple[int, int]:
        """Return the center coordinates of the room."""
        return (self.x1 + self.x2) // 2, (self.y1 + self.y2) // 2

    @property
    def inner(self) -> Tuple[slice, slice]:
        """Return the inner area of this room as a 2D array index."""
        return slice(self.x1 + 1, self.x2), slice(self.y1 + 1, self.y2)

    def intersects(self, other: RectangularRoom) -> bool:
        """Return True if this room overlaps with another RectangularRoom."""
        return self.x1 <= other.x2 and self.x2 >= other.x1 and self.y1 <= other.y2 and self.y2 >= other.y1

...
```

The `RectangularRoom` class will be used to manage the placement of rooms.
We have `center` to get the center of the room in integer coordinates, `inner` to perform [basic slicing](https://numpy.org/doc/stable/reference/arrays.indexing.html) on NumPy arrays, and `intersects` to detect collisions with other rooms to prevent rooms from overlapping.

The [@property](https://docs.python.org/3/library/functions.html#property) decorator means that these attributes act like a read-only variable rather than a method.

```python
...  # # /game/procgen.py (continued)

def tunnel_between(
    engine: game.engine.Engine, start: Tuple[int, int], end: Tuple[int, int]
) -> Iterator[Tuple[int, int]]:
    """Return an L-shaped tunnel between these two points."""
    x1, y1 = start
    x2, y2 = end
    if engine.rng.random() < 0.5:  # 50% chance.
        corner_x, corner_y = x2, y1  # Move horizontally, then vertically.
    else:
        corner_x, corner_y = x1, y2  # Move vertically, then horizontally.

    # Generate the coordinates for this tunnel.
    for x, y in tcod.los.bresenham((x1, y1), (corner_x, corner_y)).tolist():
        yield x, y
    for x, y in tcod.los.bresenham((corner_x, corner_y), (x2, y2)).tolist():
        yield x, y

...
```

`tunnel_between` iterates over the `x, y` coordinates of a tunnel between `start` and `end` including both endpoints.
`engine.rng` is used from random numbers, this object has all of the typical functions you'd expect from the [random module](https://docs.python.org/3/library/random.html).
[tcod.los.bresenham](https://python-tcod.readthedocs.io/en/latest/tcod/los.html#tcod.los.bresenham) is used to generate the straight lines.


This function is a Python [generator](https://docs.python.org/3/howto/functional.html#generators) due to its use of `yield` statements.

```python
...  # /game/procgen.py (continued)

def generate_dungeon(
    max_rooms: int,
    room_min_size: int,
    room_max_size: int,
    map_width: int,
    map_height: int,
    engine: game.engine.Engine,
) -> game.game_map.GameMap:
    """Generate a new dungeon map."""
    dungeon = game.game_map.GameMap(engine, map_width, map_height)

    rooms: List[RectangularRoom] = []

    for _ in range(max_rooms):
        room_width = engine.rng.randint(room_min_size, room_max_size)
        room_height = engine.rng.randint(room_min_size, room_max_size)

        x = engine.rng.randint(0, dungeon.width - room_width - 1)
        y = engine.rng.randint(0, dungeon.height - room_height - 1)

        # "RectangularRoom" class makes rectangles easier to work with.
        new_room = RectangularRoom(x, y, room_width, room_height)

        # Run through the other rooms and see if they intersect with this one.
        if any(new_room.intersects(other_room) for other_room in rooms):
            continue  # This room intersects, so go to the next attempt.
        # If there are no intersections then the room is valid.

        # Dig out this rooms inner area.
        dungeon.tiles[new_room.inner] = FLOOR

        if len(rooms) == 0:
            # The first room, where the player starts.
            dungeon.enter_xy = new_room.center
        else:  # All rooms after the first.
            # Dig out a tunnel between this room and the previous one.
            for x, y in tunnel_between(engine, rooms[-1].center, new_room.center):
                dungeon.tiles[x, y] = FLOOR

        # Finally, append the new room to the list.
        rooms.append(new_room)

    return dungeon
```

`(new_room.intersects(other_room) for other_room in rooms)` is a [generator expression](https://docs.python.org/3/howto/functional.html#generator-expressions-and-list-comprehensions), and when combined with the [any](https://docs.python.org/3/library/functions.html#any) function this quickly tests if any rooms intersect.

Hopefully this function is documented well enough to be understandable.
The [2020 tutorial already had a good explain on how this generator works](http://rogueliketutorials.com/tutorials/tcod/v2/part-3/).

With the generator setup it is now time to update `/main.py` to use it:

```diff

 import game.entity
 import game.game_map
 import game.input_handlers
+import game.procgen


 def main() -> None:
 ...
     map_width = 80
     map_height = 45
{{' '}}
+    room_max_size = 10
+    room_min_size = 6
+    max_rooms = 30
+
     tileset = tcod.tileset.load_tilesheet("data/dejavu16x16_gs_tc.png", 32, 8, tcod.tileset.CHARMAP_TCOD)

     engine = game.engine.Engine()
-    engine.game_map = game.game_map.GameMap(engine, map_width, map_height)
-    engine.game_map.tiles[1:-1, 1:-1] = 1
-    engine.game_map.tiles[30:33, 22] = 0
-    engine.player = game.entity.Entity(engine.game_map, screen_width // 2, screen_height // 2, "@", (255, 255, 255))
-
-    game.entity.Entity(engine.game_map, screen_width // 2 - 5, screen_height // 2, "@", (255, 255, 0))  # NPC
+    engine.rng = random.Random()
+    engine.game_map = game.procgen.generate_dungeon(
+        max_rooms=max_rooms,
+        room_min_size=room_min_size,
+        room_max_size=room_max_size,
+        map_width=map_width,
+        map_height=map_height,
+        engine=engine,
+    )
+    engine.player = game.entity.Entity(engine.game_map, *engine.game_map.enter_xy, "@", (255, 255, 255))

     event_handler = game.input_handlers.EventHandler(engine)
```

The dungeon testing code is removed and replaced with the dungeon generator.
`engine.rng` has to be set before `generate_dungeon` is called.
`random.Random()` with no parameter will generate a random seed, but you can give a seed manually if you need a reproducible dungeon layout for testing.

The player is now created with the coordinates from `engine.game_map.enter_xy`.
The `enter_xy` tuple is unpacked into the function call, so the tuple is spread across the `x` and `y` parameters without having to reference the tuple twice.

You can see the current progress of this code in its entirety [here](https://github.com/TStand90/tcod_tutorial_v2/tree/2021/part-3).

Part 4 is still in development.

[Return to the hub](.).
