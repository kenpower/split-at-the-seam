# Split at the seam

How to pull a game's **rules** out of the SFML loop so you can **test** them without opening a window.

This repository holds:

- lecture notes on *seams* (places where you can change what a program does without editing it), and
- **Catcher**, a small worked example: a basket catches falling fruit. It is shown before the split, after the split, and in six small steps in between, plus unit tests for the rules.

It accompanies the [16_SFML_Games](https://github.com/kenpower/16_SFML_Games) project. Catcher is not one of those 16 games, so you can copy the method but not the answer.

## Start here

| Read / open | What it is |
|-------------|------------|
| [seams_lecture_notes.md](https://github.com/kenpower/split-at-the-seam/blob/main/seams_lecture_notes.md) | The ideas: what a seam is and why it makes code testable. |
| [howto_split.md](https://github.com/kenpower/split-at-the-seam/blob/main/howto_split.md) | The step-by-step recipe, using Catcher. |
| [howto_split/before](howto_split/before) | Catcher before: everything in one loop. |
| [howto_split/steps](howto_split/steps) | One complete program per step (1 to 6). |
| [howto_split/after](howto_split/after) | Catcher after: the rules in `CatcherGame.h`, plus the tests. |

## Build and test

You need CMake 3.21 or newer and a C++17 compiler. SFML 3 is used if it is installed; otherwise CMake downloads and builds it for you (this takes a few minutes the first time). Catcher needs no image files.

### Visual Studio (easiest)

1. *File → Open → Folder...* and choose this folder.
2. Pick the **Debug** configuration preset in the toolbar.
3. *Build → Build All*.
4. Choose `catcher_before.exe` (or `catcher_step1.exe`, ..., `catcher_after.exe`) as the startup item and press F5.
5. *Test → Test Explorer → Run All Tests* runs the unit tests.

### Command line

```sh
cmake --workflow --preset default
```

This configures, builds and runs the tests (Release, in `build/`). The programs are in `build/bin/`. Use `--preset debug` for a Debug build in `build-debug/`.

To build only the tests (no SFML needed): `cmake -S . -B build -DCATCHER_BUILD_GAMES=OFF`, then `cmake --build build` and `ctest --test-dir build --output-on-failure`.

## Controls

Left and Right arrows move the basket. Space restarts after game over. The score and lives are shown in the window title.
