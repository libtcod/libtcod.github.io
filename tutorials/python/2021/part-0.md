---
---

# Part 0 - Setting Up

#### Prior knowledge

This tutorial assumes some basic familiarity with programming in general, and with Python.
If you've never used Python before, this tutorial could be a little confusing. There are many free resources online about learning programming and Python, and I'd recommend learning about objects and functions in Python at the very least before attempting to read this tutorial.
The Python documentation [includes a good tutorial to get started with Python](https://docs.python.org/3/tutorial/).

... Of course, there are those who have ignored this advice and done well with this tutorial anyway, so feel free to ignore that last paragraph if you're feeling bold\!

You should keep track of the [Python Documentation](https://docs.python.org/3/), the [NumPy Documentation](https://numpy.org/doc/stable/), and the [python-tcod Documentation](https://python-tcod.readthedocs.io/en/latest/).
External references to these will be made throughout the tutorial as they become relevant.

#### Installation

To do this tutorial, you'll need Python version 3.8 or higher.
The latest version of Python is recommended (currently 3.9 as of June 2021).

[Download Python here](https://www.python.org/downloads/).

You'll also want the latest version of the TCOD library, which is what this tutorial is based on.
[Installation instructions for TCOD can be found here.](https://python-tcod.readthedocs.io/en/latest/installation.html)

While you can certainly install TCOD and complete this tutorial without it, I'd highly recommend using a virtual environment.
[Documentation on how to do that can be found here.](https://docs.python.org/3/library/venv.html)

Additionally, if you are going to use a virtual environment, you may want to take the time to set up a `requirements.txt` file.
This will allow you to track your project dependencies if you add any in the future, and more easily install them if you need to (for example, if you pull from a remote git repository.)

You can set up your `requirements.txt` file in the same directory that you plan on working in for the project. Create the file `requirements.txt` and put the following in it:

```
tcod>=12.6.2
numpy>=1.21.0
```

Once that's done, with your virtual environment activated, type the following command:

`pip install -r requirements.txt`

This should install the TCOD library, along with its dependency, numpy.

Depending on your computer, you might also need to install SDL2.
Check the instructions for installing it based on your operating system.
For example, Ubuntu can install it with the following command:

`sudo apt-get install libsdl2-dev`

#### Editors

Any text editor can work for writing Python.
You could even use Notepad if you really wanted to.
Personally, I'm a fan of [Pycharm](https://www.jetbrains.com/pycharm/) and [Visual Studio Code](https://code.visualstudio.com/).
Whatever you choose, I strongly recommend something that can help catch Python syntax errors at the very least.
I've been working with Python for over five years, and I still make these types of mistakes all the time\!

#### Making sure Python works

To verify that your installation of both Python 3 and TCOD are working, create a new file (in whatever directory you plan on using for the tutorial) called `main.py`, and enter the following text into it:

```py3
#!/usr/bin/env python3
import tcod


def main():
    print("Hello World!")


if __name__ == "__main__":
    main()
```

Run the file in your terminal (or alternatively in your editor, if possible):

`python main.py`

You should see "Hello World\!" printed out to the terminal. If you receive an error, there is probably an issue with either your Python or TCOD installation.

### Downloading the Image File

For this tutorial, we'll need an image file. The default one is provided below.

![Font File](https://raw.githubusercontent.com/libtcod/libtcod/master/data/fonts/dejavu16x16_gs_tc.png)

Right click the image and save it to `/data/dejavu16x16_gs_tc.png` in your project directory.
Other options for fonts [are available here](https://github.com/libtcod/libtcod/tree/master/data/fonts), but you'll need a font ending in `_tc.png` for this tutorial.

### Getting help

Be sure to check out the [Roguelike Development Subreddit](https://www.reddit.com/r/roguelikedev) for help.
There's a link there to the Discord channel as well.

-----

### Ready to go?

Once you're set up and ready to go, you can proceed to [Part 1](part-1).
