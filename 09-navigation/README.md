# 09. Navigation & SLAM

Making a mobile robot know where it is and move autonomously.

Up to now, `my_robot` either followed your keyboard or reacted blindly to obstacles. In this module it learns to navigate like a real autonomous robot: it builds a **map** of its environment, **localizes** itself on that map, and **plans and follows paths** to goals you give it, avoiding obstacles on the way. You will use **slam_toolbox** for mapping and **Nav2**, the ROS 2 navigation stack, for everything else.

## Lessons

1. [Mapping (SLAM)](mapping.md) — Building a map of an unknown environment.
2. [Localization](localization.md) — Estimating the robot's pose on a known map.
3. [Nav2](nav2.md) — The ROS 2 navigation stack and how to configure it.
4. [Autonomous Navigation](autonomous-navigation.md) — Send goals and waypoints to a robot.

The order matters: you first need a map (lesson 1) before the robot can locate itself on it (lesson 2), and both before it can plan paths (lessons 3 and 4).

## The Big Picture

```text
            ┌──────────────────────────── Nav2 ─────────────────────────────┐
  goal ───▶ │  BT Navigator ──▶ Planner (path) ──▶ Controller (cmd_vel)    │ ──▶ /cmd_vel ──▶ robot
            │        ▲                ▲                    ▲                │
            │        │          global costmap        local costmap         │
            └────────┼────────────────┼────────────────────┼────────────────┘
                     │                │                    │
      map ◀── map_server         /map + /scan            /scan
      pose ◀── AMCL  ◀── /scan, /odom, TF
```

| Question | Answered by | Lesson |
|----------|-------------|--------|
| What does the world look like? | A map from **SLAM** (`slam_toolbox`) | [Mapping](mapping.md) |
| Where am I on the map? | **AMCL** localization | [Localization](localization.md) |
| How do I get to the goal without hitting anything? | Nav2's **planner**, **controller**, and **costmaps** | [Nav2](nav2.md) |
| How do I tell the robot where to go? | RViz2, the CLI, or Python with the **Simple Commander** API | [Autonomous Navigation](autonomous-navigation.md) |

## Before You Start

You need the simulation from [Module 08](../08-simulation/README.md): `my_robot` with its LiDAR in `my_world.sdf`, started with:

```bash
$ ros2 launch my_robot_gazebo sim.launch.py use_rviz:=false
```

Check that the three things navigation depends on are present:

```bash
$ ros2 topic hz /scan                          # laser data
$ ros2 topic hz /odom                          # odometry
$ ros2 run tf2_ros tf2_echo odom base_footprint   # odom → base_footprint transform
```

Install the navigation packages:

```bash
$ sudo apt install ros-jazzy-navigation2 ros-jazzy-nav2-bringup ros-jazzy-slam-toolbox \
    ros-jazzy-nav2-simple-commander
```

Create a package for your maps, parameters, and launch files:

```bash
$ cd ~/ros2_ws/src
$ ros2 pkg create my_robot_navigation --build-type ament_cmake --license Apache-2.0
$ cd my_robot_navigation
$ rm -rf include src
$ mkdir maps config launch
```

```cmake
cmake_minimum_required(VERSION 3.8)
project(my_robot_navigation)

find_package(ament_cmake REQUIRED)

install(DIRECTORY maps config launch
  DESTINATION share/${PROJECT_NAME}
)

ament_package()
```

> 💡 Navigation runs many nodes and is demanding on the CPU together with Gazebo. Close other programs, and if the simulation's Real Time Factor drops far below 100 %, lower the sensor rates from [Module 08](../08-simulation/sensors.md).

---

⬅️ [Previous: Simulation](../08-simulation/README.md) | [🏠 Home](../README.md) | [Next: Manipulation](../10-manipulation/README.md) ➡️
