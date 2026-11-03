# 04. ROS 2 Client Libraries: Python (rclpy)

Hands-on implementation of the core ROS 2 communication mechanisms in Python.

In [Module 01](../01-ros2-basics/README.md) you explored topics, services, and actions from the command line. Now you write them yourself with **rclpy**, the ROS 2 Python client library. Every lesson adds new nodes to the `my_first_pkg` package you created in [Module 03](../03-workspaces-and-packages/README.md), and most of them control turtlesim, so you can see your code working on screen.

## Lessons

1. [Writing a Publisher](publisher.md) — Create a node that publishes messages on a topic.
2. [Writing a Subscriber](subscriber.md) — Create a node that receives messages from a topic.
3. [Writing a Service Server and Client](services.md) — Implement request/response communication.
4. [Writing an Action Server and Client](actions.md) — Implement a long-running, cancellable task.
5. [Custom Interfaces](custom-messages.md) — Define your own message, service, and action types.

## What You Will Build

| Lesson | Executable | What it does |
|--------|------------|--------------|
| Publisher | `talker` | Publishes text messages on `/chatter` |
| Publisher | `circle_driver` | Drives the turtle in a circle |
| Subscriber | `listener` | Prints messages from `/chatter` |
| Subscriber | `wall_bouncer` | Reads the turtle's pose and turns away from walls |
| Services | `add_two_ints_server` / `add_two_ints_client` | A minimal service server and client |
| Services | `turtle_spawner` | Calls turtlesim's `/spawn` service from code |
| Actions | `fibonacci_server` | An action server with feedback and cancellation |
| Actions | `rotate_client` | Sends rotation goals to turtlesim |
| Custom Interfaces | `status_publisher` | Publishes a custom `TurtleStatus` message |

## Before You Start

You need:

- The workspace `~/ros2_ws` with the `my_first_pkg` package from [Module 03](../03-workspaces-and-packages/setup-py.md).
- A terminal for building and at least two for running nodes.
- The habit of **rebuilding and re-sourcing** after adding a new executable:

```bash
$ cd ~/ros2_ws
$ colcon build --packages-select my_first_pkg --symlink-install
$ source install/setup.bash
```

---

⬅️ [Previous: Workspaces & Packages](../03-workspaces-and-packages/README.md) | [🏠 Home](../README.md) | [Next: ROS 2 Client Libraries: C++ (rclcpp)](../05-cpp/README.md) ➡️
