# setup.py

In an `ament_python` package, `setup.py` tells the build which Python modules to install, which **executables** to create, and which extra files (launch files, parameter files) to copy into `install/`. Most "executable not found" and "launch file not found" errors come from this file. In this lesson you will understand every line of it and finish `my_first_pkg` with a launch file and a parameter file.

## Topics Covered

- The generated `setup.py`, line by line
- `entry_points`: turning functions into executables
- `data_files`: installing launch and config files
- `setup.cfg`
- Completing `my_first_pkg`
- Common errors

---

## The Generated `setup.py`

This is the file `ros2 pkg create` generated for `my_first_pkg`:

```python
from setuptools import find_packages, setup

package_name = 'my_first_pkg'

setup(
    name=package_name,
    version='0.0.0',
    packages=find_packages(exclude=['test']),
    data_files=[
        ('share/ament_index/resource_index/packages',
            ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
    ],
    install_requires=['setuptools'],
    zip_safe=True,
    maintainer='utku',
    maintainer_email='utku@todo.todo',
    description='TODO: Package description',
    license='Apache-2.0',
    tests_require=['pytest'],
    entry_points={
        'console_scripts': [
            'hello_node = my_first_pkg.hello_node:main'
        ],
    },
)
```

> 📝 The exact template varies slightly between ROS 2 distributions (for example, newer ones use `extras_require` instead of `tests_require`). The parts explained below are the same.

| Field | Meaning |
|-------|---------|
| `name` | Package name. Must match `<name>` in `package.xml` |
| `version`, `maintainer`, `maintainer_email`, `description`, `license` | Metadata. Keep it consistent with `package.xml` |
| `packages=find_packages(...)` | Install every Python module (folder with `__init__.py`) except `test` |
| `data_files` | Extra, non-Python files to install (see below) |
| `install_requires`, `zip_safe` | Standard setuptools settings. Leave them as they are |
| `tests_require` | What the tests need |
| `entry_points` | The executables `ros2 run` can start (see below) |

---

## `entry_points`: Creating Executables

```python
entry_points={
    'console_scripts': [
        'hello_node = my_first_pkg.hello_node:main'
    ],
},
```

Each line has the form:

```text
'<executable name> = <python module path>:<function>'
     │                      │                  │
  what you type in     my_first_pkg/       the function
  `ros2 run`           hello_node.py       to call
```

So `ros2 run my_first_pkg hello_node` calls `main()` in `my_first_pkg/hello_node.py`.

To add a second executable, add a line, separated by a comma:

```python
'console_scripts': [
    'hello_node = my_first_pkg.hello_node:main',
    'goodbye_node = my_first_pkg.goodbye_node:main',
],
```

Then **rebuild**. New executables never appear without a rebuild, not even with `--symlink-install`.

> 💡 The executable name does not have to match the file name, but keeping them the same makes packages much easier to navigate.

---

## `data_files`: Installing Non-Python Files

`data_files` is a list of `(destination, [source files])` pairs. Destinations are relative to the package's install folder, `install/my_first_pkg/`.

The two generated entries are required by ROS 2 and must stay:

```python
('share/ament_index/resource_index/packages', ['resource/' + package_name]),   # registers the package
('share/' + package_name, ['package.xml']),                                    # installs the manifest
```

Launch files and parameter files are **not** installed automatically. You must add them here, otherwise `ros2 launch` cannot find them. The usual way is to install whole folders with `glob`:

```python
import os
from glob import glob

from setuptools import find_packages, setup

package_name = 'my_first_pkg'

setup(
    # ...
    data_files=[
        ('share/ament_index/resource_index/packages',
            ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
        (os.path.join('share', package_name, 'launch'), glob('launch/*.launch.py')),
        (os.path.join('share', package_name, 'config'), glob('config/*.yaml')),
    ],
    # ...
)
```

After building, the files are in `install/my_first_pkg/share/my_first_pkg/launch/` and `.../config/`, which is exactly where `get_package_share_directory()` and `ros2 launch` look for them.

---

## `setup.cfg`

```ini
[develop]
script_dir=$base/lib/my_first_pkg
[install]
install_scripts=$base/lib/my_first_pkg
```

This tells setuptools to install executables into `lib/<package_name>/`, the folder where `ros2 run` looks for them. You never need to change it, but if you rename the package, update the name here too. A wrong or missing `setup.cfg` causes `No executable found` even though the build succeeds.

---

## Completing `my_first_pkg`

Let's add a parameter file and a launch file to the package, so `hello_node` starts with its configuration in one command.

### 1. Parameter file

Create `config/hello.yaml`:

```bash
$ cd ~/ros2_ws/src/my_first_pkg
$ mkdir config launch
```

```yaml
# config/hello.yaml
/hello_node:
  ros__parameters:
    greeting: "Merhaba"
```

### 2. Launch file

Create `launch/hello.launch.py`:

```python
import os

from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch_ros.actions import Node


def generate_launch_description():
    config = os.path.join(
        get_package_share_directory('my_first_pkg'), 'config', 'hello.yaml')

    return LaunchDescription([
        Node(
            package='my_first_pkg',
            executable='hello_node',
            name='hello_node',
            parameters=[config],
            output='screen',
        ),
    ])
```

`get_package_share_directory('my_first_pkg')` returns the package's installed `share` folder, so the launch file finds the YAML file on any computer.

### 3. Install them

Add the two `data_files` lines and the `os` / `glob` imports shown above to `setup.py`, and make sure `launch` and `launch_ros` are declared in `package.xml` (see [package.xml](package-xml.md#adding-dependencies)).

### 4. Build and launch

```bash
$ cd ~/ros2_ws
$ colcon build --packages-select my_first_pkg --symlink-install
```

```bash
$ source ~/ros2_ws/install/setup.bash
$ ros2 launch my_first_pkg hello.launch.py
[INFO] [launch]: All log files can be found below /home/utku/.ros/log/...
[INFO] [hello_node-1]: process started with pid [23456]
[hello_node-1] [INFO] [1729500000.123456789] [hello_node]: Merhaba #1
[hello_node-1] [INFO] [1729500001.123456789] [hello_node]: Merhaba #2
...
```

The greeting comes from the YAML file. Because you built with `--symlink-install`, you can now change `hello.yaml`, restart the launch, and see the new greeting without rebuilding.

Your package now has the structure shown in the [module overview](README.md#what-you-will-build): code, launch file, and configuration, installed and runnable. This is the template for every Python package you will write in the following modules.

---

## Common Errors

| Error | Likely cause |
|-------|--------------|
| `No executable found` | Missing or misspelled line in `entry_points`, no rebuild after adding it, or a broken `setup.cfg` |
| `file 'hello.launch.py' was not found in the share directory of package 'my_first_pkg'` | Launch folder not in `data_files`, file name doesn't match the glob (`*.launch.py`), or no rebuild |
| `ModuleNotFoundError: No module named 'my_first_pkg.xyz'` | Wrong module path in `entry_points`, or the inner folder lacks `__init__.py` |
| `AttributeError: module ... has no attribute 'main'` | The function named after `:` does not exist in that file |
| `SyntaxError` in `setup.py` | Usually a missing comma between list entries |

---

## Try It Yourself

1. Create `my_first_pkg/goodbye_node.py` with a `main()` that logs a goodbye message, register it in `entry_points`, rebuild, and run it.
2. Add a second node to `hello.launch.py` that runs `goodbye_node`, and launch both together.
3. Add a parameter `period` (seconds between greetings) to `hello_node`, set it in `hello.yaml`, and use it for the timer.
4. Remove the `launch` line from `data_files`, clean the package's `build/` and `install/` folders, rebuild, and try to launch. Read the error, then fix it.
5. Look at the `install/my_first_pkg/` folder and find where your executables, launch file, and YAML file ended up.

---

⬅️ [Previous: package.xml](package-xml.md) | [⬆️ Workspaces & Packages](README.md)
