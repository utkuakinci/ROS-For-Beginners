# ros2 action

`ros2 action` lets you discover actions, see who provides and uses them, and send goals from the terminal. It is the easiest way to test an action server, for example to send a navigation goal to a robot without writing a client.

> 📖 Concepts: [Actions](../01-ros2-basics/actions.md)

## Topics Covered

- Listing actions
- Inspecting action servers and clients
- Sending goals and watching feedback
- Cancelling goals
- Troubleshooting

---

## Command Overview

| Command | Purpose |
|---------|---------|
| `ros2 action list` | List all actions |
| `ros2 action info <action>` | Show the action's servers and clients |
| `ros2 action send_goal <action> <type> <goal>` | Send a goal and wait for the result |

Use `ros2 interface show <type>` to see an action's goal, result, and feedback fields.

The examples below assume turtlesim and the teleop node are running.

---

## Listing Actions

```bash
$ ros2 action list
/turtle1/rotate_absolute
```

| Option | Effect |
|--------|--------|
| `-t`, `--show-types` | Show each action's type |
| `-c`, `--count-actions` | Print only the number of actions |

```bash
$ ros2 action list -t
/turtle1/rotate_absolute [turtlesim/action/RotateAbsolute]
```

---

## `ros2 action info`

```bash
$ ros2 action info /turtle1/rotate_absolute
Action: /turtle1/rotate_absolute
Action clients: 1
    /teleop_turtle
Action servers: 1
    /turtlesim
```

| Option | Effect |
|--------|--------|
| `-t`, `--show-types` | Also show the type next to each node |
| `-c`, `--count` | Print only the number of clients and servers |

If **Action servers** is `0`, any goal you send will wait forever. Check this first when a goal is not accepted.

---

## Inspecting an Action Type

An action type has three sections: **goal**, **result**, and **feedback**, in that order.

```bash
$ ros2 interface show turtlesim/action/RotateAbsolute
# The desired heading in radians
float32 theta
---
# The angular displacement in radians to the starting position
float32 delta
---
# The remaining rotation in radians
float32 remaining
```

---

## `ros2 action send_goal`

```bash
ros2 action send_goal [options] <action> <type> "<goal in YAML>"
```

```bash
$ ros2 action send_goal /turtle1/rotate_absolute turtlesim/action/RotateAbsolute "{theta: 1.57}"
Waiting for an action server to become available...
Sending goal:
     theta: 1.57

Goal accepted with ID: f8db8f44410849eaa93d3feb747dd444

Result:
    delta: -1.5679999589920044

Goal finished with status: SUCCEEDED
```

| Option | Effect |
|--------|--------|
| `-f`, `--feedback` | Print feedback messages while the goal runs |

```bash
$ ros2 action send_goal -f /turtle1/rotate_absolute turtlesim/action/RotateAbsolute "{theta: 3.14}"
...
Feedback:
    remaining: 1.5508...

Feedback:
    remaining: 1.5188...
...
```

### Cancelling a goal

Press **`Ctrl` + `C`** while `send_goal` is running. The CLI sends a cancel request to the server, and the goal ends with status `CANCELED`.

### Final statuses

| Status | Meaning |
|--------|---------|
| `SUCCEEDED` | The server completed the goal |
| `CANCELED` | The goal was cancelled at the client's request |
| `ABORTED` | The server stopped the goal itself (error, or replaced by a newer goal) |
| `Goal was rejected.` | The server refused the goal before starting, e.g. because it was invalid |

---

## A Look Ahead: Real-World Actions

The same commands work for any action. Once you reach the navigation module, for example, you can send a robot to a position on the map like this:

```bash
$ ros2 action send_goal -f /navigate_to_pose nav2_msgs/action/NavigateToPose \
    "{pose: {header: {frame_id: map}, pose: {position: {x: 2.0, y: 1.0}}}}"
```

Feedback then reports the robot's current pose and the distance remaining.

---

## Troubleshooting

| Symptom | Likely cause |
|---------|--------------|
| `Waiting for an action server to become available...` forever | No server: check `ros2 action info`, the namespace, and `ROS_DOMAIN_ID` |
| `Goal was rejected.` | The server considered the goal invalid. Check the values and the server's log |
| Status `ABORTED` right after starting | Another client sent a newer goal, or the server hit an error. Check the server's log |
| The goal runs, but you see no progress | You forgot `--feedback` |

---

## Try It Yourself

1. Send a goal to rotate the turtle to `-1.57` with feedback. Count roughly how many feedback messages arrive.
2. Send a goal and cancel it halfway with `Ctrl` + `C`. Check the turtlesim terminal for the log message.
3. Start a `send_goal` to `3.14`, and while it runs, send another goal to `0.0` from a second terminal. What status does each terminal report?
4. Stop turtlesim and send a goal. What does the CLI do? Start turtlesim again while it waits.

---

⬅️ [Previous: ros2 service](ros2-service.md) | [⬆️ CLI Tools](README.md) | [Next: ros2 param](ros2-param.md) ➡️
