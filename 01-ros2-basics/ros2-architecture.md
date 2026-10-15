# ROS 2 Architecture

You have seen two programs talk to each other without being told where the other one was. This lesson explains how that works: the *computation graph* that describes a running ROS 2 system, and the software layers underneath your code that move the data.

## Topics Covered

- The computation graph
- Communication patterns: topics, services, actions
- The software layers: client libraries, `rcl`, RMW, and DDS
- Discovery and `ROS_DOMAIN_ID`
- Packages and workspaces (a first look)

---

## The Computation Graph

A running ROS 2 system is described as a **graph**: a network of processes and the connections between them.

```text
                     /turtle1/cmd_vel
 ┌────────────────┐   (Twist messages)   ┌────────────────┐
 │  /teleop_turtle │ ───────────────────▶ │   /turtlesim   │
 └────────────────┘                      └────────────────┘
                                                 │
                                                 │ /turtle1/pose
                                                 ▼
                                         (anyone interested)
```

The building blocks of the graph are:

| Element | What it is | Lesson |
|---------|-----------|--------|
| **Node** | A process that performs one job (read a sensor, plan a path, drive motors) | [Nodes](nodes.md) |
| **Topic** | A named channel for a continuous stream of messages | [Topics](topics.md) |
| **Service** | A named request/response call | [Services](services.md) |
| **Action** | A named long-running task with feedback and cancellation | [Actions](actions.md) |
| **Parameter** | A configuration value belonging to a node | [Parameters](parameters.md) |
| **Message / interface** | The data type exchanged (e.g. a velocity, an image, a laser scan) | throughout |

Every element has a **name**, like a path in a filesystem: `/turtlesim`, `/turtle1/cmd_vel`, `/spawn`. Names are how nodes find each other.

---

## Communication Patterns

ROS 2 offers three ways for nodes to communicate. Choosing the right one is one of the first design decisions you make.

```text
 TOPIC (publish / subscribe)         one-way, continuous, many-to-many
 ┌───────────┐  msg  msg  msg   ┌────────────┐
 │ Publisher │ ───────────────▶ │ Subscriber │
 └───────────┘                  └────────────┘

 SERVICE (request / response)        two-way, short, one-to-one
 ┌────────┐ ──── request ────▶ ┌────────┐
 │ Client │ ◀─── response ──── │ Server │
 └────────┘                    └────────┘

 ACTION (goal / feedback / result)   two-way, long-running, cancellable
 ┌────────┐ ──── goal ───────▶ ┌────────┐
 │ Client │ ◀─── feedback ──── │ Server │
 │        │ ◀─── feedback ──── │        │
 │        │ ◀─── result ────── │        │
 └────────┘                    └────────┘
```

| Use a... | When | Example |
|----------|------|---------|
| **Topic** | Data flows continuously and the sender doesn't need a reply | Camera images, laser scans, velocity commands |
| **Service** | You need an answer, and it comes back quickly | "Reset the simulation", "What is the current map?" |
| **Action** | The task takes time and you want progress updates or to cancel | "Navigate to the kitchen", "Rotate 90 degrees" |

---

## The Software Layers

When your node publishes a message, it passes through several layers before reaching the network:

```text
┌─────────────────────────────────────────────┐
│  Your node (Python or C++)                  │
├──────────────────────┬──────────────────────┤
│  rclpy (Python)      │  rclcpp (C++)        │  ◀── client libraries
├──────────────────────┴──────────────────────┤
│  rcl  (shared C library)                    │  ◀── common logic
├─────────────────────────────────────────────┤
│  rmw  (ROS MiddleWare interface)            │  ◀── abstraction layer
├─────────────────────────────────────────────┤
│  DDS implementation                         │  ◀── actual transport
│  (eProsima Fast DDS, Eclipse Cyclone DDS…)  │
└─────────────────────────────────────────────┘
```

- **Client libraries** (`rclpy`, `rclcpp`) are what you program against. They feel natural in their language.
- **`rcl`** is a C library containing the logic both client libraries share. This is why a Python node and a C++ node behave the same way.
- **`rmw`** is an interface that hides which middleware is used underneath, so ROS 2 can swap it without changing your code.
- **DDS** (Data Distribution Service) is an industry standard for publish/subscribe communication, used in aerospace, defense, and industrial systems. It handles discovery, serialization, and delivering data over the network.

The default middleware in Humble and Jazzy is **Fast DDS**. You can switch implementations with an environment variable, for example:

```bash
$ sudo apt install ros-jazzy-rmw-cyclonedds-cpp
$ export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

You rarely need to do this as a beginner, but it is useful to know the layer exists when you read error messages that mention `rmw` or DDS.

---

## Discovery and `ROS_DOMAIN_ID`

ROS 2 has **no central master**. When a node starts, it announces itself on the network, and other nodes reply. Within a few moments, every node knows which topics, services, and actions the others offer. This is called **discovery**, and it is why the talker and listener found each other automatically.

Discovery works across all computers on the same local network. That is powerful (a laptop can talk to a robot without configuration) but can surprise you: on a shared network, you may see other people's nodes, or they may accidentally drive your robot.

The **`ROS_DOMAIN_ID`** environment variable separates ROS 2 systems on the same network. Nodes only discover nodes with the **same** domain ID. The default is `0`.

```bash
$ export ROS_DOMAIN_ID=42
```

Try it with the talker and listener:

```bash
# Terminal 1
$ export ROS_DOMAIN_ID=42
$ ros2 run demo_nodes_cpp talker

# Terminal 2 (default domain 0): hears nothing
$ ros2 run demo_nodes_py listener

# Terminal 3: same domain, hears the talker
$ export ROS_DOMAIN_ID=42
$ ros2 run demo_nodes_py listener
```

> 💡 Pick a number between **0 and 101** to be safe on all platforms. In a classroom or lab, give every student or robot a different domain ID and add the `export` line to `~/.bashrc`.

To keep ROS 2 traffic on your own computer only, Jazzy provides `ROS_AUTOMATIC_DISCOVERY_RANGE`:

```bash
$ export ROS_AUTOMATIC_DISCOVERY_RANGE=LOCALHOST
```

(On Humble, the equivalent is `export ROS_LOCALHOST_ONLY=1`.)

---

## Packages and Workspaces: A First Look

Code in ROS 2 is organized into **packages**. A package is a folder containing nodes, launch files, configuration, and a `package.xml` that describes it. When you ran `ros2 run turtlesim turtlesim_node`, `turtlesim` was the **package** and `turtlesim_node` the **executable** inside it.

Packages you write live in a **workspace**, a folder that you build with the `colcon` tool. Packages installed with `apt` live under `/opt/ros/<distro>`. You will create your own workspace and packages in [Module 03](../03-workspaces-and-packages/README.md).

```bash
$ ros2 pkg list | head -n 5            # Installed packages
$ ros2 pkg executables turtlesim       # Executables in a package
turtlesim draw_square
turtlesim mimic
turtlesim turtle_teleop_key
turtlesim turtlesim_node
```

---

## Visualizing the Graph with `rqt_graph`

`rqt_graph` draws the live computation graph. Start turtlesim and the teleop node, then run it in a third terminal:

```bash
$ ros2 run turtlesim turtlesim_node        # Terminal 1
$ ros2 run turtlesim turtle_teleop_key     # Terminal 2
$ rqt_graph                                # Terminal 3
```

You should see two nodes, `/teleop_turtle` and `/turtlesim`, connected by an arrow labeled `/turtle1/cmd_vel`. That arrow is how your key presses move the turtle. (If you see nothing, click the refresh button in the top left.)

---

## Try It Yourself

1. With turtlesim and teleop running, open `rqt_graph` and uncheck **Hide: Dead sinks** and **Hide: Leaf topics**. What new topics appear?
2. Set `ROS_DOMAIN_ID=7` in the teleop terminal only (restart teleop). Can you still drive the turtle? Why not?
3. Run `ros2 pkg executables demo_nodes_cpp` and pick an executable you haven't tried. Run it and observe what it does.
4. For each scenario, decide whether a topic, service, or action fits best: (a) a temperature sensor reporting 10 times per second, (b) asking a robot arm to move to a pose, which takes 5 seconds, (c) asking a node for the current battery level once.

---

⬅️ [Previous: What is ROS 2?](what-is-ros2.md) | [⬆️ ROS 2 Core Concepts](README.md) | [Next: Nodes](nodes.md) ➡️
