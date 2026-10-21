# ros2 node

`ros2 node` shows which nodes are running and how each one is connected to the rest of the system. It is usually the first command you run when something does not work: *is the node I expect actually running, under the name I expect?*

> 📖 Concepts: [Nodes](../01-ros2-basics/nodes.md)

## Topics Covered

- `ros2 node list`
- `ros2 node info`
- Hidden nodes
- Debugging with `ros2 node`

---

## Command Overview

| Command | Purpose |
|---------|---------|
| `ros2 node list` | List all running nodes |
| `ros2 node info <node>` | Show a node's publishers, subscribers, services, and actions |

The examples below assume turtlesim and the teleop node are running:

```bash
$ ros2 run turtlesim turtlesim_node        # Terminal 1
$ ros2 run turtlesim turtle_teleop_key     # Terminal 2
```

---

## `ros2 node list`

```bash
$ ros2 node list
/teleop_turtle
/turtlesim
```

| Option | Effect |
|--------|--------|
| `-a`, `--all` | Also show hidden nodes (names starting with `_`) |
| `-c`, `--count-nodes` | Print only the number of nodes |

```bash
$ ros2 node list --count-nodes
2
```

Counting is useful in scripts, for example to wait until all nodes of a launch file are up.

---

## `ros2 node info`

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
    ...
  Service Clients:

  Action Servers:
    /turtle1/rotate_absolute: turtlesim/action/RotateAbsolute
  Action Clients:

```

The output has six sections:

| Section | Meaning |
|---------|---------|
| **Subscribers** | Topics the node receives |
| **Publishers** | Topics the node sends |
| **Service Servers** | Services the node provides |
| **Service Clients** | Services the node calls |
| **Action Servers** | Actions the node provides |
| **Action Clients** | Actions the node uses |

| Option | Effect |
|--------|--------|
| `--include-hidden` | Also show hidden topics, services, and actions (such as the internals of actions) |

> 💡 The node name must be the **full name**, including the namespace: `ros2 node info /robot1/turtlesim`, not `ros2 node info turtlesim`. Use `Tab` to complete it.

---

## Debugging with `ros2 node`

| Symptom | What to check |
|---------|---------------|
| The node does not appear in `ros2 node list` | Did it crash at startup? Check its terminal for errors. Is `ROS_DOMAIN_ID` the same in both terminals? |
| The node appears, but under a different name | A namespace or `__node` remapping is applied, often from a launch file |
| The same name appears twice | Two instances are running. Stop one or give them different names or namespaces |
| A node is listed but already stopped | The CLI daemon cached stale data. Run `ros2 daemon stop`, then try again |
| Two nodes should be connected but are not | Compare their `ros2 node info` output: are the topic **names** and **types** exactly the same? |

The last check finds a surprising number of bugs. A publisher on `/cmd_vel` and a subscriber on `/turtle1/cmd_vel` will never connect, and neither node reports an error.

---

## Try It Yourself

1. Start turtlesim in the namespace `/robot1` and check how its name appears in `ros2 node list`.
2. Start turtlesim and teleop with mismatched namespaces (`/robot1` and `/robot2`). Use `ros2 node info` on both to explain why the turtle does not move.
3. Launch `turtlesim multisim.launch.py` and use `ros2 node list --count-nodes` to count its nodes.
4. Run `ros2 node info /turtlesim --include-hidden` and find the hidden entries that belong to the `rotate_absolute` action.

---

[⬆️ CLI Tools](README.md) | [Next: ros2 topic](ros2-topic.md) ➡️
