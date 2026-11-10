# 07. Robot Modeling

Describing a robot's physical structure so ROS 2 tools can visualize and simulate it.

In [Module 06](../06-tf2/README.md) you published transforms by hand. A real robot has many parts, and writing a broadcaster for each would be tedious. Instead, you describe the robot once in a **robot model**: its parts, their shapes, and how they are connected. From that model, ROS 2 publishes the whole TF tree, RViz2 draws the robot, and Gazebo simulates it.

## Lessons

1. [URDF](urdf.md) — The XML format used to describe a robot's links and joints.
2. [Xacro](xacro.md) — Macros that keep robot descriptions short and reusable.
3. [robot_state_publisher](robot-state-publisher.md) — Publish the robot's transforms from its description and joint states.
4. [RViz2](rviz.md) — Visualize the robot model, transforms, and sensor data.

## What You Will Build

A small differential-drive robot, `my_robot`, in its own description package. You will reuse it for simulation in [Module 08](../08-simulation/README.md).

```text
              ┌──────┐ laser
       ┌──────┴──────┴──────┐
       │     base_link      │◀── caster (front)
       └──┬──────────────┬──┘
     left_wheel      right_wheel
```

```text
~/ros2_ws/src/my_robot_description/
├── CMakeLists.txt
├── package.xml
├── urdf/
│   ├── my_robot.urdf            ← lesson 1
│   └── my_robot.urdf.xacro      ← lesson 2
├── launch/
│   └── display.launch.py        ← lesson 3
└── rviz/
    └── display.rviz             ← lesson 4
```

## Before You Start

Install the packages used in this module:

```bash
$ sudo apt install ros-jazzy-robot-state-publisher ros-jazzy-joint-state-publisher \
    ros-jazzy-joint-state-publisher-gui ros-jazzy-xacro ros-jazzy-rviz2 liburdfdom-tools
```

Create the description package. Description packages contain no code, only files to install, so they are usually `ament_cmake` packages:

```bash
$ cd ~/ros2_ws/src
$ ros2 pkg create my_robot_description --build-type ament_cmake --license Apache-2.0
$ cd my_robot_description
$ rm -rf include src
$ mkdir urdf launch rviz
```

Replace the build part of `CMakeLists.txt` so the folders are installed:

```cmake
cmake_minimum_required(VERSION 3.8)
project(my_robot_description)

find_package(ament_cmake REQUIRED)

install(DIRECTORY urdf launch rviz
  DESTINATION share/${PROJECT_NAME}
)

ament_package()
```

And declare the run-time dependencies in `package.xml`:

```xml
<exec_depend>robot_state_publisher</exec_depend>
<exec_depend>joint_state_publisher</exec_depend>
<exec_depend>joint_state_publisher_gui</exec_depend>
<exec_depend>xacro</exec_depend>
<exec_depend>rviz2</exec_depend>
```

---

⬅️ [Previous: Transform Library (TF2)](../06-tf2/README.md) | [🏠 Home](../README.md) | [Next: Simulation](../08-simulation/README.md) ➡️
