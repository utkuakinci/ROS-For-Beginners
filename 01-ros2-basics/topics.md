# Topics

**Topics** are the most common way nodes communicate in ROS 2. A topic is a named channel that carries a continuous stream of messages from **publishers** to **subscribers**. Sensor data, velocity commands, robot state: most data in a robot flows over topics.

## Topics Covered

- Publishers and subscribers
- Message types and interfaces
- Inspecting topics (`list`, `info`, `echo`, `hz`)
- Publishing from the command line
- Many-to-many communication
- Quality of Service (QoS) in brief
- When to use a topic

---

## Publishers and Subscribers

A node that sends data on a topic is a **publisher**; a node that receives it is a **subscriber**. The two never talk to each other directly. They only agree on the topic's **name** and **message type**.

```text
                         /turtle1/cmd_vel
┌────────────────┐   geometry_msgs/msg/Twist   ┌────────────────┐
│ /teleop_turtle │ ──────────────────────────▶ │   /turtlesim   │
│  (publisher)   │                             │  (subscriber)  │
└────────────────┘                             └────────────────┘
```

This **decoupling** is the key idea: the publisher doesn't know or care who is listening (if anyone), and the subscriber doesn't care who is sending. You can add, remove, or replace nodes on either side without changing the others.

Topics are **one-way** and **fire-and-forget**: a publisher sends messages whether or not anyone receives them, and it never gets a reply.

---

## Listing Topics

Start turtlesim and the teleop node in two terminals, then:

```bash
$ ros2 topic list
/parameter_events
/rosout
/turtle1/cmd_vel
/turtle1/color_sensor
/turtle1/pose
```

Add `-t` to show each topic's message type:

```bash
$ ros2 topic list -t
/parameter_events [rcl_interfaces/msg/ParameterEvent]
/rosout [rcl_interfaces/msg/Log]
/turtle1/cmd_vel [geometry_msgs/msg/Twist]
/turtle1/color_sensor [turtlesim/msg/Color]
/turtle1/pose [turtlesim/msg/Pose]
```

---

## Message Types

Every topic has exactly one **message type** that defines the structure of its data. Type names have three parts:

```text
geometry_msgs / msg / Twist
     │           │      │
  package      kind    name
```

See the fields of a type with `ros2 interface show`:

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
```

A `Twist` is a velocity: `linear` is speed along each axis (m/s), and `angular` is rotation speed around each axis (rad/s). Turtlesim only uses `linear.x` (forward/backward) and `angular.z` (turning left/right).

```bash
$ ros2 interface show turtlesim/msg/Pose
float32 x
float32 y
float32 theta

float32 linear_velocity
float32 angular_velocity
```

Messages are built from basic types (`bool`, `int32`, `float64`, `string`, ...), arrays, and other messages. A few message packages you will see everywhere:

| Package | Contains | Examples |
|---------|----------|----------|
| `std_msgs` | Simple types | `String`, `Int32`, `Bool`, `Header` |
| `geometry_msgs` | Positions, orientations, velocities | `Twist`, `Pose`, `Point`, `TransformStamped` |
| `sensor_msgs` | Sensor data | `LaserScan`, `Image`, `Imu`, `JointState` |
| `nav_msgs` | Navigation | `Odometry`, `Path`, `OccupancyGrid` |

You will create your own message types in [Module 04](../04-python/custom-messages.md).

---

## Inspecting a Topic

### `ros2 topic info`: who is connected

```bash
$ ros2 topic info /turtle1/cmd_vel
Type: geometry_msgs/msg/Twist
Publisher count: 1
Subscription count: 1
```

Add `--verbose` to see *which* nodes, along with their QoS settings.

### `ros2 topic echo`: see the data

```bash
$ ros2 topic echo /turtle1/pose
x: 5.544444561004639
y: 5.544444561004639
theta: 0.0
linear_velocity: 0.0
angular_velocity: 0.0
---
x: 5.544444561004639
...
```

Each block separated by `---` is one message. Now drive the turtle with the teleop node and watch the values change. Stop with `Ctrl` + `C`.

> 💡 `ros2 topic echo` is itself a subscriber. Run `ros2 topic info /turtle1/pose` while echoing and you will see the subscription count go up.

### `ros2 topic hz`: how fast

```bash
$ ros2 topic hz /turtle1/pose
average rate: 62.502
        min: 0.015s max: 0.017s std dev: 0.00045s window: 64
```

Turtlesim publishes the pose about 62 times per second. Checking the rate is one of the first things to do when debugging sensor data.

---

## Publishing from the Command Line

You can publish messages yourself, without writing any code. Stop the teleop node, then send a single velocity command:

```bash
$ ros2 topic pub --once /turtle1/cmd_vel geometry_msgs/msg/Twist \
    "{linear: {x: 2.0, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 1.8}}"
publisher: beginning loop
publishing #1: geometry_msgs.msg.Twist(linear=geometry_msgs.msg.Vector3(x=2.0, y=0.0, z=0.0), angular=geometry_msgs.msg.Vector3(x=0.0, y=0.0, z=1.8))
```

The turtle moves along a short arc and stops. The message is written in YAML; fields you leave out default to zero, so this is equivalent:

```bash
$ ros2 topic pub --once /turtle1/cmd_vel geometry_msgs/msg/Twist "{linear: {x: 2.0}, angular: {z: 1.8}}"
```

To keep publishing, give a rate in Hz instead of `--once`:

```bash
$ ros2 topic pub --rate 1 /turtle1/cmd_vel geometry_msgs/msg/Twist "{linear: {x: 2.0}, angular: {z: 1.8}}"
```

The turtle now drives in circles until you press `Ctrl` + `C`.

> 💡 Press `Tab` after typing the message type: the CLI autocompletes a full message template for you.

---

## Many-to-Many Communication

A topic can have any number of publishers and subscribers.

```text
┌────────────┐                        ┌──────────────┐
│ teleop_key │ ──┐                ┌─▶ │  turtlesim   │
└────────────┘   │  /turtle1/     │   └──────────────┘
┌────────────┐   ├─  cmd_vel   ───┤   ┌──────────────┐
│ topic pub  │ ──┘                └─▶ │ topic echo   │
└────────────┘                        └──────────────┘
```

Try it: run the teleop node *and* the `ros2 topic pub --rate 1` command at the same time, while echoing `/turtle1/cmd_vel` in another terminal. Turtlesim receives messages from both publishers.

Many subscribers is common and useful (a camera image used by several algorithms). Many publishers on a command topic, however, usually means they fight over control. Avoid that in real robots.

---

## Quality of Service (QoS) in Brief

Every publisher and subscriber has **Quality of Service** settings that control *how* messages are delivered. The most important ones are:

| Setting | Options | Meaning |
|---------|---------|---------|
| **Reliability** | `reliable` / `best_effort` | Retry until delivered, or send once and accept losses |
| **Durability** | `volatile` / `transient_local` | Late subscribers get nothing, or get the last messages sent |
| **History / Depth** | number of messages | How many messages to queue |

The defaults (reliable, volatile, depth 10) work for most cases. Sensor drivers, however, often publish **best effort** for speed, and a **reliable** subscriber cannot connect to a best-effort publisher. When a topic exists but you receive no data, a QoS mismatch is a prime suspect. `ros2 topic info --verbose` shows each endpoint's QoS.

You will configure QoS in code in [Module 04](../04-python/README.md).

---

## When to Use a Topic

Use a topic when:

- Data is produced **continuously** or periodically (sensor readings, state, commands).
- The sender does **not need a response**.
- **Several** nodes may be interested in the same data.
- Missing an occasional message is acceptable because a newer one is coming.

If you need an answer to a request, use a [service](services.md). If you need to start a long task and track it, use an [action](actions.md).

---

## Try It Yourself

1. Make the turtle drive in a large circle using `ros2 topic pub --rate`. Then make it drive in a smaller circle by changing only one value.
2. Use `ros2 topic echo /turtle1/pose --field x` to print only the turtle's x position while you drive it.
3. Run `ros2 interface show turtlesim/msg/Color`, then echo `/turtle1/color_sensor`. What does this topic report?
4. Run `rqt_graph` while publishing from the command line. Where does the `ros2 topic pub` node appear?
5. Find the message type of `/rosout` and inspect it with `ros2 interface show`. Which fields does a log message have?

---

⬅️ [Previous: Nodes](nodes.md) | [⬆️ ROS 2 Core Concepts](README.md) | [Next: Services](services.md) ➡️
