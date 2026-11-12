# 08. Simulation

Testing robots in a physics simulator before running on real hardware.

A simulator lets you develop and test robot software without a robot: no hardware to break, no batteries to charge, and you can reset the world in a second. In this module you bring `my_robot` from [Module 07](../07-robot-modeling/README.md) to life in **Gazebo**: it drives with physics, its wheels turn, and it gets a simulated LiDAR, camera, and IMU that publish data to ROS 2 exactly like real sensors would.

## Lessons

1. [Gazebo](gazebo.md) — Getting started with the Gazebo simulator.
2. [Spawning Robots](spawning-robots.md) — Load your own robot description into a simulated world.
3. [Sensors](sensors.md) — Attach simulated sensors to a robot and read their data in ROS 2.
4. [Simulation Project](simulation-project.md) — Put the module together in a complete simulated robot.

## Which Gazebo?

"Gazebo" refers to several generations of the simulator, and each ROS 2 distribution pairs with a specific one:

| ROS 2 | Recommended Gazebo | Command | Notes |
|-------|--------------------|---------|-------|
| **Jazzy** | **Gazebo Harmonic** | `gz sim` | Used in this guide |
| Humble | Gazebo Fortress | `ign gazebo` | Same concepts, older names (see below) |
| — | Gazebo Classic (`gazebo`) | `gazebo` | **End of life** since January 2025. Avoid it, even though many older tutorials use it |

The examples in this module are written for **Jazzy with Gazebo Harmonic**. If you use Humble with Fortress, the concepts and structure are identical, but some names differ:

| Harmonic (Jazzy) | Fortress (Humble) |
|------------------|-------------------|
| `gz sim`, `gz topic` | `ign gazebo`, `ign topic` |
| `gz-sim-diff-drive-system` / `gz::sim::systems::DiffDrive` | `ignition-gazebo-diff-drive-system` / `ignition::gazebo::systems::DiffDrive` |
| `gz.msgs.Twist` (bridge types) | `ignition.msgs.Twist` |
| `<gz_frame_id>` (sensor tag) | `<ignition_frame_id>` |
| `GZ_SIM_RESOURCE_PATH` | `IGN_GAZEBO_RESOURCE_PATH` |

## What You Will Build

A new package, `my_robot_gazebo`, next to the description package:

```text
~/ros2_ws/src/
├── my_robot_description/
│   └── urdf/
│       ├── my_robot.urdf.xacro      ← includes the file below
│       └── my_robot.gazebo.xacro    ← new: Gazebo plugins and sensors
└── my_robot_gazebo/
    ├── worlds/my_world.sdf          ← a world with obstacles
    ├── config/bridge.yaml           ← which topics connect ROS 2 and Gazebo
    └── launch/sim.launch.py         ← starts everything with one command
```

## Before You Start

Install Gazebo and the ROS 2 integration packages:

```bash
$ sudo apt install ros-jazzy-ros-gz ros-jazzy-teleop-twist-keyboard
```

`ros-jazzy-ros-gz` installs the matching Gazebo Harmonic version together with the bridge and launch tools, so you don't need to install Gazebo separately.

> ⚠️ Gazebo needs a working 3D graphics driver. In a virtual machine or WSL2, rendering can be slow or fail; see the troubleshooting section of the [Gazebo](gazebo.md#troubleshooting) lesson.

---

⬅️ [Previous: Robot Modeling](../07-robot-modeling/README.md) | [🏠 Home](../README.md) | [Next: Navigation & SLAM](../09-navigation/README.md) ➡️
