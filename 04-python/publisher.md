# Writing a Publisher

A **publisher** sends messages on a topic. In this lesson you write two publisher nodes: a minimal `talker` that sends text, and a `circle_driver` that sends velocity commands to turtlesim. Along the way you will learn the structure that every rclpy node follows.

## Topics Covered

- The anatomy of an rclpy node
- Creating a publisher
- Publishing periodically with a timer
- Registering and running the node
- Publishing velocity commands to turtlesim
- Making a node configurable with parameters
- Quality of Service for publishers

---

## The Anatomy of an rclpy Node

Every Python node in this guide follows the same structure:

```python
import rclpy
from rclpy.node import Node


class MyNode(Node):                      # 1. A class that inherits from Node
    def __init__(self):
        super().__init__('my_node')      # 2. Give the node its name
        # 3. Create publishers, subscribers, timers, ... here


def main():
    rclpy.init()                         # 4. Initialize ROS 2 communication
    node = MyNode()                      # 5. Create the node
    try:
        rclpy.spin(node)                 # 6. Process callbacks until stopped
    except KeyboardInterrupt:
        pass                             #    Ctrl+C ends spin
    finally:
        node.destroy_node()              # 7. Clean up
        rclpy.try_shutdown()


if __name__ == '__main__':
    main()
```

| Step | Why |
|------|-----|
| `rclpy.init()` | Sets up the connection to the ROS 2 middleware. Must come before creating nodes |
| `class MyNode(Node)` | Your node gets all of `Node`'s features: publishers, timers, logging, parameters |
| `super().__init__('my_node')` | Registers the node under this name (what `ros2 node list` shows) |
| `rclpy.spin(node)` | Waits for events (timer ticks, incoming messages) and calls your callbacks. Blocks until shutdown |
| `destroy_node()` / `try_shutdown()` | Releases resources cleanly when the node stops |

> 💡 Nothing happens in a node unless `spin` is running. A callback is only ever called from inside `spin`.

---

## A Minimal Publisher: `talker`

Create the file `~/ros2_ws/src/my_first_pkg/my_first_pkg/talker.py`:

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String


class Talker(Node):
    def __init__(self):
        super().__init__('talker')
        self.publisher = self.create_publisher(String, 'chatter', 10)
        self.timer = self.create_timer(0.5, self.timer_callback)
        self.count = 0

    def timer_callback(self):
        msg = String()
        msg.data = f'Hello World: {self.count}'
        self.publisher.publish(msg)
        self.get_logger().info(f'Publishing: "{msg.data}"')
        self.count += 1


def main():
    rclpy.init()
    node = Talker()
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
from std_msgs.msg import String
```

Message types are Python classes generated from the message definitions. `std_msgs/msg/String` becomes `from std_msgs.msg import String`.

```python
self.publisher = self.create_publisher(String, 'chatter', 10)
```

`create_publisher(message_type, topic_name, qos)` creates the publisher:

- **`String`**: every message on this topic must be a `String`.
- **`'chatter'`**: the topic name. It has no leading slash, so it is *relative* and will follow the node's namespace.
- **`10`**: the queue depth, the simplest form of QoS (see [below](#quality-of-service)).

```python
self.timer = self.create_timer(0.5, self.timer_callback)
```

`create_timer(period_in_seconds, callback)` calls `timer_callback` every 0.5 s (2 Hz). Note that you pass the method **without** parentheses.

```python
msg = String()
msg.data = f'Hello World: {self.count}'
self.publisher.publish(msg)
```

Create a message object, fill in its fields (`ros2 interface show std_msgs/msg/String` shows the single field `data`), and publish it.

---

## Registering and Running the Node

### 1. Declare the dependency

The node imports `std_msgs`, so add it to `package.xml` (if you didn't already in [Module 03](../03-workspaces-and-packages/package-xml.md#adding-dependencies)):

```xml
<depend>rclpy</depend>
<depend>std_msgs</depend>
```

### 2. Add an entry point

In `setup.py`, add the executable:

```python
entry_points={
    'console_scripts': [
        'hello_node = my_first_pkg.hello_node:main',
        'talker = my_first_pkg.talker:main',
    ],
},
```

### 3. Build, source, run

```bash
$ cd ~/ros2_ws
$ colcon build --packages-select my_first_pkg --symlink-install
```

```bash
$ source ~/ros2_ws/install/setup.bash
$ ros2 run my_first_pkg talker
[INFO] [1730700000.123456789] [talker]: Publishing: "Hello World: 0"
[INFO] [1730700000.623456789] [talker]: Publishing: "Hello World: 1"
...
```

### 4. Verify with the CLI

In another terminal:

```bash
$ ros2 topic echo /chatter
data: 'Hello World: 5'
---
data: 'Hello World: 6'
---

$ ros2 topic hz /chatter
average rate: 2.000
```

Your publisher works exactly like the `demo_nodes_cpp talker` from [What is ROS 2?](../01-ros2-basics/what-is-ros2.md#talker-and-listener). Try it: `ros2 run demo_nodes_py listener` receives your messages.

---

## Driving the Turtle: `circle_driver`

Now publish something useful: velocity commands. Turtlesim subscribes to `/turtle1/cmd_vel` with type `geometry_msgs/msg/Twist` (see [Topics](../01-ros2-basics/topics.md#message-types)).

Create `my_first_pkg/circle_driver.py`:

```python
import rclpy
from geometry_msgs.msg import Twist
from rclpy.node import Node


class CircleDriver(Node):
    def __init__(self):
        super().__init__('circle_driver')
        self.declare_parameter('linear_speed', 2.0)
        self.declare_parameter('angular_speed', 1.0)

        self.publisher = self.create_publisher(Twist, 'turtle1/cmd_vel', 10)
        self.timer = self.create_timer(0.1, self.timer_callback)

    def timer_callback(self):
        msg = Twist()
        msg.linear.x = self.get_parameter('linear_speed').value
        msg.angular.z = self.get_parameter('angular_speed').value
        self.publisher.publish(msg)


def main():
    rclpy.init()
    node = CircleDriver()
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

What is new:

- **Nested fields:** a `Twist` contains two `Vector3` messages, so you set `msg.linear.x` and `msg.angular.z`. Unset fields default to `0.0`.
- **Parameters:** `declare_parameter(name, default)` declares a parameter with a default value. Its type is taken from the default: `2.0` makes it a `double`. `get_parameter(name).value` reads the current value, so changes made at runtime take effect on the next timer tick.
- **10 Hz:** velocity commands are usually sent continuously. Many robots stop automatically if commands stop arriving, as a safety measure.

Register it in `setup.py` (`'circle_driver = my_first_pkg.circle_driver:main'`), add `<depend>geometry_msgs</depend>` to `package.xml`, rebuild, and run it next to turtlesim:

```bash
$ ros2 run turtlesim turtlesim_node
```

```bash
$ ros2 run my_first_pkg circle_driver
```

The turtle drives in circles. Change its path while it runs:

```bash
$ ros2 param set /circle_driver angular_speed 0.5     # bigger circle
$ ros2 param set /circle_driver linear_speed 0.0      # turn on the spot
```

Or start it with different values:

```bash
$ ros2 run my_first_pkg circle_driver --ros-args -p linear_speed:=1.0 -p angular_speed:=-2.0
```

> 💡 The topic name `turtle1/cmd_vel` is relative. Run the node with `--ros-args -r __ns:=/robot1` and it publishes on `/robot1/turtle1/cmd_vel` instead, matching a turtlesim in the same namespace. Writing relative names in your code keeps nodes reusable.

---

## Quality of Service

The third argument of `create_publisher` is the QoS. Passing an integer is a shortcut for "reliable, volatile, keep the last N messages", which suits most topics. For more control, pass a `QoSProfile`:

```python
from rclpy.qos import QoSProfile, ReliabilityPolicy, DurabilityPolicy, HistoryPolicy

qos = QoSProfile(
    reliability=ReliabilityPolicy.RELIABLE,
    durability=DurabilityPolicy.TRANSIENT_LOCAL,
    history=HistoryPolicy.KEEP_LAST,
    depth=1,
)
self.publisher = self.create_publisher(String, 'robot_status', qos)
```

`TRANSIENT_LOCAL` keeps the last message and delivers it to subscribers that connect later, which is useful for data that rarely changes, like a map or a robot's description.

rclpy also provides ready-made profiles:

```python
from rclpy.qos import qos_profile_sensor_data   # best effort, small queue: for high-rate sensor data
```

See [Topics: QoS in brief](../01-ros2-basics/topics.md#quality-of-service-qos-in-brief) for what the settings mean and how mismatches show up.

---

## Common Mistakes

| Symptom | Cause |
|---------|-------|
| `No executable found` | Entry point missing in `setup.py`, or no rebuild after adding it |
| `ModuleNotFoundError: No module named 'geometry_msgs'` | Dependency not installed; declare it in `package.xml` and run `rosdep install` |
| Node runs but nothing is published | The timer was not created, or `spin` is never reached |
| The turtle doesn't move | Topic name mismatch: check with `ros2 node info /circle_driver` and `ros2 topic info /turtle1/cmd_vel` |
| `AttributeError: 'Twist' object has no attribute 'x'` | Nested message: use `msg.linear.x`, not `msg.x` |
| Error when creating the timer, or the callback runs only once at startup | You wrote `create_timer(0.5, self.timer_callback())`, passing the *result* of a call instead of the function. Remove the parentheses |

---

## Try It Yourself

1. Add a `message` parameter to `talker` so the text can be changed with `ros2 param set`.
2. Change the talker's rate to 10 Hz and confirm with `ros2 topic hz`.
3. Write a node `square_driver` that drives the turtle in a (roughly) square path: drive straight for 2 seconds, turn for about 1.57 rad, repeat. Hint: keep a counter in the timer callback to switch between "drive" and "turn".
4. Add `circle_driver` to `launch/hello.launch.py` from [Module 03](../03-workspaces-and-packages/setup-py.md#completing-my_first_pkg) together with a turtlesim node, and set its speeds in `config/hello.yaml`.

---

[⬆️ Python (rclpy)](README.md) | [Next: Writing a Subscriber](subscriber.md) ➡️
