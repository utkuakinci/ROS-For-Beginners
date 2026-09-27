# 🤖 ROS-For-Beginners
Welcome to **ROS-For-Beginners**! This repository is designed as a structured, hands-on, and beginner-friendly roadmap to learning **ROS 2 (Robot Operating System)** from scratch.

Whether you are a student, hobbyist, or software engineer transitioning into robotics, this guide will take you step-by-step from fundamental prerequisites to advanced autonomous navigation and manipulation.

---

## 🗺️ Learning Roadmap

The content is organized into progressive modules. We recommend following them in order:

### 00. Prerequisites
- [Linux Basics](00-prerequisites/linux-basics.md)
- [Git Basics](00-prerequisites/git-basics.md)
- [Python Basics](00-prerequisites/python-basics.md)

### 01. ROS 2 Core Concepts
- [What is ROS 2?](01-ros2-basics/what-is-ros2.md)
- [ROS 2 Architecture](01-ros2-basics/ros2-architecture.md)
- [Nodes](01-ros2-basics/nodes.md) | [Topics](01-ros2-basics/topics.md) | [Services](01-ros2-basics/services.md) | [Actions](01-ros2-basics/actions.md)
- [Parameters](01-ros2-basics/parameters.md) & [Launch Files](01-ros2-basics/launch-files.md)

### 02. CLI Tools
Master the essential command-line interface tools to introspect running ROS 2 systems:
- `ros2 node`, `ros2 topic`, `ros2 service`, `ros2 action`, `ros2 param`, `ros2 run`, `ros2 launch`

### 03. Workspaces & Packages
Learn how ROS 2 manages software builds:
- Workspaces, Packages, `colcon`, `package.xml`, and `setup.py`

### 04. ROS 2 Client Libraries: Python (rclpy)
- Hands-on implementation of Publishers, Subscribers, Services, Actions, and Custom Interfaces.

### 05. ROS 2 Client Libraries: C++ (rclcpp)
- Modern C++ implementations of core ROS 2 communication mechanisms.

### 06. Transform Library (TF2)
- Understanding coordinate frames, static/dynamic broadcasters, listeners, and debugging tools.

### 07. Robot Modeling
- Describing robots with `URDF` and `Xacro`, displaying state with `robot_state_publisher`, and visualizing in `RViz2`.

### 08. Simulation
- Physics simulation using Gazebo, spawning custom robots, attaching sensors (LiDAR, Camera, IMU), and running simulation projects.

### 09. Navigation & SLAM
- Robot localization, map building (SLAM), Nav2 stack configuration, and autonomous navigation.

### 10. Manipulation
- Introduction to robotic arms, kinematics, and trajectory planning with MoveIt 2.

### 11. Debugging & Troubleshooting
- Resolving common ROS 2 errors, using `ros2 doctor`, logging/playing back data with `rosbag2`.

---

## 🚀 Hands-on Projects

Apply what you've learned through incremental projects:

1. **`01-turtlesim`**: Getting started with basic velocity commands and telemetry.
2. **`02-obstacle-avoidance`**: Sensor data integration for basic collision prevention.
3. **`03-mapping`**: Building 2D grid maps using simulated LiDAR.
4. **`04-autonomous-navigation`**: Waypoint navigation in dynamic environments.
5. **`05-final-project`**: End-to-end integration combining simulation, navigation, and manipulation.

---

## 🛠️ Recommended Setup

- **OS:** Ubuntu 22.04 LTS (Jammy Jellyfish) or Ubuntu 24.04 LTS
- **ROS 2 Version:** ROS 2 Humble Hawksbill (LTS) or ROS 2 Jazzy Jalisco (LTS)
- **Language Requirements:** Python 3.10+ / C++17

---

## 🤝 Contributing

Contributions are what make the open-source community an amazing place to learn, inspire, and create. Any contributions you make are **greatly appreciated**!

Please check [CONTRIBUTING.md](CONTRIBUTING.md) before submitting a Pull Request.

---

## 📁 Repository Structure

ROS-For-Beginners/
│
├── README.md
├── CONTRIBUTING.md
├── LICENSE
│
├── 00-prerequisites/
│   ├── README.md
│   ├── linux-basics.md
│   ├── git-basics.md
│   └── python-basics.md
│
├── 01-ros2-basics/
│   ├── README.md
│   ├── what-is-ros2.md
│   ├── ros2-architecture.md
│   ├── nodes.md
│   ├── topics.md
│   ├── services.md
│   ├── actions.md
│   ├── parameters.md
│   └── launch-files.md
│
├── 02-ros2-cli/
│   ├── README.md
│   ├── ros2-node.md
│   ├── ros2-topic.md
│   ├── ros2-service.md
│   ├── ros2-action.md
│   ├── ros2-param.md
│   ├── ros2-run.md
│   └── ros2-launch.md
│
├── 03-workspaces-and-packages/
│   ├── README.md
│   ├── workspace.md
│   ├── packages.md
│   ├── colcon.md
│   ├── package-xml.md
│   └── setup-py.md
│
├── 04-python/
│   ├── README.md
│   ├── publisher.md
│   ├── subscriber.md
│   ├── services.md
│   ├── actions.md
│   └── custom-messages.md
│
├── 05-cpp/
│   ├── README.md
│   ├── publisher.md
│   ├── subscriber.md
│   ├── services.md
│   └── actions.md
│
├── 06-tf2/
│   ├── README.md
│   ├── coordinate-frames.md
│   ├── tf2-broadcaster.md
│   ├── tf2-listener.md
│   └── tf2-tools.md
│
├── 07-robot-modeling/
│   ├── README.md
│   ├── urdf.md
│   ├── xacro.md
│   ├── robot-state-publisher.md
│   └── rviz.md
│
├── 08-simulation/
│   ├── README.md
│   ├── gazebo.md
│   ├── spawning-robots.md
│   ├── sensors.md
│   └── simulation-project.md
│
├── 09-navigation/
│   ├── README.md
│   ├── localization.md
│   ├── mapping.md
│   ├── nav2.md
│   └── autonomous-navigation.md
│
├── 10-manipulation/
│   ├── README.md
│   ├── robot-arms.md
│   ├── moveit.md
│   └── manipulation-project.md
│
├── 11-debugging/
│   ├── README.md
│   ├── common-errors.md
│   ├── ros2-doctor.md
│   ├── rosbag.md
│   └── debugging-workflow.md
│
└── projects/
    ├── 01-turtlesim/
    ├── 02-obstacle-avoidance/
    ├── 03-mapping/
    ├── 04-autonomous-navigation/
    └── 05-final-project/

---

## 📜 License

Distributed under the MIT License. See `LICENSE` for more information.
