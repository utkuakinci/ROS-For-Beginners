# TF2 Listeners

A **listener** receives all transforms published in the system and lets you ask questions like "where is frame A relative to frame B, right now or at a given time?". In this lesson you write `turtle_follower`, a node that makes `turtle2` chase `turtle1` using only TF2, rebuilding the demo from [Coordinate Frames](coordinate-frames.md#a-first-tf2-demo) yourself. You will also convert data from one frame to another.

## Topics Covered

- The buffer and the transform listener
- Looking up transforms with `lookup_transform`
- Handling `TransformException`
- Time: latest transform vs a specific time
- Transforming points between frames
- A launch file for the complete follow demo
- The C++ equivalents

---

## Buffer and Listener

Two objects work together:

```text
  /tf, /tf_static ──▶ TransformListener ──▶ Buffer ──▶ lookup_transform(...)
                      (subscribes)          (stores ~10 s of history)
```

- The **`TransformListener`** subscribes to `/tf` and `/tf_static`.
- The **`Buffer`** stores the received transforms and answers queries by chaining them through the tree.

```python
from tf2_ros.buffer import Buffer
from tf2_ros.transform_listener import TransformListener

self.tf_buffer = Buffer()
self.tf_listener = TransformListener(self.tf_buffer, self)
```

Create them **once**, in the constructor, and keep them for the life of the node. A freshly created listener has an empty buffer; it needs a moment to fill up.

---

## Looking Up a Transform

```python
t = self.tf_buffer.lookup_transform(target_frame, source_frame, time)
```

The result is a `TransformStamped` that tells you where the **source** frame is, expressed in the **target** frame. With it, data in the source frame can be converted into the target frame.

> 💡 A way to remember the order: `lookup_transform('turtle2', 'turtle1', ...)` answers "where is turtle1, as seen **from** turtle2?". The first argument is your point of view.

`time` selects *when*:

| Value | Meaning |
|-------|---------|
| `rclpy.time.Time()` | The **latest** available transform. The usual choice |
| `rclpy.time.Time.from_msg(msg.header.stamp)` | The transform at the moment a message was captured |

If TF2 cannot answer (the frame doesn't exist yet, the tree is not connected, or the requested time is outside the buffer), `lookup_transform` raises a `TransformException`. **Always** catch it: right after startup, it is completely normal for a transform to be missing for a moment.

---

## The Follower: `turtle_follower`

The plan:

1. Spawn `turtle2` by calling turtlesim's `/spawn` service (as in [Services](../04-python/services.md#calling-a-service-from-a-running-node-turtle_spawner)).
2. Ten times per second, look up where `turtle1` is relative to `turtle2`.
3. Steer `turtle2`: turn toward the target, and drive faster the farther away it is.

Create `~/ros2_ws/src/my_first_pkg/my_first_pkg/turtle_follower.py`:

```python
import math

import rclpy
from geometry_msgs.msg import Twist
from rclpy.node import Node
from tf2_ros import TransformException
from tf2_ros.buffer import Buffer
from tf2_ros.transform_listener import TransformListener
from turtlesim.srv import Spawn


class TurtleFollower(Node):
    def __init__(self):
        super().__init__('turtle_follower')
        self.declare_parameter('target_frame', 'turtle1')

        self.tf_buffer = Buffer()
        self.tf_listener = TransformListener(self.tf_buffer, self)

        self.spawn_client = self.create_client(Spawn, 'spawn')
        self.spawn_requested = False
        self.turtle_spawned = False

        self.publisher = self.create_publisher(Twist, 'turtle2/cmd_vel', 10)
        self.timer = self.create_timer(0.1, self.on_timer)

    def on_timer(self):
        if not self.turtle_spawned:
            self.spawn_turtle()
            return

        target_frame = self.get_parameter('target_frame').value
        try:
            t = self.tf_buffer.lookup_transform('turtle2', target_frame, rclpy.time.Time())
        except TransformException as ex:
            self.get_logger().info(
                f'Could not transform turtle2 to {target_frame}: {ex}',
                throttle_duration_sec=1.0)
            return

        dx = t.transform.translation.x
        dy = t.transform.translation.y

        msg = Twist()
        msg.angular.z = 1.0 * math.atan2(dy, dx)
        msg.linear.x = 0.5 * math.sqrt(dx ** 2 + dy ** 2)
        self.publisher.publish(msg)

    def spawn_turtle(self):
        if self.spawn_requested or not self.spawn_client.service_is_ready():
            return
        request = Spawn.Request()
        request.x = 4.0
        request.y = 2.0
        request.theta = 0.0
        request.name = 'turtle2'
        future = self.spawn_client.call_async(request)
        future.add_done_callback(self.spawn_done_callback)
        self.spawn_requested = True

    def spawn_done_callback(self, future):
        self.turtle_spawned = True
        self.get_logger().info('Spawned turtle2, following now')


def main():
    rclpy.init()
    node = TurtleFollower()
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
t = self.tf_buffer.lookup_transform('turtle2', target_frame, rclpy.time.Time())
```

This gives `turtle1`'s position **in turtle2's own frame**, where x points forward and y points left (see [REP 103](coordinate-frames.md#units-and-axes-rep-103)). That is exactly what a steering controller needs:

```text
            y (left)
            ▲
            │        ● turtle1 at (dx, dy)
            │      ╱
            │    ╱  angle = atan2(dy, dx)
            │  ╱
   turtle2  ●─────────────▶ x (forward)
```

- `atan2(dy, dx)` is the angle to the target: positive means "turn left", negative "turn right".
- `sqrt(dx² + dy²)` is the distance: drive faster when the target is far away, slow down when close.

This is a simple **proportional controller**: the command is proportional to the error. The gains `1.0` and `0.5` decide how aggressively the turtle reacts.

Notice what the follower **doesn't** need to know: the turtles' positions in the world, or any trigonometry about their headings. TF2 did all of that by chaining `world → turtle1` and `world → turtle2`.

### Run it

Register `'turtle_follower = my_first_pkg.turtle_follower:main'` and rebuild. The follower needs a broadcaster for **each** turtle:

```bash
$ ros2 run turtlesim turtlesim_node
$ ros2 run my_first_pkg turtle_tf_broadcaster --ros-args -p turtlename:=turtle1 -r __node:=broadcaster1
$ ros2 run my_first_pkg turtle_tf_broadcaster --ros-args -p turtlename:=turtle2 -r __node:=broadcaster2
$ ros2 run my_first_pkg turtle_follower
$ ros2 run turtlesim turtle_teleop_key
```

Right after starting, the follower logs `Could not transform turtle2 to turtle1: ...` a few times: `turtle2` doesn't exist yet, so neither does its frame. Once it is spawned and broadcast, `turtle2` starts following.

Six terminals is a lot. Time for a launch file.

---

## A Launch File for the Demo

Create `~/ros2_ws/src/my_first_pkg/launch/tf_follow.launch.py`:

```python
from launch import LaunchDescription
from launch_ros.actions import Node


def generate_launch_description():
    return LaunchDescription([
        Node(package='turtlesim', executable='turtlesim_node', name='sim'),
        Node(
            package='my_first_pkg',
            executable='turtle_tf_broadcaster',
            name='broadcaster1',
            parameters=[{'turtlename': 'turtle1'}],
        ),
        Node(
            package='my_first_pkg',
            executable='turtle_tf_broadcaster',
            name='broadcaster2',
            parameters=[{'turtlename': 'turtle2'}],
        ),
        Node(package='my_first_pkg', executable='static_frames'),
        Node(package='my_first_pkg', executable='turtle_follower'),
    ])
```

The launch folder is already installed by the `data_files` entry from [setup.py](../03-workspaces-and-packages/setup-py.md#data_files-installing-non-python-files), but a **new** launch file needs a rebuild. Then:

```bash
$ ros2 launch my_first_pkg tf_follow.launch.py
```

```bash
$ ros2 run turtlesim turtle_teleop_key
```

### Follow a different frame

The target frame is a parameter, so the follower can chase **any** frame in the tree, including the static frames from the [previous lesson](tf2-broadcaster.md#writing-a-static-broadcaster):

```bash
$ ros2 param set /turtle_follower target_frame turtle1_camera
```

`turtle2` now aims for the point 0.3 m in front of `turtle1`. You changed the behavior without changing any code, just by pointing at a different frame.

---

## Transforming Data Between Frames

Looking up a transform is often just the first step. Usually you want to convert **data**: a point, a pose, a velocity. The `tf2_geometry_msgs` package provides functions for `geometry_msgs` types:

```python
from geometry_msgs.msg import PointStamped
from tf2_geometry_msgs import do_transform_point

# A point 1 m in front of turtle1, expressed in turtle1's frame
point = PointStamped()
point.header.frame_id = 'turtle1'
point.point.x = 1.0

try:
    t = self.tf_buffer.lookup_transform('world', 'turtle1', rclpy.time.Time())
    point_in_world = do_transform_point(point, t)
    self.get_logger().info(
        f'Point in world: ({point_in_world.point.x:.2f}, {point_in_world.point.y:.2f})')
except TransformException as ex:
    self.get_logger().warn(f'{ex}')
```

`do_transform_pose`, `do_transform_vector3`, and others work the same way. Add `<depend>tf2_geometry_msgs</depend>` to `package.xml` when you use them.

This is exactly what a robot does with sensor data: a LiDAR point in the `laser` frame is transformed into `map` so the planner can use it.

---

## Time and Transforms

Every transform has a timestamp, and the buffer keeps about 10 seconds of history. Two typical errors relate to time:

| Error message contains | Meaning |
|------------------------|---------|
| `Lookup would require extrapolation into the future` | You asked for a time newer than the latest transform received. Often the transform is simply a few milliseconds behind |
| `Lookup would require extrapolation into the past` | You asked for a time older than the buffer's history |

Using `rclpy.time.Time()` (latest available) avoids both in simple cases. When you transform sensor data, use the message's own timestamp instead, so the data is combined with the transform that was valid when it was captured.

If you need to wait briefly for a transform, `lookup_transform` accepts a timeout:

```python
from rclpy.duration import Duration

t = self.tf_buffer.lookup_transform(
    'map', 'laser', rclpy.time.Time(), timeout=Duration(seconds=0.1))
```

> ⚠️ A timeout **blocks** the callback while waiting, with the same consequences as described in [Avoiding Deadlocks](../04-python/services.md#avoiding-deadlocks). Prefer short timeouts or none at all, and check availability first with `self.tf_buffer.can_transform(...)`.

---

## In C++

| Python | C++ |
|--------|-----|
| `Buffer()` | `std::make_unique<tf2_ros::Buffer>(this->get_clock())` |
| `TransformListener(self.tf_buffer, self)` | `std::make_shared<tf2_ros::TransformListener>(*tf_buffer_)` |
| `lookup_transform('a', 'b', rclpy.time.Time())` | `tf_buffer_->lookupTransform("a", "b", tf2::TimePointZero)` |
| `except TransformException` | `catch (const tf2::TransformException & ex)` |
| `do_transform_point(p, t)` | `tf2::doTransform(p, p_out, t)` (`#include "tf2_geometry_msgs/tf2_geometry_msgs.hpp"`) |

Includes: `tf2_ros/buffer.h`, `tf2_ros/transform_listener.h`, `tf2/exceptions.h`.

---

## Try It Yourself

1. Change the follower's gains. What happens with `linear 2.0`? With `angular 4.0`?
2. Make `turtle2` stop when it is closer than 1 m to its target.
3. Spawn a third turtle that follows `turtle2`, forming a chain. (You need another broadcaster and a follower with remapped topics and frames.)
4. Write a node that logs the distance between `turtle1` and `turtle2` once per second using `lookup_transform('turtle1', 'turtle2', ...)`.
5. Use `do_transform_point` to log the world position of `turtle1_antenna` every second.

---

⬅️ [Previous: TF2 Broadcasters](tf2-broadcaster.md) | [⬆️ Transform Library (TF2)](README.md) | [Next: TF2 Tools](tf2-tools.md) ➡️
