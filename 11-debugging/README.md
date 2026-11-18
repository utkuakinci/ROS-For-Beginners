# 11. Debugging & Troubleshooting

Finding out why a ROS 2 system is not doing what you expect.

Every ROS 2 developer spends a large part of their time debugging. A robot is a distributed system: a dozen processes, hundreds of topics and transforms, real or simulated time, possibly several computers. When something goes wrong, the error is often silent: no crash, just a robot that doesn't move. This module collects the most common errors in one place, introduces the diagnostic and recording tools, and teaches a systematic workflow that finds problems quickly.

## Lessons

1. [Common Errors](common-errors.md) — The errors beginners hit most often and how to fix them.
2. [ros2 doctor](ros2-doctor.md) — Check the health of your ROS 2 setup.
3. [rosbag2](rosbag.md) — Record and replay data from a running system.
4. [Debugging Workflow](debugging-workflow.md) — A repeatable process for diagnosing problems.

## Your Debugging Toolbox

Most of these tools appeared in earlier modules. This table shows which one answers which question:

| Question | Tool | Covered in |
|----------|------|------------|
| Is my environment set up correctly? | `ros2 doctor`, `printenv \| grep ROS` | [ros2 doctor](ros2-doctor.md) |
| Which nodes run, and how are they connected? | `ros2 node list/info`, `rqt_graph` | [ros2 node](../02-ros2-cli/ros2-node.md) |
| Is data flowing, at the right rate? | `ros2 topic echo/hz/info -v` | [ros2 topic](../02-ros2-cli/ros2-topic.md) |
| What does a node have configured? | `ros2 param list/get` | [ros2 param](../02-ros2-cli/ros2-param.md) |
| Is a Nav2 or MoveIt node actually running? | `ros2 lifecycle get` | [Localization](../09-navigation/localization.md#lifecycle-nodes) |
| Are the frames right? | `view_frames`, `tf2_echo`, RViz2 | [TF2 Tools](../06-tf2/tf2-tools.md) |
| What did the nodes log? | Terminal output, `rqt_console`, `~/.ros/log` | [Debugging Workflow](debugging-workflow.md) |
| What exactly happened during that run? | `ros2 bag record/play` | [rosbag2](rosbag.md) |
| Where does my code crash? | `pdb`, `gdb` | [Debugging Workflow](debugging-workflow.md) |

---

⬅️ [Previous: Manipulation](../10-manipulation/README.md) | [🏠 Home](../README.md)
