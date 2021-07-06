---
toc: true
---

# Part 2 - The generic Entity, the render functions, and the map

From here on the project will be organized into modules.
The following files will be added to the project:

```
/game/__init__.py
/game/actions.py
/game/engine.py
/game/entity.py
/game/game_map.py
/game/rendering.py
```

`/game/__init__.py` will be a blank file as it's only needed to [define a Python package](https://docs.python.org/3/tutorial/modules.html#packages).

```python
# /game/__init__.py
```

Then we make an `Engine` class which is kept in `/game/engine.py`:

```python
# /game/engine.py
from __future__ import annotations

import game.entity
import game.game_map


class Engine:
    game_map: game.game_map.GameMap
    player: game.entity.Entity
```

`from __future__ import annotations` tells Python to do [Postponed Evaluation of Annotations](https://www.python.org/dev/peps/pep-0563/), this helps reduce issues from modules referencing each other which can happen often whenever type-hinting is being used.
This will be added to all new modules.

The `Engine` class is missing its constructor method `__init__` which will normally be defined in all other classes.
It is excluded here because the objects held by `Engine` will need `Engine` to already exist for they can be created.
We add the annotations for `game_map` and `player` to the class itself and we will assign these variables manually after creating an instance of `Engine`.
`game_map` will be a reference to the active map.
`player` is a reference to the object currently controlled by the player.

We have a place to put the map in `Engine`.
We must now create `GameMap` at `/game/game_map.py`:

```python
# /game/game_map.py
from __future__ import annotations

from typing import Set

import numpy as np

import game.engine
import game.entity


class GameMap:
    def __init__(self, engine: game.engine.Engine, width: int, height: int):
        self.engine = engine
        self.width, self.height = width, height
        self.tiles = np.zeros((width, height), dtype=np.uint8, order="F")
        self.entities: Set[game.entity.Entity] = set()

    def in_bounds(self, x: int, y: int) -> bool:
        """Return True if x and y are inside of the bounds of this map."""
        return 0 <= x < self.width and 0 <= y < self.height
```

`GameMap` holds a reference to the `Engine`, its own shape, an array of tiles, and a set of entities.

`tiles` is an array of integers which defaults to 0, we won't need large numbers for this so the type is set to `np.uint8`.
For now zero will be used for walls and 1 will be used for floors.


`entities` is a [Python set](https://docs.python.org/3/library/stdtypes.html#set-types-set-frozenset).
A type-hint must be manually given since the assignment of `set()` can not determinate what the set contains.

`in_bounds` is added here that we won't have to copy this code every time we need to check if a coordinate is in the boundary of the map.

Next is `/game/entity.py`:

```python
from __future__ import annotations

from typing import Tuple

import game.game_map


class Entity:
    """A generic object to represent players, enemies, items, etc."""

    def __init__(
        self,
        gamemap: game.game_map.GameMap,
        x: int,
        y: int,
        char: str,
        color: Tuple[int, int, int],
    ):
        self.gamemap = gamemap
        gamemap.entities.add(self)
        self.x = x
        self.y = y
        self.char = char
        self.color = color
```

`Entity`'s hold their position and graphics.
They link themselves with a `GameMap`, adding themselves to its `entities` set and strong the `GameMap` in itself.

Now that the map and entity data is setup we can add a way to display their data in `/game/rendering.py`:

```python
from __future__ import annotations

import numpy as np
import tcod

import game.game_map

tile_graphics = np.array(
    [
        (ord("#"), (0x80, 0x80, 0x80), (0x40, 0x40, 0x40)),  # wall
        (ord("."), (0x40, 0x40, 0x40), (0x18, 0x18, 0x18)),  # floor
    ],
    dtype=tcod.console.rgb_graphic,
)


def render_map(console: tcod.Console, gamemap: game.game_map.GameMap) -> None:
    console.rgb[0 : gamemap.width, 0 : gamemap.height] = tile_graphics[gamemap.tiles]

    for entity in gamemap.entities:
        console.print(entity.x, entity.y, entity.char, fg=entity.color)
```

`tile_graphics` holds an array of tile graphics compatible with `tcod.Console`.
The first index starts at zero and will be the wall graphic, the 2nd will be the floor graphic.
[tcod.console.rgb_graphic](https://python-tcod.readthedocs.io/en/latest/tcod/console.html#tcod.console.rgb_graphic) is an alias for `np.dtype([("ch", np.intc), ("fg", "3B"), ("bg", "3B")])`, and is a [structured array](https://numpy.org/doc/stable/user/basics.rec.html) type.

The `render_map` function draws the map tiles and all entities to a console.

`tile_graphics` holds the graphics data and `gamemap.tiles` holds a 2D array of indexes into that data.
`tile_graphics[gamemap.tiles]` is a form of [integer indexing](https://www.tutorialspoint.com/numpy/numpy_advanced_indexing.htm) and will result in a 2D array of graphics at the correct positions, which is then assigned to `console.rgb`.
The shape of `console.rgb` is bigger than `gamemap.tiles` so it must be sliced until they both have the same shape.

The next step iterates over all the entities in the map and draws them using [Console.print](https://python-tcod.readthedocs.io/en/latest/tcod/console.html#tcod.console.Console.print).

The next module is `/game/actions.py` which is based loosely on [Bob Nystrom's "Is There More to Game Architecture than ECS?"](https://www.youtube.com/watch?v=JxI3Eu5DPwE).

```python
# /game/actions.py
from __future__ import annotations

import game.entity


class Action:
    def __init__(self, entity: game.entity.Entity) -> None:
        super().__init__()
        self.entity = entity  # The object performing the action.
        self.engine = entity.gamemap.engine

    def perform(self) -> None:
        """Perform this action now.

        This method must be overridden by Action subclasses.
        """
        raise NotImplementedError()

...
```

`Action` is created as an abstract class for all actions.
All actions hold the `entity` which they act on, as well as holding onto the `engine` for convenience.

The `perform` method is called to perform the action, in the base class this raises `NotImplementedError` as this is supposed to be overridden buy subclasses.

We continue adding code to `/game/actions.py`:

```python
...  # /game/actions.py (continued)

class Move(Action):
    def __init__(self, entity: game.entity.Entity, dx: int, dy: int):
        super().__init__(entity)

        self.dx = dx
        self.dy = dy

    def perform(self) -> None:
        dest_x = self.entity.x + self.dx
        dest_y = self.entity.y + self.dy

        if not self.engine.game_map.in_bounds(dest_x, dest_y):
            return  # Destination is out of bounds.
        if not self.engine.game_map.tiles[dest_x, dest_y]:
            return  # Destination is blocked by a tile.

        self.entity.x, self.entity.y = dest_x, dest_y
```

The `Move` class stores the direction of movement, which can be combined with the current `entity` position to get its destination.
Because this is in a module in a package the fully qualified name for `Move` is `game.actions.Move`, so it won't be necessary to add the word `Action` to every subclass of `Action`.

The last new module is `/game/input_handlers.py`:

```python
# /game/input_handlers.py
from __future__ import annotations

from typing import Optional, Union

import tcod

import game.actions
import game.engine
import game.rendering

MOVE_KEYS = {
    # Arrow keys.
    tcod.event.K_UP: (0, -1),
    tcod.event.K_DOWN: (0, 1),
    tcod.event.K_LEFT: (-1, 0),
    tcod.event.K_RIGHT: (1, 0),
    tcod.event.K_HOME: (-1, -1),
    tcod.event.K_END: (-1, 1),
    tcod.event.K_PAGEUP: (1, -1),
    tcod.event.K_PAGEDOWN: (1, 1),
    # Numpad keys.
    tcod.event.K_KP_1: (-1, 1),
    tcod.event.K_KP_2: (0, 1),
    tcod.event.K_KP_3: (1, 1),
    tcod.event.K_KP_4: (-1, 0),
    tcod.event.K_KP_6: (1, 0),
    tcod.event.K_KP_7: (-1, -1),
    tcod.event.K_KP_8: (0, -1),
    tcod.event.K_KP_9: (1, -1),
    # Vi keys.
    tcod.event.K_h: (-1, 0),
    tcod.event.K_j: (0, 1),
    tcod.event.K_k: (0, -1),
    tcod.event.K_l: (1, 0),
    tcod.event.K_y: (-1, -1),
    tcod.event.K_u: (1, -1),
    tcod.event.K_b: (-1, 1),
    tcod.event.K_n: (1, 1),
}

ActionOrHandler = Union["game.actions.Action", "EventHandler"]
"""An event handler return value which can trigger an action or switch active handlers.

If a handler is returned then it will become the active handler for future events.
If an action is returned it will be attempted and if it's valid then
MainGameEventHandler will become the active handler.
"""

...
```

`MOVE_KEYS` has been expanded to support diagonal movement and all common layouts.

`ActionOrHandler` is a custom [Union](https://docs.python.org/3/library/typing.html#typing.Union) type which is used by the following class.
Types were put in quotes here since the type they refer to are not available yet.

```python
...  # /game/input_handlers.py (continued)

class EventHandler(tcod.event.EventDispatch[ActionOrHandler]):
    def __init__(self, engine: game.engine.Engine) -> None:
        super().__init__()
        self.engine = engine

    def handle_events(self, event: tcod.event.Event) -> EventHandler:
        """Handle an event, perform any actions, then return the next active event handler."""
        action_or_state = self.dispatch(event)
        if isinstance(action_or_state, EventHandler):
            return action_or_state
        elif isinstance(action_or_state, game.actions.Action):
            return self.handle_action(action_or_state)
        return self

    def handle_action(self, action: game.actions.Action) -> EventHandler:
        """Handle actions returned from event methods."""
        action.perform()
        return self

    def ev_quit(self, event: tcod.event.Quit) -> Optional[game.actions.Action]:
        raise SystemExit(0)

    def ev_keydown(self, event: tcod.event.KeyDown) -> Optional[game.actions.Action]:
        key = event.sym

        if key in MOVE_KEYS:
            dx, dy = MOVE_KEYS[key]
            return game.actions.Move(self.engine.player, dx=dx, dy=dy)
        elif key == tcod.event.K_ESCAPE:
            raise SystemExit(0)

        return None

    def on_render(self, console: tcod.Console) -> None:
        game.rendering.render_map(console, self.engine.game_map)
```

The `EventHandler` class inherits from [tcod.event.EventDispatch](https://python-tcod.readthedocs.io/en/latest/tcod/event.html#tcod.event.EventDispatch).
The generic type is filled with `ActionOrHandler` which means the event methods (`ev_*`) can return a subclass of either the `Action` or `EventHandler` classes.

To be clear the `dispatch`, `ev_quit`, and `ev_keydown` methods are from `EventDispatch`, while the `__init__`, `handle_events`, `handle_action`, and `on_render` are unique to the new `EventHandler` class.

`handle_events` takes an event and calls [self.dispatch(event)](https://python-tcod.readthedocs.io/en/latest/tcod/event.html#tcod.event.EventDispatch.dispatch).
What happens next depends on the return value.
If an `Action` is returned then `handle_action` is called and will perform that action, followed by the handler returning itself as the handler to remain active.
If a `EventHandler` was returned then that handier will become active.
`None` can also be returned which will make the handier return itself.

Trying to close the window will trigger a call to `ev_quit`.
We quit the game in this case.

Key events trigger `ev_keydown`, which will return a `Move` action if the key is in `MOVE_KEYS` or quit the game if escape is pressed.
Unexpected keys will return `None` instead.

`on_render` is a custom method.
When `EventHandler` is the active handler the map will be rendered normally.

Now that the new modules are setup `/main.py` has to be modified to use them:

```diff
 #!/usr/bin/env python3
 import tcod
{{' '}}
-MOVE_KEYS = {
-    tcod.event.K_UP: (0, -1),
-    tcod.event.K_DOWN: (0, 1),
-    tcod.event.K_LEFT: (-1, 0),
-    tcod.event.K_RIGHT: (1, 0),
-}
+import game.engine
+import game.entity
+import game.game_map
+import game.input_handlers


 def main() -> None:
     screen_width = 80
     screen_height = 50
{{' '}}
-    player_x: int = screen_width // 2
-    player_y: int = screen_height // 2
+    map_width = 80
+    map_height = 45

     tileset = tcod.tileset.load_tilesheet("data/dejavu16x16_gs_tc.png", 32, 8, tcod.tileset.CHARMAP_TCOD)
{{' '}}
+    engine = game.engine.Engine()
+    engine.game_map = game.game_map.GameMap(engine, map_width, map_height)
+    engine.game_map.tiles[1:-1, 1:-1] = 1
+    engine.game_map.tiles[30:33, 22] = 0
+    engine.player = game.entity.Entity(engine.game_map, screen_width // 2, screen_height // 2, "@", (255, 255, 255))
+
+    game.entity.Entity(engine.game_map, screen_width // 2 - 5, screen_height // 2, "@", (255, 255, 0))  # NPC
+
+    event_handler = game.input_handlers.EventHandler(engine)
+
     with tcod.context.new(
         columns=screen_width,
         rows=screen_height,
```

`MOVE_KEYS` is no longer needed here and is removed.
The player will become an entity so the player position is no longer needed.

An instance of `Engine` must be created since everything else uses it to keep track of the games state.
We marked the class as having variables but these are not filled yet, so we must assign them now before the game loop starts.

A `GameMap` is created using the engine and a given size.
The current rendering setup means the map size must be smaller than the window size.
`engine.game_map.tiles[1:-1, 1:-1]` selects a view inside of the map that is one tile away from all the sides, this is [basic slicing](https://numpy.org/doc/stable/reference/arrays.indexing.html).
Assigning 1 to this view hollows out the map and leaves a 1-tile border of walls.
`engine.game_map.tiles[30:33, 22]` selects a small horizontal view which is then filled with walls.

Now the entity for the player is created in the center of the screen.
What makes this the player is assigning to `engine.player`, otherwise there is nothing special about this entity.

An NPC is added as a test entity.
The game map will keep track of this entity, so it is not assigned to any name.

Now that the engine is setup we create the `EventHandler` instance.

Next is the game loop:

```diff
         root_console = tcod.Console(screen_width, screen_height, order="F")
         while True:
             root_console.clear()
-            root_console.print(x=player_x, y=player_y, string="@")
+            event_handler.on_render(console=root_console)
             context.present(root_console)

             for event in tcod.event.wait():
-                if isinstance(event, tcod.event.Quit):
-                    raise SystemExit(0)
-                if isinstance(event, tcod.event.KeyDown):
-                    if event.sym in MOVE_KEYS:
-                        delta_x, delta_y = MOVE_KEYS[event.sym]
-                        dest_x = player_x + delta_x
-                        dest_y = player_y + delta_y
-                        if 0 <= dest_x < screen_width and 0 <= dest_y < screen_height:
-                            player_x, player_y = dest_x, dest_y
+                event_handler = event_handler.handle_events(event)
```

The code for drawing the player is replaced with `event_handler.on_render`, and the event handling code is replaced with `event_handler.handle_events`.

Nothing else is needed, when you run `main.py` you will now see a dungeon with a 1-tile border wall, a small wall in the middle, an NPC, and the player.
There will be an empty black area below the dungeon space, this area will be used later for a user interface.

You can see the current progress of this code in its entirety [here](https://github.com/TStand90/tcod_tutorial_v2/tree/2021/part-2).

[Continue to part 3](part-3).

[Return to the hub](.).
