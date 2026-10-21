# colcon

**colcon** (*collective construction*) is the build tool for ROS 2 workspaces. It finds every package in `src`, works out the order they must be built in from their dependencies, builds each one with the right tool (setuptools for Python, CMake for C++), and installs the results into `install/`.

## Topics Covered

- `colcon build` and what it does
- Building only some packages
- `--symlink-install`
- Controlling build output
- Running tests with `colcon test`
- Inspecting a workspace with `colcon list` and `colcon graph`
- Cleaning and common build errors

---

## `colcon build`

Always run colcon from the **workspace root**:

```bash
$ cd ~/ros2_ws
$ colcon build
Starting >>> my_robot_interfaces
Finished <<< my_robot_interfaces [4.12s]
Starting >>> my_first_pkg
Starting >>> my_robot_control
Finished <<< my_first_pkg [1.02s]
Finished <<< my_robot_control [1.31s]

Summary: 3 packages finished [5.60s]
```

colcon read each package's `package.xml`, saw that the other packages depend on `my_robot_interfaces`, and built that one first. Independent packages are built in parallel.

> ⚠️ Running `colcon build` inside `src` or a package folder creates a second set of `build/install/log` folders in the wrong place. If that happens, delete them and run colcon from the workspace root.

After building, source the workspace (in a new terminal) to use the result:

```bash
$ source install/setup.bash
```

---

## Building Only Some Packages

Building everything gets slow as a workspace grows. Select packages instead:

| Option | Builds |
|--------|--------|
| `--packages-select <pkg> ...` | Only the listed packages |
| `--packages-up-to <pkg> ...` | The listed packages **and** everything they depend on |
| `--packages-skip <pkg> ...` | Everything except the listed packages |
| `--packages-above <pkg> ...` | The listed packages and everything that depends on them |

```bash
$ colcon build --packages-select my_first_pkg
$ colcon build --packages-up-to my_robot_control
```

Use `--packages-select` while iterating on one package, and `--packages-up-to` when its dependencies have changed too.

---

## `--symlink-install`

By default, colcon **copies** files into `install/`. Every time you edit a Python file, a launch file, or a YAML file, you have to rebuild before the change takes effect.

With `--symlink-install`, colcon creates **links** to your source files instead of copies:

```bash
$ colcon build --symlink-install
```

Now edits to existing Python files, launch files, and config files take effect immediately. Just restart the node.

You still need to rebuild when you:

- **Add** a new file (a new launch file, a new node module)
- Change `setup.py`, `setup.cfg`, `package.xml`, or `CMakeLists.txt`, for example to add an executable
- Change **C++** code (it always needs compiling)

> 💡 Use `--symlink-install` consistently. Switching between symlink and normal builds in the same workspace can cause errors. If that happens, clean the workspace and rebuild.

---

## Controlling Build Output

By default, colcon hides compiler output unless something fails. Useful options:

| Option | Effect |
|--------|--------|
| `--event-handlers console_direct+` | Show all build output live in the terminal |
| `--parallel-workers <N>` | Build at most N packages at the same time |
| `--executor sequential` | Build one package at a time |
| `--cmake-args -DCMAKE_BUILD_TYPE=Release` | Pass options to CMake (C++ packages) |

```bash
$ colcon build --packages-select my_cpp_pkg --event-handlers console_direct+
```

> 💡 Large C++ workspaces can use all your RAM and freeze a laptop or a Raspberry Pi. Reduce parallelism with `--parallel-workers 1` and, for even fewer compiler processes, `MAKEFLAGS="-j2" colcon build`.

The full output of every build is saved in `log/latest_build/`, one folder per package.

---

## Running Tests

```bash
$ colcon test --packages-select my_first_pkg
Starting >>> my_first_pkg
Finished <<< my_first_pkg [2.10s]

Summary: 1 package finished [2.40s]

$ colcon test-result --verbose
Summary: 3 tests, 0 errors, 0 failures, 0 skipped
```

`colcon test` runs the tests but does not print failures in detail. Always follow it with `colcon test-result --verbose` to see what failed.

The generated Python package already includes tests that check code style (flake8), docstring style (pep257), and license headers. They are a good habit, and they help you match the style guide described in [CONTRIBUTING.md](../CONTRIBUTING.md).

---

## Inspecting a Workspace

```bash
$ colcon list
my_first_pkg    src/my_first_pkg    (ros.ament_python)
my_cpp_pkg      src/my_cpp_pkg      (ros.ament_cmake)

$ colcon list --names-only
my_cpp_pkg
my_first_pkg

$ colcon graph
```

`colcon list` shows which packages colcon found and their build types. If a package is missing from the list, colcon cannot see it (usually a missing or broken `package.xml`). `colcon graph` shows the dependency order.

To make colcon ignore a folder in `src`, put an empty file named `COLCON_IGNORE` in it:

```bash
$ touch src/experimental_pkg/COLCON_IGNORE
```

---

## Cleaning

colcon has no built-in clean command. Delete the generated folders:

```bash
$ cd ~/ros2_ws
$ rm -rf build install log                                  # everything
$ rm -rf build/my_first_pkg install/my_first_pkg            # one package
```

Then build again.

---

## Common Build Errors

| Error | Likely cause and fix |
|-------|----------------------|
| `colcon: command not found` | Install `python3-colcon-common-extensions` |
| `Could not find a package configuration file provided by "xyz"` | A dependency is missing. Add it to `package.xml` and run `rosdep install --from-paths src --ignore-src -y` |
| `ModuleNotFoundError` when running a node | The dependency is not declared or not installed, or the workspace is not sourced |
| A new executable or launch file is "not found" | You added it to `setup.py`/`CMakeLists.txt` but did not rebuild, or forgot to install it |
| Strange errors after renaming or deleting a package | Old files remain in `build/` and `install/`. Clean and rebuild |
| `--symlink-install` related errors | You switched between symlink and copy builds. Clean and rebuild |

When a build fails, scroll up to the **first** error. Later errors are often just consequences of the first one.

---

## Try It Yourself

1. Build your workspace with `--symlink-install`. Change the default greeting in `hello_node.py` and run the node again **without** rebuilding. Does the change appear?
2. Now add a second executable to the package (see [setup.py](setup-py.md)). Does it work without rebuilding?
3. Run `colcon test` and `colcon test-result --verbose` on `my_first_pkg`. Then add a line longer than 100 characters to `hello_node.py` and run the tests again. What fails?
4. Put a `COLCON_IGNORE` file in one of your packages and check with `colcon list` that it disappears.
5. Open `log/latest_build/my_first_pkg/` and look at the files inside.

---

⬅️ [Previous: Packages](packages.md) | [⬆️ Workspaces & Packages](README.md) | [Next: package.xml](package-xml.md) ➡️
