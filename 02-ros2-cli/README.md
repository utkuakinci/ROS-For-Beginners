# 02. CLI Tools

The command-line tools used to inspect and interact with a running ROS 2 system.

You already used many of these commands in [Module 01](../01-ros2-basics/README.md) to explore concepts. This module is the **reference**: every subcommand, the options worth knowing, and the tricks that save time when debugging a real robot. Keep it open while you work through the rest of the guide.

## Lessons

1. [ros2 node](ros2-node.md) — List and inspect the nodes in the graph.
2. [ros2 topic](ros2-topic.md) — Inspect, echo, and publish to topics.
3. [ros2 service](ros2-service.md) — Discover and call services from the terminal.
4. [ros2 action](ros2-action.md) — Inspect actions and send goals from the terminal.
5. [ros2 param](ros2-param.md) — Read, set, and save node parameters at runtime.
6. [ros2 run](ros2-run.md) — Start a single executable from a package.
7. [ros2 launch](ros2-launch.md) — Start a whole system from a launch file.

## Quick Reference

| I want to... | Command |
|--------------|---------|
| See which nodes are running | `ros2 node list` |
| See what a node publishes, subscribes, and serves | `ros2 node info <node>` |
| List topics with their types | `ros2 topic list -t` |
| Print messages on a topic | `ros2 topic echo <topic>` |
| Check how fast a topic publishes | `ros2 topic hz <topic>` |
| Publish a message | `ros2 topic pub --once <topic> <type> "<yaml>"` |
| See the fields of a message type | `ros2 interface show <type>` |
| Call a service | `ros2 service call <service> <type> "<yaml>"` |
| Send an action goal with feedback | `ros2 action send_goal -f <action> <type> "<yaml>"` |
| Read / change a parameter | `ros2 param get <node> <name>` / `ros2 param set <node> <name> <value>` |
| Save a node's parameters | `ros2 param dump <node> > params.yaml` |
| Run a node with options | `ros2 run <pkg> <exe> --ros-args -r __ns:=/robot1 -p name:=value` |
| See a launch file's arguments | `ros2 launch <pkg> <file> --show-args` |

## General Tips

- **`Tab` completion works everywhere**: commands, package names, executables, topic names, and even message templates. If completion stops working, the terminal probably has not sourced ROS 2.
- **`-h` shows help** for any command: `ros2 topic -h`, `ros2 topic echo -h`.
- **The CLI uses discovery** like any node. If a command shows nothing, check that you are on the same `ROS_DOMAIN_ID` as the nodes you expect.
- **Daemon:** to answer quickly, the CLI uses a background process called the ROS 2 daemon. If the CLI shows stale results (nodes that no longer exist), restart it with `ros2 daemon stop` followed by `ros2 daemon start`.

---

⬅️ [Previous: ROS 2 Core Concepts](../01-ros2-basics/README.md) | [🏠 Home](../README.md) | [Next: Workspaces & Packages](../03-workspaces-and-packages/README.md) ➡️
