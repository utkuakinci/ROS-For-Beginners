# Packages

A **package** is the unit of organization in ROS 2. Every node, launch file, message definition, and configuration file belongs to a package. Packages are what you build, share, and install. In this lesson you create your first package, build it, and run your own node.

## Topics Covered

- What belongs in a package
- `ament_python` vs `ament_cmake`
- Creating a package with `ros2 pkg create`
- The layout of a Python package
- Building and running your first node
- A first look at `ament_cmake` packages and `CMakeLists.txt`
- Organizing a project into packages

---

## What Is a Package?

A package is a folder with a `package.xml` file at its root. It groups everything that belongs together:

- **Executables** (nodes) written in Python or C++
- **Launch files** to start them
- **Configuration files** (YAML parameter files, RViz configurations)
- **Interface definitions** (custom messages, services, actions)
- **Robot descriptions** (URDF, meshes)
- **Tests**

Packages you install with `apt` (like `turtlesim`) and packages you build yourself work exactly the same way.

---

## Build Types: `ament_python` vs `ament_cmake`

Every package has a **build type** that tells colcon how to build it:

| | `ament_python` | `ament_cmake` |
|---|---|---|
| **Language** | Python only | C++ (Python can be added) |
| **Build configuration** | `setup.py` + `setup.cfg` | `CMakeLists.txt` |
| **Use for** | Python nodes and tools | C++ nodes, custom interfaces, libraries |

> 📝 Custom message, service, and action definitions **must** be in an `ament_cmake` package, even if all your nodes are in Python. That is why projects commonly keep interfaces in a separate package (see [Custom Interfaces](../04-python/custom-messages.md)).

This guide focuses on `ament_python` for now. C++ packages are covered in [Module 05](../05-cpp/README.md).

---

## Creating a Package

Packages are created in the workspace's `src` folder with `ros2 pkg create`:

```bash
$ cd ~/ros2_ws/src
$ ros2 pkg create my_first_pkg \
    --build-type ament_python \
    --dependencies rclpy \
    --node-name hello_node \
    --license Apache-2.0
going to create a new package
package name: my_first_pkg
destination directory: /home/utku/ros2_ws/src
package format: 3
version: 0.0.0
description: TODO: Package description
maintainer: ['utku <utku@todo.todo>']
licenses: ['Apache-2.0']
build type: ament_python
dependencies: ['rclpy']
node_name: hello_node
creating folder ./my_first_pkg
creating ./my_first_pkg/package.xml
...
```

| Option | Meaning |
|--------|---------|
| `--build-type` | `ament_python` or `ament_cmake` (the default is `ament_cmake`) |
| `--dependencies` | Packages this package depends on; added to `package.xml` for you |
| `--node-name` | Also create a "hello world" executable with this name |
| `--license` | The license to declare (`Apache-2.0`, `MIT`, `BSD-3-Clause`, ...) |
| `--description`, `--maintainer-name`, `--maintainer-email` | Fill in the metadata now instead of editing it later |

### Naming rules

Package names must be **lowercase**, start with a letter, and use only letters, digits, and underscores: `my_first_pkg` ✅, `MyFirstPkg` ❌, `my-first-pkg` ❌.

---

## Layout of a Python Package

```text
my_first_pkg/
├── package.xml              ← manifest: name, version, dependencies
├── setup.py                 ← build configuration: executables, installed files
├── setup.cfg                ← tells setuptools where to install executables
├── resource/
│   └── my_first_pkg         ← empty marker file so ROS 2 can find the package
├── my_first_pkg/            ← the Python module: your code goes here
│   ├── __init__.py
│   └── hello_node.py
└── test/                    ← style and license checks run by `colcon test`
    ├── test_copyright.py
    ├── test_flake8.py
    └── test_pep257.py
```

Note the two folders named `my_first_pkg`: the outer one is the **package**, the inner one is the **Python module** with your source files.

The generated node is a plain Python script:

```python
# my_first_pkg/hello_node.py
def main():
    print('Hi from my_first_pkg.')


if __name__ == '__main__':
    main()
```

---

## Building and Running

Build from the workspace root, then source the workspace **in a new terminal** and run the node:

```bash
$ cd ~/ros2_ws
$ colcon build --packages-select my_first_pkg
Starting >>> my_first_pkg
Finished <<< my_first_pkg [1.05s]

Summary: 1 package finished [1.35s]
```

```bash
$ source ~/ros2_ws/install/setup.bash
$ ros2 run my_first_pkg hello_node
Hi from my_first_pkg.
```

You have built and run your first package. 🎉

---

## Turning It into a Real ROS 2 Node

The generated script doesn't use ROS 2 yet. Replace the contents of `my_first_pkg/hello_node.py` with the node you saw in [Python Basics](../00-prerequisites/python-basics.md#putting-it-together-a-preview-of-a-ros-2-node), extended with a parameter:

```python
import rclpy
from rclpy.node import Node


class HelloNode(Node):
    def __init__(self):
        super().__init__('hello_node')
        self.declare_parameter('greeting', 'Hello')
        self.count = 0
        self.create_timer(1.0, self.tick)

    def tick(self):
        self.count += 1
        greeting = self.get_parameter('greeting').value
        self.get_logger().info(f'{greeting} #{self.count}')


def main():
    rclpy.init()
    node = HelloNode()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.try_shutdown()


if __name__ == '__main__':
    main()
```

Rebuild and run:

```bash
$ colcon build --packages-select my_first_pkg       # build terminal
```

```bash
$ ros2 run my_first_pkg hello_node                  # run terminal
[INFO] [1729500000.123456789] [hello_node]: Hello #1
[INFO] [1729500001.123456789] [hello_node]: Hello #2
...
```

Everything you learned in [Module 02](../02-ros2-cli/README.md) works on your own node:

```bash
$ ros2 node list
/hello_node

$ ros2 param set /hello_node greeting "Merhaba"
Set parameter successful
```

The output changes to `Merhaba #5`, `Merhaba #6`, ... How the code works is explained step by step in [Module 04](../04-python/README.md).

> 💡 Tired of rebuilding after every change to a Python file? Build with `colcon build --symlink-install` once, and edits to existing Python files take effect immediately. See [colcon](colcon.md#--symlink-install).

---

## A First Look at `ament_cmake` Packages

A C++ package is created the same way, with a different build type:

```bash
$ ros2 pkg create my_cpp_pkg --build-type ament_cmake --dependencies rclcpp --node-name hello_node
```

```text
my_cpp_pkg/
├── CMakeLists.txt           ← build configuration
├── package.xml              ← manifest
├── include/my_cpp_pkg/      ← public C++ headers
└── src/
    └── hello_node.cpp       ← source files
```

Instead of `setup.py`, the build is described in `CMakeLists.txt`. Stripped down to its essentials, it looks like this:

```cmake
cmake_minimum_required(VERSION 3.8)
project(my_cpp_pkg)

# Find dependencies (must match package.xml)
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)

# Build an executable from a source file and link it against its dependencies
add_executable(hello_node src/hello_node.cpp)
ament_target_dependencies(hello_node rclcpp)

# Install the executable where `ros2 run` can find it
install(TARGETS hello_node
  DESTINATION lib/${PROJECT_NAME})

# Install launch and config folders (if the package has them)
# install(DIRECTORY launch config
#   DESTINATION share/${PROJECT_NAME})

ament_package()
```

The four steps (find dependencies, build, install, `ament_package()`) appear in every `ament_cmake` package. You will write `CMakeLists.txt` files in detail in [Module 05](../05-cpp/README.md).

---

## Organizing a Project into Packages

Small experiments fit in one package, but real robot projects are split into several packages with clear roles. A common pattern:

```text
src/
├── my_robot_interfaces/    ← custom messages, services, actions (ament_cmake)
├── my_robot_description/   ← URDF, meshes, RViz configuration
├── my_robot_control/       ← nodes: drivers, controllers, behaviors
└── my_robot_bringup/       ← launch files and parameter files that start everything
```

Why split?

- Other projects can reuse one part (e.g. the interfaces) without pulling in everything else.
- Interfaces need `ament_cmake`, but the nodes can stay in Python.
- Launch and configuration changes don't require rebuilding code packages.

Several packages can live in the same Git repository.

---

## Try It Yourself

1. Create a second Python package `my_tools` with a node `counter` and run it.
2. Change the default greeting of `hello_node` to your own and rebuild. Then override it at startup with `--ros-args -p greeting:=...` without rebuilding.
3. Try to create a package named `My-Package`. What does `ros2 pkg create` report?
4. Run `ros2 pkg executables my_first_pkg` and `ros2 pkg prefix my_first_pkg`. Where is your package installed?
5. Create the `my_cpp_pkg` package shown above, build it, and run its `hello_node`.

---

⬅️ [Previous: Workspaces](workspace.md) | [⬆️ Workspaces & Packages](README.md) | [Next: colcon](colcon.md) ➡️
