# ros2 topic

`ros2 topic` is the command you will use most. It lets you see every data stream in the system, print it, measure it, and publish your own messages, all without writing code. This lesson also covers `ros2 interface`, which shows what a message type contains.

> 📖 Concepts: [Topics](../01-ros2-basics/topics.md)

## Topics Covered

- Listing and finding topics
- Printing messages with `echo` and its filtering options
- Measuring rate, bandwidth, and delay
- Publishing messages with `pub`
- QoS options on the command line
- Inspecting message types with `ros2 interface`

---

## Command Overview

| Command | Purpose |
|---------|---------|
| `ros2 topic list` | List all topics |
| `ros2 topic info <topic>` | Show the type and number of publishers and subscribers |
| `ros2 topic type <topic>` | Print only the message type |
| `ros2 topic find <type>` | List topics that use a given message type |
| `ros2 topic echo <topic>` | Print messages as they arrive |
| `ros2 topic hz <topic>` | Measure the publishing rate |
| `ros2 topic bw <topic>` | Measure the bandwidth used |
| `ros2 topic delay <topic>` | Measure latency using the message's header timestamp |
| `ros2 topic pub <topic> <type> <data>` | Publish messages |

The examples below assume turtlesim is running (`ros2 run turtlesim turtlesim_node`).

---

## Listing and Finding Topics

```bash
$ ros2 topic list
/parameter_events
/rosout
/turtle1/cmd_vel
/turtle1/color_sensor
/turtle1/pose
```

| Option | Effect |
|--------|--------|
| `-t`, `--show-types` | Show each topic's message type |
| `-c`, `--count-topics` | Print only the number of topics |
| `-v`, `--verbose` | Show the publishers and subscribers of every topic |
| `--include-hidden-topics` | Also show hidden topics (for example, action internals) |

```bash
$ ros2 topic list -t
/turtle1/cmd_vel [geometry_msgs/msg/Twist]
/turtle1/pose [turtlesim/msg/Pose]
...

$ ros2 topic type /turtle1/pose
turtlesim/msg/Pose

$ ros2 topic find geometry_msgs/msg/Twist
/turtle1/cmd_vel
```

`ros2 topic find` is handy on a large robot: "which topic carries the velocity commands?"

---

## `ros2 topic info`

```bash
$ ros2 topic info /turtle1/cmd_vel
Type: geometry_msgs/msg/Twist
Publisher count: 0
Subscription count: 1
```

With `-v` (`--verbose`), it lists every endpoint with its node name and QoS profile:

```bash
$ ros2 topic info /turtle1/cmd_vel --verbose
Type: geometry_msgs/msg/Twist

Publisher count: 0

Subscription count: 1

Node name: turtlesim
Node namespace: /
Topic type: geometry_msgs/msg/Twist
Endpoint type: SUBSCRIPTION
GID: ...
QoS profile:
  Reliability: RELIABLE
  History (Depth): KEEP_LAST (10)
  Durability: VOLATILE
  ...
```

This is the command to use when a topic exists but data does not arrive: compare the **Reliability** and **Durability** of the publisher and subscriber.

---

## `ros2 topic echo`

```bash
$ ros2 topic echo /turtle1/pose
x: 5.544444561004639
y: 5.544444561004639
theta: 0.0
linear_velocity: 0.0
angular_velocity: 0.0
---
...
```

Large messages (laser scans, images, point clouds) flood the terminal. These options help:

| Option | Effect |
|--------|--------|
| `--once` | Print one message and exit |
| `--field <name>` | Print only one field, e.g. `--field x` or `--field linear.x` |
| `--no-arr` | Hide array contents (print only their length) |
| `--no-str` | Hide string contents |
| `--csv` | Print each message as one comma-separated line |
| `--flow-style` | Print nested data on a single line |
| `--filter "<python expression>"` | Print only messages for which the expression is true; the message is `m` |

```bash
$ ros2 topic echo /turtle1/pose --once
$ ros2 topic echo /turtle1/pose --field theta
$ ros2 topic echo /turtle1/pose --csv
5.544444561004639,5.544444561004639,0.0,0.0,0.0
...
$ ros2 topic echo /turtle1/pose --filter "m.x > 8.0"
```

The last command prints poses only while the turtle is in the right part of the window, which is a quick way to watch for a specific condition.

> 💡 Save data for later with standard shell redirection: `ros2 topic echo /turtle1/pose --csv > pose.csv`. For proper recording and playback, use `rosbag2` ([Module 11](../11-debugging/rosbag.md)).

---

## Measuring Rate, Bandwidth, and Delay

### `ros2 topic hz`

```bash
$ ros2 topic hz /turtle1/pose
average rate: 62.502
        min: 0.015s max: 0.017s std dev: 0.00045s window: 64
```

| Option | Effect |
|--------|--------|
| `-w`, `--window <N>` | Number of messages used for the average (smaller reacts faster) |

Check the rate against what you expect from a sensor's datasheet. A camera that should run at 30 Hz but shows 12 Hz points to a driver, CPU, or network problem.

### `ros2 topic bw`

```bash
$ ros2 topic bw /turtle1/pose
Subscribed to [/turtle1/pose]
1.25 KB/s from 62 messages
        Message size mean: 0.02 KB min: 0.02 KB max: 0.02 KB
...
```

Bandwidth matters when data travels over Wi-Fi. A single uncompressed camera stream can use tens of megabytes per second.

### `ros2 topic delay`

```bash
$ ros2 topic delay /scan
```

Shows the time between the timestamp inside each message's `header` and the moment it was received. It only works for message types that have a `std_msgs/Header` (most sensor messages do; turtlesim's `Pose` does not).

---

## `ros2 topic pub`

```bash
ros2 topic pub [options] <topic> <type> "<data in YAML>"
```

```bash
$ ros2 topic pub --once /turtle1/cmd_vel geometry_msgs/msg/Twist "{linear: {x: 2.0}, angular: {z: 1.8}}"
```

| Option | Effect |
|--------|--------|
| `--once`, `-1` | Publish one message, then exit |
| `-r`, `--rate <Hz>` | Publishing rate (default: 1 Hz) |
| `-t`, `--times <N>` | Publish N messages, then exit |
| `-w`, `--wait-matching-subscriptions <N>` | Wait until N subscribers are connected before publishing |
| `-p`, `--print <N>` | Print only every Nth published message |

Examples:

```bash
# Drive in a circle at 10 Hz until Ctrl+C
$ ros2 topic pub -r 10 /turtle1/cmd_vel geometry_msgs/msg/Twist "{linear: {x: 2.0}, angular: {z: 1.8}}"

# Publish exactly 5 messages
$ ros2 topic pub -t 5 /turtle1/cmd_vel geometry_msgs/msg/Twist "{linear: {x: 1.0}}"

# Publish a simple string
$ ros2 topic pub --once /chatter std_msgs/msg/String "{data: 'Hello from the CLI'}"
```

### Writing the YAML data

- Fields you omit are set to their default (zero, `false`, or an empty string).
- Nested messages use nested braces: `{linear: {x: 1.0}}`.
- Arrays use square brackets: `{data: [1, 2, 3]}`.
- Wrap the whole data in double quotes so the shell does not interpret it.
- Press `Tab` after the type to have the CLI write a full template for you.

> ⚠️ A topic with a **single** `--once` publish may arrive before the subscriber has connected and get lost. If a one-shot message seems to have no effect, add `-w 1` so the CLI waits for a subscriber first.

---

## QoS Options

`echo`, `hz`, and `pub` accept QoS options so they can match publishers and subscribers that use non-default settings:

| Option | Values |
|--------|--------|
| `--qos-profile` | `system_default`, `sensor_data`, `services_default`, `parameters`, ... |
| `--qos-reliability` | `reliable`, `best_effort` |
| `--qos-durability` | `volatile`, `transient_local` |
| `--qos-depth` | Queue size |

Two common cases:

```bash
# A sensor driver publishes best-effort data
$ ros2 topic echo /scan --qos-reliability best_effort

# A topic that keeps its last message for late subscribers, like /map or /robot_description
$ ros2 topic echo /robot_description --qos-durability transient_local
```

---

## `ros2 interface`: Inspecting Message Types

Every topic, service, and action uses an **interface** type. `ros2 interface` lets you explore them.

| Command | Purpose |
|---------|---------|
| `ros2 interface show <type>` | Show the fields of a type |
| `ros2 interface proto <type>` | Print a YAML template you can paste into `pub` or `call` |
| `ros2 interface list` | List all installed messages, services, and actions |
| `ros2 interface packages` | List packages that define interfaces |
| `ros2 interface package <pkg>` | List the interfaces in one package |

```bash
$ ros2 interface show geometry_msgs/msg/Twist
# This expresses velocity in free space broken into its linear and angular parts.

Vector3  linear
        float64 x
        float64 y
        float64 z
Vector3  angular
        float64 x
        float64 y
        float64 z

$ ros2 interface proto geometry_msgs/msg/Twist
"linear:
  x: 0.0
  y: 0.0
  z: 0.0
angular:
  x: 0.0
  y: 0.0
  z: 0.0
"

$ ros2 interface package turtlesim
turtlesim/msg/Color
turtlesim/msg/Pose
turtlesim/srv/Kill
turtlesim/srv/SetPen
turtlesim/srv/Spawn
turtlesim/srv/TeleportAbsolute
turtlesim/srv/TeleportRelative
turtlesim/action/RotateAbsolute
```

`ros2 interface show` works for services (`.srv`) and actions (`.action`) too.

---

## Try It Yourself

1. Use `ros2 topic find` to find the topic of type `turtlesim/msg/Color`, then print only its `r` field.
2. Make the turtle drive straight for exactly 3 seconds at 1 Hz using `--times`.
3. Print the turtle's pose only while it is moving forward (`linear_velocity > 0`) using `--filter`, while you drive with the teleop node.
4. Save one minute of poses to a CSV file and open it in a spreadsheet program.
5. Use `ros2 interface package sensor_msgs` to list the sensor message types, then show the fields of `sensor_msgs/msg/LaserScan`.

---

⬅️ [Previous: ros2 node](ros2-node.md) | [⬆️ CLI Tools](README.md) | [Next: ros2 service](ros2-service.md) ➡️
