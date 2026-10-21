# package.xml

Every ROS 2 package has a `package.xml` file at its root. This **manifest** tells ROS 2 tools what the package is called, who maintains it, how to build it, and, most importantly, what other packages it depends on. colcon uses it to decide the build order, and rosdep uses it to install missing dependencies.

## Topics Covered

- The structure of `package.xml`
- Required metadata tags
- Dependency types
- Adding a dependency to your package
- Resolving dependencies with `rosdep`
- Common mistakes

---

## A Generated `package.xml`

This is the file `ros2 pkg create` generated for `my_first_pkg` in [Packages](packages.md):

```xml
<?xml version="1.0"?>
<?xml-model href="http://download.ros.org/schema/package_format3.xsd" schematypens="http://www.w3.org/2001/XMLSchema"?>
<package format="3">
  <name>my_first_pkg</name>
  <version>0.0.0</version>
  <description>TODO: Package description</description>
  <maintainer email="utku@todo.todo">utku</maintainer>
  <license>Apache-2.0</license>

  <depend>rclpy</depend>

  <test_depend>ament_copyright</test_depend>
  <test_depend>ament_flake8</test_depend>
  <test_depend>ament_pep257</test_depend>
  <test_depend>python3-pytest</test_depend>

  <export>
    <build_type>ament_python</build_type>
  </export>
</package>
```

It has three parts: **metadata** (who and what), **dependencies** (what it needs), and **export** (how to build it).

---

## Metadata

| Tag | Required | Meaning |
|-----|----------|---------|
| `<name>` | ✅ | Package name. Must match the folder name (and `package_name` in `setup.py`) |
| `<version>` | ✅ | Version in `MAJOR.MINOR.PATCH` form |
| `<description>` | ✅ | One or two sentences about what the package does |
| `<maintainer email="...">` | ✅ | Who to contact. Can appear more than once |
| `<license>` | ✅ | License identifier, e.g. `Apache-2.0`, `MIT`, `BSD-3-Clause` |
| `<url type="...">` | | Links: `website`, `repository`, `bugtracker` |
| `<author>` | | Original authors, if different from the maintainers |

Replace the `TODO` placeholders right away. Fill in the description and your real email:

```xml
<description>My first ROS 2 package: a configurable greeting node.</description>
<maintainer email="you@example.com">Your Name</maintainer>
```

> 📝 Tags must appear in a fixed order: `name`, `version`, `description`, `maintainer`, `license`, then `url` and `author`, then dependencies, then `export`. Keep the order of the generated file when adding tags.

---

## Dependency Types

A dependency can be needed at different stages: while **building**, while **running**, or only while **testing**. Each stage has its own tag:

| Tag | Needed when | Typical use |
|-----|-------------|-------------|
| `<depend>` | Building **and** running | The usual choice: `rclpy`, `rclcpp`, message packages |
| `<build_depend>` | Only building | Code generators, header-only libraries |
| `<build_export_depend>` | Building packages that use this one | Headers your public C++ headers include |
| `<exec_depend>` | Only running | Python libraries, launch tools, packages started from your launch files |
| `<test_depend>` | Only testing | Linters, test frameworks |
| `<buildtool_depend>` | The build tool itself | `ament_cmake` in C++ packages |

`<depend>` is shorthand for `build_depend` + `build_export_depend` + `exec_depend`. When in doubt, use `<depend>`.

For Python packages, nothing is compiled, so what really matters is that every module you `import` and every package your launch files start is declared, either with `<depend>` or `<exec_depend>`.

---

## Adding Dependencies

Suppose `hello_node` will publish its greeting on a topic using `std_msgs/msg/String`, and the package will get a launch file in the next lesson. The package then needs:

- `std_msgs` for the message type (imported in Python)
- `launch` and `launch_ros` to run the launch file

Add them below the existing `rclpy` dependency:

```xml
  <depend>rclpy</depend>
  <depend>std_msgs</depend>

  <exec_depend>launch</exec_depend>
  <exec_depend>launch_ros</exec_depend>
```

Every time you add an `import` of another ROS 2 package to your code, add a matching line to `package.xml`.

> ⚠️ A missing dependency often goes unnoticed on your own computer, because the package happens to be installed already. It then fails on a teammate's computer, in CI, or on the robot. Declaring every dependency is what makes `rosdep install` work for everyone.

### Dependencies in C++ packages

In an `ament_cmake` package, every `<depend>` usually has a matching `find_package()` in `CMakeLists.txt`:

```xml
<buildtool_depend>ament_cmake</buildtool_depend>
<depend>rclcpp</depend>
<depend>std_msgs</depend>
```

```cmake
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)
```

---

## Resolving Dependencies with `rosdep`

Dependency names in `package.xml` are **rosdep keys**. For ROS packages, the key is the package name (`std_msgs`). For system libraries, it is a name from the [rosdep database](https://github.com/ros/rosdistro/tree/master/rosdep), such as `python3-numpy` or `libopencv-dev`.

rosdep translates keys into `apt` packages and installs them:

```bash
$ cd ~/ros2_ws
$ rosdep install --from-paths src --ignore-src -y
```

Other useful rosdep commands:

```bash
# Which apt package does a key map to?
$ rosdep resolve python3-numpy
#apt
python3-numpy

$ rosdep resolve std_msgs
#apt
ros-jazzy-std-msgs

# Are all dependencies of the workspace installed?
$ rosdep check --from-paths src --ignore-src
All system dependencies have been satisfied

# List all keys the workspace depends on
$ rosdep keys --from-paths src --ignore-src
```

If rosdep reports `Cannot locate rosdep definition for [xyz]`, the key is misspelled or not in the database. Search the `base.yaml` and `python.yaml` files of the [rosdep database](https://github.com/ros/rosdistro/tree/master/rosdep) for the correct name.

---

## Export Section

```xml
<export>
  <build_type>ament_python</build_type>
</export>
```

`<build_type>` tells colcon how to build the package: `ament_python` or `ament_cmake`. Other packages and tools can also read extra information from `<export>`, but as a beginner you will only use `build_type`.

---

## Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| `<name>` differs from the folder or `setup.py` name | Build or `ros2 run` errors about the package not being found |
| Missing dependency | Works on your machine, fails elsewhere; `ModuleNotFoundError` or CMake errors |
| Tags in the wrong order | Warnings or errors from colcon and linters |
| `TODO` left in description, maintainer, or license | Fine for practice, but not acceptable in a shared package |
| Typo in a dependency name | `rosdep` cannot resolve it |

---

## Try It Yourself

1. Fill in the description, maintainer, and license of `my_first_pkg` with real values.
2. Add the `std_msgs`, `launch`, and `launch_ros` dependencies shown above and rebuild.
3. Use `rosdep resolve` to find the apt package names for `rclpy`, `python3-yaml`, and `geometry_msgs`.
4. Run `rosdep check --from-paths src --ignore-src` in your workspace. Then add a nonexistent dependency like `<depend>not_a_real_pkg</depend>` and run it again. What does it report? Remove the line afterwards.
5. Open the `package.xml` of an installed package: `cat $(ros2 pkg prefix --share turtlesim)/package.xml`. Which dependency types does it use?

---

⬅️ [Previous: colcon](colcon.md) | [⬆️ Workspaces & Packages](README.md) | [Next: setup.py](setup-py.md) ➡️
