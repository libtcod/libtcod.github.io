---
toc: true
---

# Part 1 - Drawing the '@' symbol and moving it around

Welcome to part 1 of this tutorial! This series will help you create your very first roguelike game, written in Python\!

The 2021 tutorial is largely based off [the 2020 tutorial](http://rogueliketutorials.com/tutorials/tcod/v2/), and is primarily a smaller update rather than a larger rewrite that the previous versions were.

This part assumes that you have either checked [Part 0](part-0) or are already familiar with Python and Python-tcod.
If not, be sure to check that page, and make sure that you've got Python and TCOD installed, as well as having an IDE or editor ready.

Assuming that you've done all that, let's get started.

We will start by setting up the following directory structure.
The font from [Part 0](part-0) will go into a `data` directory, then `main.py` will be created as an entry point.

```
/data/dejavu16x16_gs_tc.png
/main.py
```

`/main.py` is the entry point of the program.  You can use `python main.py` to start the program after the following is implemented.

```python
#!/usr/bin/env python3
# main.py
import tcod

MOVE_KEYS = {
    tcod.event.K_UP: (0, -1),
    tcod.event.K_DOWN: (0, 1),
    tcod.event.K_LEFT: (-1, 0),
    tcod.event.K_RIGHT: (1, 0),
}


def main() -> None:
    screen_width = 80
    screen_height = 50

    player_x: int = screen_width // 2
    player_y: int = screen_height // 2

    tileset = tcod.tileset.load_tilesheet("data/dejavu16x16_gs_tc.png", 32, 8, tcod.tileset.CHARMAP_TCOD)

    with tcod.context.new(
        columns=screen_width,
        rows=screen_height,
        tileset=tileset,
        title="Yet Another Roguelike Tutorial",
        vsync=True,
    ) as context:
        root_console = tcod.Console(screen_width, screen_height, order="F")
        while True:
            root_console.clear()
            root_console.print(x=player_x, y=player_y, string="@")
            context.present(root_console)

            for event in tcod.event.wait():
                if isinstance(event, tcod.event.Quit):
                    raise SystemExit(0)
                if isinstance(event, tcod.event.KeyDown):
                    if event.sym in MOVE_KEYS:
                        delta_x, delta_y = MOVE_KEYS[event.sym]
                        dest_x = player_x + delta_x
                        dest_y = player_y + delta_y
                        if 0 <= dest_x < screen_width and 0 <= dest_y < screen_height:
                            player_x, player_y = dest_x, dest_y


if __name__ == "__main__":
    main()
```

`#!/usr/bin/env python3` is a [shebang](https://en.wikipedia.org/wiki/Shebang_(Unix)) and must be the first line to be useful.
It is normally used to make scripts executable on Linux, but is also used by the Windows Python launcher.

`tcod` is imported along with the two other modules we've added.

`MOVE_KEYS` is a Python dictionary mapping of [tcod KeySym's](https://python-tcod.readthedocs.io/en/latest/tcod/event.html#tcod.event.KeySym) to in-game directions.
This will be expanded in later parts.

The `main` function will be the entry point of the program.
There's nothing special about the name other than the terminology, the special nature of this function comes from the `__name__ == "__main__"` condition at the bottom of the script which is only True when the script is directly run, compared to importing main from an interactive prompt.

`main` starts by setting the screen size, then sets the player position to the center of the screen.
The floor division operator is used so that the numbers don't promote to a float type.

The tileset is loaded with [tcod.tileset.load_tilesheet](https://python-tcod.readthedocs.io/en/latest/tcod/tileset.html#tcod.tileset.load_tilesheet)
The Python-tcod docs have a [character reference](https://python-tcod.readthedocs.io/en/latest/tcod/charmap-reference.html) to keep track of which layouts have what glyphs.

[tcod.context.new](https://python-tcod.readthedocs.io/en/latest/tcod/context.html#tcod.context.new) is used to setup the window and returns a [Context](https://python-tcod.readthedocs.io/en/latest/tcod/context.html#tcod.context.Context) instance.
Contexts must be closed once you're done with them, but the `with` statement will do this automatically when exiting the with-block.

Then a [tcod.Console](https://python-tcod.readthedocs.io/en/latest/tcod/console.html#tcod.console.Console) is created with the same size that was given to `tcod.context.new`,
`order="F"` sets the console arrays to be indexed in `x, y` order.
This is a convenient way to handle array indexes so we'll use `order="F"` a lot when working with NumPy.
You might wonder why `order="F"` isn't the default [and there is an explanation for that](https://numpy.org/doc/stable/reference/internals.html#multidimensional-array-indexing-order-issues).
These arrays are not being used yet.

Now with `while True:` the game-loop begins, with the only way of existing this loop normally being the `SystemExit` exceptions implemented later.
`root_console` is cleared, then the player position is printed to `root_console`, then `root_console` is displayed using `context.present(root_console)`.

The for-loop waits for there to be events then iterates over all events until none are left.
This is an efficient way to handle events when the game doesn't have real-time animations or mechanics.
If you have real-time effects then you should replace [tcod.event.wait](https://python-tcod.readthedocs.io/en/latest/tcod/event.html#tcod.event.wait) with [tcod.event.get](https://python-tcod.readthedocs.io/en/latest/tcod/event.html#tcod.event.get).

[isinstance](https://docs.python.org/3/library/functions.html#isinstance) is being used here to sort events by type.
This has an effect on type-checking, where anything within an `isinstance` branch can be assumed to actually be that type, much like a type cast in any other language.

Trying to close the window will result in a `Quit` event.
When this happens we will raise [SystemExit](https://docs.python.org/3/library/exceptions.html#SystemExit) which will propagate and terminate the script.

For `KeyDown` events we check if the key is in the `MOVE_KEYS` dictionary then try to move by that amount.
If the destination is on the screen then the player moves to that spot.

You can see the current progress of this code in its entirety [here](https://github.com/TStand90/tcod_tutorial_v2/tree/2021/part-1).

If you want to distribute your game then [you can set that up now or at any time later](distribution).

[Continue to part 2](part-2).

[Return to the hub](.).
