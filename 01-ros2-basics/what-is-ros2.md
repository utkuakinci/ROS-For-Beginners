# What is ROS 2?

Before writing any code, it helps to understand what ROS 2 actually is, what problem it solves, and how its releases are organized. This lesson also gets your first ROS 2 programs running so you can confirm your installation works.

## Topics Covered

- The problem ROS solves
- What ROS 2 is (and what it is not)
- ROS 1 vs ROS 2
- Distributions and release cycle
- The ROS 2 ecosystem at a glance
- Running your first ROS 2 programs

---

## The Problem ROS Solves

Even a simple mobile robot has many parts that must work together:

```text
  LiDAR ──┐                                ┌──▶ Left wheel motor
  Camera ─┼──▶  Perception ──▶ Planning ──▶ Control ──┤
  IMU ────┘                                └──▶ Right wheel motor
```

Each part needs to exchange data with the others, often at high rates, sometimes across several computers. Without a common framework, every team ends up writing the same plumbing over and over: how to send sensor data between programs, how to start everything together, how to configure it, how to visualize what the robot "sees", and how to record data for debugging.

**ROS (Robot Operating System)** provides that plumbing, along with a huge library of ready-made tools and algorithms, so you can focus on what makes your robot unique.

---

## What ROS 2 Is (and What It Is Not)

Despite its name, ROS is **not an operating system** like Linux or Windows. It is a **middleware and set of tools** that runs *on top of* an operating system (primarily Ubuntu Linux).

ROS 2 gives you:

| Part | What it provides |
|------|------------------|
| **Communication** | A standard way for programs (*nodes*) to exchange data: topics, services, and actions |
| **Tools** | Command-line tools, visualization (RViz2), data recording (rosbag2), simulation integration (Gazebo) |
| **Libraries** | Navigation (Nav2), manipulation (MoveIt 2), transforms (TF2), drivers for common sensors |
| **Build system** | A standard way to organize, build, and share code as *packages* |
| **Community** | Thousands of open-source packages, documentation, and an active community |

---

## ROS 1 vs ROS 2

ROS 1 was released in 2010 and became the standard in robotics research. Its design had limits that mattered more and more as robots moved into products: a single central master process, no built-in security, weak real-time support, and Linux-only operation. ROS 2 is a redesign that addresses these.

| | ROS 1 | ROS 2 |
|---|---|---|
| **Discovery** | Central `roscore` master required | Fully distributed, no master |
| **Communication** | Custom protocol (TCPROS) | Industry-standard DDS |
| **Quality of Service** | Not configurable | Configurable reliability, history, durability |
| **Real-time** | Not designed for it | Designed with real-time use in mind |
| **Security** | None built-in | SROS2 (authentication, encryption) |
| **Platforms** | Linux | Linux, Windows, macOS |
| **Python** | Python 2 (later 3) | Python 3 |
| **Status** | Final release (Noetic) reached end of life in May 2025 | Actively developed |

> 📝 Many online tutorials still show ROS 1 commands such as `roscore`, `rosrun`, or `catkin_make`. They do not work in ROS 2. If a tutorial uses them, look for a ROS 2 version instead.

---

## Distributions and Release Cycle

A **distribution** (distro) is a versioned set of ROS 2 packages that are tested to work together. Distros are named alphabetically after turtles, and a new one is released every year in May.

- **Even-year releases** are **LTS** (Long-Term Support): supported for 5 years and recommended for learning and products.
- **Odd-year releases** are supported for about 1.5 years.
- **Rolling** is a continuously updated development version. Avoid it while learning.

The distros this guide supports:

| Distro | Released | End of life | Ubuntu |
|--------|----------|-------------|--------|
| **Jazzy Jalisco** (LTS) | May 2024 | May 2029 | 24.04 |
| **Humble Hawksbill** (LTS) | May 2022 | May 2027 | 22.04 |

Each distro targets a specific Ubuntu version. You cannot easily install Jazzy on Ubuntu 22.04 or Humble on 24.04 using the standard packages, so pick the distro that matches your OS. See the [official distribution list](https://docs.ros.org/en/rolling/Releases.html) for newer releases.

> 📝 Examples in this guide use **Jazzy**. If you use Humble, replace `jazzy` with `humble` in commands.

---

## The ROS 2 Ecosystem at a Glance

```text
┌──────────────────────────────────────────────────────────────┐
│  Your application: your nodes, launch files, configuration   │
├──────────────────────────────────────────────────────────────┤
│  Ecosystem: Nav2 · MoveIt 2 · TF2 · ros2_control · drivers   │
├──────────────────────────────────────────────────────────────┤
│  Tools: ros2 CLI · RViz2 · rqt · rosbag2 · Gazebo · colcon   │
├──────────────────────────────────────────────────────────────┤
│  Core: client libraries (rclpy, rclcpp) · communication      │
├──────────────────────────────────────────────────────────────┤
│  Operating system: Ubuntu Linux                              │
└──────────────────────────────────────────────────────────────┘
```

This guide moves from the bottom up: core concepts and tools first, then writing your own code, then the higher-level libraries.

---

## Running Your First ROS 2 Programs

This guide assumes ROS 2 is already installed. If it is not, follow the official installation guide for your distro:

- [ROS 2 Jazzy installation (Ubuntu 24.04)](https://docs.ros.org/en/jazzy/Installation/Ubuntu-Install-Debs.html)
- [ROS 2 Humble installation (Ubuntu 22.04)](https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debs.html)

Install the `ros-<distro>-desktop` package; it includes RViz2, demos, and turtlesim, which the following lessons use.

### Check your installation

Open a terminal and source ROS 2 (see [Linux Basics](../00-prerequisites/linux-basics.md#environment-variables-and-bashrc) to make this automatic):

```bash
$ source /opt/ros/jazzy/setup.bash
$ printenv ROS_DISTRO
jazzy
```

### Talker and listener

ROS 2 ships with a classic demo: one program sends messages and another receives them.

**Terminal 1:**

```bash
$ ros2 run demo_nodes_cpp talker
[INFO] [1729070000.123456789] [talker]: Publishing: 'Hello World: 1'
[INFO] [1729070001.123456789] [talker]: Publishing: 'Hello World: 2'
...
```

**Terminal 2:**

```bash
$ ros2 run demo_nodes_py listener
[INFO] [1729070001.124567890] [listener]: I heard: [Hello World: 2]
[INFO] [1729070002.124567890] [listener]: I heard: [Hello World: 3]
...
```

Notice two things:

1. The talker is written in **C++** (`demo_nodes_cpp`) and the listener in **Python** (`demo_nodes_py`), yet they communicate without any extra work. ROS 2 handles that for you.
2. You never told them how to find each other. ROS 2 **discovered** each other automatically.

Stop both programs with **`Ctrl` + `C`**.

### Turtlesim

Turtlesim is a tiny 2D simulator used throughout this module to explore ROS 2 concepts.

**Terminal 1:**

```bash
$ ros2 run turtlesim turtlesim_node
```

A window with a turtle appears.

**Terminal 2:**

```bash
$ ros2 run turtlesim turtle_teleop_key
Reading from keyboard
---------------------------
Use arrow keys to move the turtle.
...
```

With the second terminal focused, use the arrow keys to drive the turtle. Keep turtlesim in mind: in the next lessons you will find out exactly *how* the keyboard program moves the turtle.

---

## Try It Yourself

1. Run the talker and **two** listeners in three terminals. Do both listeners receive every message?
2. Stop the talker and start it again. What do the listeners do?
3. Run `ros2 run demo_nodes_py talker` together with `ros2 run demo_nodes_cpp listener` (languages swapped). Does it still work?
4. Look up which ROS 2 distro is the latest LTS on the [official list](https://docs.ros.org/en/rolling/Releases.html) and note its end-of-life date.

---

[⬆️ ROS 2 Core Concepts](README.md) | [Next: ROS 2 Architecture](ros2-architecture.md) ➡️
