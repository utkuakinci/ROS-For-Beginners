# Actions

Some tasks take time: driving to a location, rotating to a heading, picking up an object. For these, a service is a poor fit because the client waits blindly and cannot cancel. **Actions** solve this. An action is a long-running task that sends **feedback** while it runs, returns a **result** when it is done, and can be **cancelled** at any time.

## Topics Covered

- Goal, feedback, and result
- Action servers and clients
- Inspecting actions (`list`, `info`)
- Sending goals from the command line
- Cancelling a goal
- How actions are built on topics and services
- When to use an action

---

## Goal, Feedback, and Result

An action interaction has three parts:

| Part | Direction | Purpose |
|------|-----------|---------|
| **Goal** | Client → Server | What to do: "rotate to 90°" |
| **Feedback** | Server → Client (repeatedly) | Progress updates: "45° remaining..." |
| **Result** | Server → Client (once) | The outcome when finished: "rotated by 90°" |

```text
┌──────────┐ ── goal: {theta: 1.57} ───────────▶ ┌──────────┐
│  Action  │ ◀─ accepted ─────────────────────── │  Action  │
│  client  │ ◀─ feedback: {remaining: 1.2} ───── │  server  │
│          │ ◀─ feedback: {remaining: 0.6} ───── │          │
│          │ ◀─ feedback: {remaining: 0.1} ───── │          │
│          │ ◀─ result: {delta: -1.57} ───────── │          │
└──────────┘                                     └──────────┘
```

At any point the client may send a **cancel** request, and the server stops the task. The server can also **reject** a goal it cannot achieve, or **abort** it if something goes wrong.

---

## Actions in Turtlesim

Start turtlesim and the teleop node:

```bash
$ ros2 run turtlesim turtlesim_node        # Terminal 1
$ ros2 run turtlesim turtle_teleop_key     # Terminal 2
```

The teleop node prints:

```text
Use arrow keys to move the turtle.
Use G|B|V|C|D|E|R|T keys to rotate to absolute orientations. 'F' to cancel a rotation.
'Q' to quit.
```

The arrow keys publish on a topic, as you saw in [Topics](topics.md). The letter keys around `F` are different: each one sends an **action goal** to rotate the turtle to an absolute heading, matching their position around `F` on the keyboard: `R` points up, `G` right, `V` down, `D` left, and the others diagonally.

Press `R` and let the turtle finish turning. The turtlesim terminal logs:

```text
[INFO] [turtlesim]: Rotation goal completed successfully
```

Now press `C`, and press `F` before the rotation finishes. The goal is cancelled:

```text
[INFO] [turtlesim]: Rotation goal canceled
```

Finally, press `D` and then quickly press `G` before the first rotation ends. The server accepts the new goal and aborts the old one:

```text
[WARN] [turtlesim]: Rotation goal received before a previous goal finished. Aborting previous goal
```

---

## Listing and Inspecting Actions

```bash
$ ros2 action list
/turtle1/rotate_absolute

$ ros2 action list -t
/turtle1/rotate_absolute [turtlesim/action/RotateAbsolute]
```

`ros2 action info` shows which nodes provide and use the action:

```bash
$ ros2 action info /turtle1/rotate_absolute
Action: /turtle1/rotate_absolute
Action clients: 1
    /teleop_turtle
Action servers: 1
    /turtlesim
```

---

## Action Types

An action type has **three** sections separated by `---`: goal, result, and feedback.

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

| Section | Field | Meaning |
|---------|-------|---------|
| Goal | `theta` | The heading to rotate to (radians) |
| Result | `delta` | How far the turtle rotated in total |
| Feedback | `remaining` | How much rotation is left |

> 📝 Angles in ROS are in **radians**: π/2 ≈ 1.57 is 90°, π ≈ 3.14 is 180°.

---

## Sending a Goal from the Command Line

```bash
ros2 action send_goal <action_name> <action_type> "<goal in YAML>"
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

Add `--feedback` to see progress updates as the turtle turns:

```bash
$ ros2 action send_goal /turtle1/rotate_absolute turtlesim/action/RotateAbsolute "{theta: -1.57}" --feedback
Waiting for an action server to become available...
Sending goal:
     theta: -1.57

Goal accepted with ID: e6092c831f994afda92f0086f220da27

Feedback:
    remaining: -3.1268222332000732

Feedback:
    remaining: -3.0948221683502197

...

Result:
    delta: 3.1200008392333984

Goal finished with status: SUCCEEDED
```

Pressing `Ctrl` + `C` while `send_goal` is running cancels the goal.

The possible final statuses are:

| Status | Meaning |
|--------|---------|
| `SUCCEEDED` | The task completed |
| `CANCELED` | The client cancelled it |
| `ABORTED` | The server gave up (for example, a newer goal replaced it) |

---

## How Actions Work Under the Hood

Actions are not a separate communication mechanism. They are built from **services and topics**:

| Part | Implemented as |
|------|----------------|
| Send goal | Service |
| Cancel goal | Service |
| Get result | Service |
| Feedback | Topic |
| Status | Topic |

These are hidden from `ros2 service list` and `ros2 topic list` by default. Add `--include-hidden-services` or `--include-hidden-topics` to see them:

```bash
$ ros2 service list --include-hidden-services | grep rotate
/turtle1/rotate_absolute/_action/cancel_goal
/turtle1/rotate_absolute/_action/get_result
/turtle1/rotate_absolute/_action/send_goal
```

You never use these directly, but knowing they exist explains why actions behave the way they do.

---

## When to Use an Action

| Use a... | When |
|----------|------|
| **Topic** | Continuous data, no reply needed |
| **Service** | A quick request with a response |
| **Action** | A task that takes time, where you want progress feedback and the ability to cancel |

Real-world actions you will meet later in this guide:

- **Nav2:** `navigate_to_pose`: drive to a goal while reporting distance remaining.
- **MoveIt 2:** move an arm along a planned trajectory.
- **Controllers:** `follow_joint_trajectory`: execute a joint trajectory on a robot arm.

---

## Try It Yourself

1. Send goals to rotate the turtle to face up (π/2), left (π), and down (-π/2). Use `--feedback` for one of them.
2. Start a goal with `--feedback` and cancel it with `Ctrl` + `C` halfway through. What status is reported?
3. While a `send_goal` is running in one terminal, press a rotation key in the teleop terminal. What happens to the first goal, and why?
4. Run `ros2 topic list --include-hidden-topics`. Which hidden topics belong to the action?
5. Would you implement "charge the battery to 100%" as a topic, service, or action? Explain why.

---

⬅️ [Previous: Services](services.md) | [⬆️ ROS 2 Core Concepts](README.md) | [Next: Parameters](parameters.md) ➡️
