# 06. Transform Library (TF2)

Tracking the position and orientation of every part of a robot over time.

A robot is full of coordinate frames: the map, the robot's base, each wheel, the camera, the laser scanner. A LiDAR reports an obstacle 2 m in front of *the sensor*, but the planner needs to know where it is on *the map*. **TF2** keeps track of how all these frames relate to each other over time and converts data between them. Almost every ROS 2 package beyond the basics depends on it, from RViz2 to Nav2 and MoveIt 2.

## Lessons

1. [Coordinate Frames](coordinate-frames.md) — The frames a robot uses and how they relate to each other.
2. [TF2 Broadcasters](tf2-broadcaster.md) — Publish transforms between frames.
3. [TF2 Listeners](tf2-listener.md) — Look up transforms and convert data between frames.
4. [TF2 Tools](tf2-tools.md) — Inspect and debug the transform tree.

## What You Will Build

Continuing in `my_first_pkg` from [Module 04](../04-python/README.md):

| Lesson | Executable / file | What it does |
|--------|-------------------|--------------|
| Broadcasters | `static_frames` | Publishes fixed sensor frames on a turtle |
| Broadcasters | `turtle_tf_broadcaster` | Publishes each turtle's pose as a transform |
| Listeners | `turtle_follower` | Makes `turtle2` follow `turtle1` using TF |
| Listeners | `launch/tf_follow.launch.py` | Starts the whole follow demo |

## Before You Start

Install the TF2 tools and demo packages:

```bash
$ sudo apt install ros-jazzy-tf2-ros ros-jazzy-tf2-tools ros-jazzy-tf2-geometry-msgs ros-jazzy-turtle-tf2-py
```

The examples in this module are written in Python; each lesson ends with the C++ equivalents of the key calls.

---

⬅️ [Previous: ROS 2 Client Libraries: C++ (rclcpp)](../05-cpp/README.md) | [🏠 Home](../README.md) | [Next: Robot Modeling](../07-robot-modeling/README.md) ➡️
