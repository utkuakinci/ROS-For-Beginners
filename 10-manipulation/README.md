# 10. Manipulation

An introduction to robotic arms and motion planning.

So far, every robot in this guide has driven on wheels. **Manipulation** is about robots that change the world: arms that pick, place, assemble, and sort. Moving an arm is very different from driving: the arm has several joints that must move together, its gripper must reach exact poses, and it must not hit itself or anything around it. In this module you learn the core concepts with a small arm of your own, then plan and execute motions with **MoveIt 2**, the standard motion planning framework of ROS 2, using the Franka Emika **Panda** arm that ships with MoveIt.

## Lessons

1. [Robot Arms](robot-arms.md) — The concepts behind controlling a robotic arm.
2. [MoveIt 2](moveit.md) — Motion planning for robotic arms.
3. [Manipulation Project](manipulation-project.md) — Plan and execute a simple pick-and-place task.

## What You Will Build

| Lesson | Result |
|--------|--------|
| Robot Arms | `my_arm`, a 3-joint arm model in `my_robot_description`, and a Python script for forward and inverse kinematics |
| MoveIt 2 | Planning in RViz2, and `hello_moveit`, your first C++ MoveIt program |
| Manipulation Project | `pick_place`, a C++ node that picks up a box and places it somewhere else |

## Before You Start

Install MoveIt 2 and the Panda example configuration:

```bash
$ sudo apt install ros-jazzy-moveit ros-jazzy-moveit-resources-panda-moveit-config
```

Check that the demo starts:

```bash
$ ros2 launch moveit_resources_panda_moveit_config demo.launch.py
```

RViz2 opens with the Panda arm. You will explore it in the [MoveIt 2](moveit.md) lesson.

The MoveIt programming examples are in **C++**, which is the primary and best-documented MoveIt API; review [Module 05](../05-cpp/README.md) if needed. A Python API (`moveit_py`) also exists and is mentioned where relevant.

---

⬅️ [Previous: Navigation & SLAM](../09-navigation/README.md) | [🏠 Home](../README.md) | [Next: Debugging & Troubleshooting](../11-debugging/README.md) ➡️
