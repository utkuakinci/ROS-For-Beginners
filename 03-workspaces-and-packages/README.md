# 03. Workspaces & Packages

How ROS 2 code is organized, built, and installed.

So far you have only run software that came with ROS 2. In this module you create your own **workspace**, add your first **package** to it, build it with **colcon**, and run your own node. The lessons build on each other: by the end you will have a package `my_first_pkg` with a node, a launch file, and a parameter file, ready for the coding modules that follow.

## Lessons

1. [Workspaces](workspace.md) — A workspace is the directory where you develop and build ROS 2 packages.
2. [Packages](packages.md) — Packages are the unit of organization and distribution for ROS 2 code.
3. [colcon](colcon.md) — `colcon` is the build tool for ROS 2 workspaces.
4. [package.xml](package-xml.md) — The manifest that describes a package and its dependencies.
5. [setup.py](setup-py.md) — The build configuration for Python packages.

## What You Will Build

```text
~/ros2_ws/
└── src/
    └── my_first_pkg/
        ├── package.xml
        ├── setup.py
        ├── setup.cfg
        ├── resource/my_first_pkg
        ├── my_first_pkg/
        │   ├── __init__.py
        │   └── hello_node.py
        ├── launch/
        │   └── hello.launch.py
        ├── config/
        │   └── hello.yaml
        └── test/
```

---

⬅️ [Previous: CLI Tools](../02-ros2-cli/README.md) | [🏠 Home](../README.md) | [Next: ROS 2 Client Libraries: Python (rclpy)](../04-python/README.md) ➡️
