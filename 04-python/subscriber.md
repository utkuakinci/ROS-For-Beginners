# Writing a Subscriber

A **subscriber** receives messages from a topic and runs a callback for each one. In this lesson you write a `listener` for the talker from the previous lesson, then a `wall_bouncer` node that both subscribes and publishes: it reads the turtle's position and steers it away from the walls. This sense → decide → act loop is the core of almost every robot behavior.

## Topics Covered

- Creating a subscription
- Subscription callbacks and spinning
- Reading data from other nodes
- Combining a subscriber and a publisher in one node
- Storing the latest message and acting on a timer
- Throttled logging
- QoS for subscribers

---

## A Minimal Subscriber: `listener`

Create `~/ros2_ws/src/my_first_pkg/my_first_pkg/listener.py`:

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String


class Listener(Node):
    def __init__(self):
        super().__init__('listener')
        self.subscription = self.create_subscription(
            String, 'chatter', self.listener_callback, 10)

    def listener_callback(self, msg):
        self.get_logger().info(f'I heard: "{msg.data}"')


def main():
    rclpy.init()
    node = Listener()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.try_shutdown()


if __name__ == '__main__':
    main()
```

### How it works

```python
self.subscription = self.create_subscription(
    String, 'chatter', self.listener_callback, 10)
```

`create_subscription(message_type, topic_name, callback, qos)`:

- The **type** and **topic name** must match the publisher exactly, otherwise the two never connect.
- **`callback`** is called once for every message received, with the message as its argument.
- **`10`** is the queue depth: up to 10 messages are buffered if the callback can't keep up.

Keep a reference to the subscription (`self.subscription = ...`). The node holds it, but storing it makes it clear that the subscription belongs to the node, and lets you destroy it later if needed.

```python
def listener_callback(self, msg):
    self.get_logger().info(f'I heard: "{msg.data}"')
```

The callback receives a `String` object and reads its `data` field.

### Run it

Add `'listener = my_first_pkg.listener:main'` to `entry_points` in `setup.py`, rebuild, and run it together with your talker:

```bash
$ ros2 run my_first_pkg talker        # Terminal 1
```

```bash
$ ros2 run my_first_pkg listener      # Terminal 2
[INFO] [1730700001.124567890] [listener]: I heard: "Hello World: 3"
[INFO] [1730700001.624567890] [listener]: I heard: "Hello World: 4"
...
```

Start the listener before the talker, or stop and restart the talker: the listener simply waits and picks up messages whenever they arrive. Neither node knows about the other.

---

## How Callbacks and Spinning Work

```text
          rclpy.spin(node)
               │
     ┌─────────▼──────────┐
     │  wait for an event │◀───────────────────┐
     └─────────┬──────────┘                    │
               │ message arrived / timer due   │
     ┌─────────▼──────────┐                    │
     │  call its callback │────────────────────┘
     └────────────────────┘
```

`spin` runs an **executor**, a loop that waits for events and calls the matching callback. By default, a node's callbacks run **one at a time** in a single thread. This has two consequences:

- You don't need locks to share data between callbacks of the same node.
- A slow callback blocks everything else. If a callback takes 2 seconds, no other message or timer is handled during those 2 seconds.

> ⚠️ **Never** call `time.sleep()` or run long loops in a callback. Do the work in small steps, for example with a timer, and keep callbacks short.

---

## Sense, Decide, Act: `wall_bouncer`

Turtlesim publishes the turtle's position on `/turtle1/pose` (type `turtlesim/msg/Pose`, see [Topics](../01-ros2-basics/topics.md#message-types)). The window spans roughly 0 to 11 in both x and y. Let's write a node that drives forward and turns whenever the turtle gets close to a wall.

Create `my_first_pkg/wall_bouncer.py`:

```python
import rclpy
from geometry_msgs.msg import Twist
from rclpy.node import Node
from turtlesim.msg import Pose


class WallBouncer(Node):
    def __init__(self):
        super().__init__('wall_bouncer')
        self.declare_parameter('margin', 1.5)

        self.pose = None
        self.pose_subscription = self.create_subscription(
            Pose, 'turtle1/pose', self.pose_callback, 10)
        self.cmd_publisher = self.create_publisher(Twist, 'turtle1/cmd_vel', 10)
        self.timer = self.create_timer(0.1, self.control_loop)

    def pose_callback(self, msg):
        # Only store the latest pose; the decision is made in the control loop.
        self.pose = msg

    def control_loop(self):
        if self.pose is None:
            self.get_logger().info('Waiting for the first pose...', throttle_duration_sec=2.0)
            return

        margin = self.get_parameter('margin').value
        near_wall = (
            self.pose.x < margin or self.pose.x > 11.0 - margin
            or self.pose.y < margin or self.pose.y > 11.0 - margin
        )

        cmd = Twist()
        if near_wall:
            cmd.linear.x = 0.5
            cmd.angular.z = 2.0
            self.get_logger().info(
                f'Wall ahead at ({self.pose.x:.1f}, {self.pose.y:.1f}), turning',
                throttle_duration_sec=1.0)
        else:
            cmd.linear.x = 2.0
        self.cmd_publisher.publish(cmd)


def main():
    rclpy.init()
    node = WallBouncer()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.try_shutdown()


if __name__ == '__main__':
    main()
```

### What is new

**One node, several communication objects.** The node has a subscription, a publisher, and a timer. A node can have as many as it needs.

**Store the latest message, act on a timer.** The subscription callback only saves the pose. The timer callback (`control_loop`) makes the decision and publishes. This pattern is very common because:

- Sensors and controllers often run at different rates.
- The control loop runs at a fixed, predictable rate (10 Hz here), regardless of how often data arrives.
- Callbacks stay short.

**Handle missing data.** Until the first pose arrives, `self.pose` is `None`. The control loop checks for that and does nothing yet. Always ask yourself: *what should my node do before it has received any data?*

**Throttled logging.** Logging inside a 10 Hz loop floods the terminal. `throttle_duration_sec=1.0` prints the message at most once per second.

### Run it

Add `'wall_bouncer = my_first_pkg.wall_bouncer:main'` to `setup.py` and `<depend>turtlesim</depend>` to `package.xml`, then rebuild:

```bash
$ ros2 run turtlesim turtlesim_node      # Terminal 1
```

```bash
$ ros2 run my_first_pkg wall_bouncer     # Terminal 2
[INFO] [1730700010.000000000] [wall_bouncer]: Wall ahead at (9.6, 5.5), turning
...
```

The turtle drives around the window, turning away from each wall. Experiment while it runs:

```bash
$ ros2 param set /wall_bouncer margin 3.0
```

Run `rqt_graph`: your node now both receives from and sends to turtlesim, closing the loop.

```text
              /turtle1/pose
  ┌────────────┐ ──────────────▶ ┌──────────────┐
  │ /turtlesim │                 │ /wall_bouncer│
  └────────────┘ ◀────────────── └──────────────┘
              /turtle1/cmd_vel
```

---

## QoS for Subscribers

A subscriber's QoS must be **compatible** with the publisher's, or they won't connect, without any error message. The most common case: sensor drivers often publish with **best effort** reliability, and a subscriber that requests **reliable** delivery cannot receive from them.

For sensor data, subscribe with the sensor data profile:

```python
from rclpy.qos import qos_profile_sensor_data
from sensor_msgs.msg import LaserScan

self.create_subscription(LaserScan, 'scan', self.scan_callback, qos_profile_sensor_data)
```

A best-effort subscriber can receive from both best-effort and reliable publishers, so it is the safe choice for sensor topics. Check what a publisher uses with `ros2 topic info <topic> --verbose`.

---

## Common Mistakes

| Symptom | Cause |
|---------|-------|
| The callback is never called | Topic name or type mismatch (check with `ros2 topic info`), QoS incompatibility, or `spin` never runs |
| `TypeError: listener_callback() takes 1 positional argument but 2 were given` | The callback is missing its `msg` parameter |
| `AttributeError: 'NoneType' object has no attribute 'x'` | Using the stored message before the first one arrived. Check for `None` |
| The node reacts slowly or skips messages | A callback does too much work or sleeps. Keep callbacks short |
| The terminal is flooded with log output | Logging in a fast callback. Use `throttle_duration_sec` |

---

## Try It Yourself

1. Change the listener so it counts the messages it received and prints the count every 5 seconds using a timer.
2. Write a node `pose_reporter` that subscribes to `/turtle1/pose` and logs the turtle's position and heading in degrees, at most once per second.
3. Improve `wall_bouncer`: turn left or right depending on which way is away from the wall.
4. Spawn a second turtle (`ros2 service call /spawn ...`) and run a second `wall_bouncer` for it by remapping its topics: `--ros-args -r turtle1/pose:=turtle2/pose -r turtle1/cmd_vel:=turtle2/cmd_vel -r __node:=wall_bouncer2`.
5. Write a node that subscribes to `/turtle1/pose` and publishes the turtle's distance from the center of the window as a `std_msgs/msg/Float64` on `/distance_to_center`.

---

⬅️ [Previous: Writing a Publisher](publisher.md) | [⬆️ Python (rclpy)](README.md) | [Next: Writing a Service Server and Client](services.md) ➡️
