# Nodes

A **node** is the basic unit of computation in ROS 2: a single program that does one job. A robot's software is a collection of nodes working together. In this lesson you will inspect real nodes, rename them, and group them with namespaces.

## Topics Covered

- What a node is
- Single responsibility per node
- Running nodes with `ros2 run`
- Inspecting nodes with `ros2 node list` and `ros2 node info`
- Node names and namespaces

---

## What Is a Node?

Each node is responsible for a single, well-defined task and communicates with other nodes through topics, services, actions, and parameters. A typical mobile robot might run nodes like these:

```text
┌──────────────┐   scan    ┌──────────────┐   path    ┌──────────────┐   cmd_vel   ┌──────────────┐
│ lidar_driver │ ────────▶ │    mapper    │ ────────▶ │   planner    │ ──────────▶ │ motor_driver │
└──────────────┘           └──────────────┘           └──────────────┘             └──────────────┘
```

Splitting a system into nodes has big advantages:

- **Modularity:** you can replace the LiDAR driver with a different one without touching the planner.
- **Fault isolation:** if one node crashes, the others keep running.
- **Reuse:** the same `planner` node can run on many different robots.
- **Language freedom:** each node can be written in Python or C++.
- **Distribution:** nodes can run on different computers and still talk to each other.

> 💡 **Rule of thumb:** if you can describe what a node does without using the word "and", it probably has the right size.

A node is usually one process, but ROS 2 can also run several nodes inside a single process (called *composition*) for efficiency. You will not need that as a beginner.

---

## Running a Node

Nodes are started with `ros2 run`:

```bash
ros2 run <package_name> <executable_name>
```

Start turtlesim:

```bash
$ ros2 run turtlesim turtlesim_node
[INFO] [1729070000.000000000] [turtlesim]: Starting turtlesim with node name /turtlesim
[INFO] [1729070000.000000000] [turtlesim]: Spawning turtle [turtle1] at x=[5.544445], y=[5.544445], theta=[0.000000]
```

Note that the **executable** is called `turtlesim_node`, but the **node** it creates is called `/turtlesim`. Executable names and node names do not have to match.

---

## Listing Nodes

In a second terminal, start the keyboard controller, then list the running nodes in a third:

```bash
$ ros2 run turtlesim turtle_teleop_key     # Terminal 2
```

```bash
$ ros2 node list                           # Terminal 3
/teleop_turtle
/turtlesim
```

`ros2 node list` shows every node that discovery has found, on your computer or anywhere on the network with the same `ROS_DOMAIN_ID`.

---

## Inspecting a Node

`ros2 node info` shows how a node is connected to the rest of the system:

```bash
$ ros2 node info /turtlesim
/turtlesim
  Subscribers:
    /parameter_events: rcl_interfaces/msg/ParameterEvent
    /turtle1/cmd_vel: geometry_msgs/msg/Twist
  Publishers:
    /parameter_events: rcl_interfaces/msg/ParameterEvent
    /rosout: rcl_interfaces/msg/Log
    /turtle1/color_sensor: turtlesim/msg/Color
    /turtle1/pose: turtlesim/msg/Pose
  Service Servers:
    /clear: std_srvs/srv/Empty
    /kill: turtlesim/srv/Kill
    /reset: std_srvs/srv/Empty
    /spawn: turtlesim/srv/Spawn
    /turtle1/set_pen: turtlesim/srv/SetPen
    /turtle1/teleport_absolute: turtlesim/srv/TeleportAbsolute
    /turtle1/teleport_relative: turtlesim/srv/TeleportRelative
    /turtlesim/describe_parameters: rcl_interfaces/srv/DescribeParameters
    ...
  Service Clients:

  Action Servers:
    /turtle1/rotate_absolute: turtlesim/action/RotateAbsolute
  Action Clients:

```

This single command tells you almost everything about the node's interface:

- It **subscribes** to `/turtle1/cmd_vel`: that's how it receives velocity commands.
- It **publishes** `/turtle1/pose`: the turtle's position, for anyone who wants it.
- It offers **services** such as `/spawn` and `/reset`.
- It offers an **action**, `/turtle1/rotate_absolute`.

Every node also gets some standard interfaces automatically: `/rosout` (logging), `/parameter_events`, and the `.../describe_parameters`-style parameter services.

Compare with the teleop node:

```bash
$ ros2 node info /teleop_turtle
/teleop_turtle
  Subscribers:
    /parameter_events: rcl_interfaces/msg/ParameterEvent
  Publishers:
    /parameter_events: rcl_interfaces/msg/ParameterEvent
    /rosout: rcl_interfaces/msg/Log
    /turtle1/cmd_vel: geometry_msgs/msg/Twist
  ...
  Action Clients:
    /turtle1/rotate_absolute: turtlesim/action/RotateAbsolute
```

The teleop node **publishes** `/turtle1/cmd_vel`, and turtlesim **subscribes** to it. That matching topic name is the entire connection between them.

---

## Renaming a Node

Node names must be unique. Try starting a second turtlesim with the same name:

```bash
$ ros2 run turtlesim turtlesim_node
```

Both windows appear, but now `ros2 node list` shows `/turtlesim` twice, together with a warning:

```text
WARNING: Be aware that there are nodes in the graph that share an exact name, which can have unintended side effects.
```

Tools can no longer tell the two nodes apart. Stop the second one with `Ctrl` + `C`.

You can rename a node at startup without changing its code using `--ros-args` and **remapping** (`-r`):

```bash
$ ros2 run turtlesim turtlesim_node --ros-args -r __node:=my_turtle
```

```bash
$ ros2 node list
/my_turtle
/teleop_turtle
/turtlesim
```

---

## Namespaces

Renaming fixes the node name, but both turtlesims still use the same topic `/turtle1/cmd_vel`, so the teleop node would drive both turtles at once. A **namespace** puts a prefix in front of a node's name *and* all its relative topic, service, and action names, giving a whole group of nodes its own space.

```bash
$ ros2 run turtlesim turtlesim_node --ros-args -r __ns:=/robot1
```

```bash
$ ros2 node list
/robot1/turtlesim
...

$ ros2 topic list
/robot1/turtle1/cmd_vel
/robot1/turtle1/color_sensor
/robot1/turtle1/pose
/turtle1/cmd_vel
...
```

Start a teleop in the same namespace to drive only that turtle:

```bash
$ ros2 run turtlesim turtle_teleop_key --ros-args -r __ns:=/robot1
```

This is exactly how multi-robot systems are built: the same nodes run once per robot, each in its own namespace (`/robot1`, `/robot2`, ...).

> 📝 **Relative vs absolute names:** a name without a leading slash (like `turtle1/cmd_vel` in turtlesim's code) is *relative* and gets the namespace prepended. A name that starts with `/` is *absolute* and is never changed by a namespace.

---

## Logging

Nodes report what they are doing through **logging**. The lines you saw when starting turtlesim are log messages:

```text
[INFO] [1729070000.000000000] [turtlesim]: Spawning turtle [turtle1] ...
  │           │                    │
  │           │                    └── node name
  │           └── timestamp
  └── severity level
```

The levels, from least to most severe, are `DEBUG`, `INFO`, `WARN`, `ERROR`, and `FATAL`. You can change a node's level at startup:

```bash
$ ros2 run turtlesim turtlesim_node --ros-args --log-level debug
```

All log messages are also published on the `/rosout` topic, so tools like `rqt_console` can collect logs from every node in one place.

---

## Try It Yourself

1. Start two turtlesims in namespaces `/robot1` and `/robot2`, each with its own teleop node. Drive each turtle independently.
2. Run `rqt_graph` and check that the two groups of nodes are not connected to each other.
3. Run `ros2 node info /robot1/turtlesim` and confirm that all its topics and services have the `/robot1` prefix.
4. Start turtlesim with `--log-level debug`. What extra messages do you see compared to the default?
5. Think of a robot vacuum cleaner. List five nodes it might have and what each one publishes.

---

⬅️ [Previous: ROS 2 Architecture](ros2-architecture.md) | [⬆️ ROS 2 Core Concepts](README.md) | [Next: Topics](topics.md) ➡️
