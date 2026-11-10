# TF2 Broadcasters

A **broadcaster** publishes transforms into the TF tree. In this lesson you publish static transforms from the command line, from code, and from a launch file, and then write a dynamic broadcaster that turns each turtle's pose into a transform.

## Topics Covered

- Static transforms with `static_transform_publisher`
- Writing a static broadcaster
- Static transforms in launch files
- Writing a dynamic broadcaster
- Rules for a healthy TF tree
- The C++ equivalents

---

## Static Transforms from the Command Line

A static transform never changes, like a sensor bolted to the robot. The `static_transform_publisher` tool from `tf2_ros` publishes one without writing any code:

```bash
$ ros2 run tf2_ros static_transform_publisher \
    --x 0.2 --y 0 --z 0.1 --yaw 0 --pitch 0 --roll 0 \
    --frame-id base_link --child-frame-id laser
[INFO] [static_transform_publisher_...]: Spinning until stopped - publishing transform
translation: ('0.200000', '0.000000', '0.100000')
rotation: ('0.000000', '0.000000', '0.000000', '1.000000')
from 'base_link' to 'laser'
```

| Argument | Meaning |
|----------|---------|
| `--x`, `--y`, `--z` | Translation in meters |
| `--roll`, `--pitch`, `--yaw` | Rotation in radians (or use `--qx --qy --qz --qw` for a quaternion) |
| `--frame-id` | The parent frame |
| `--child-frame-id` | The child frame |

Check it in another terminal:

```bash
$ ros2 run tf2_ros tf2_echo base_link laser
At time 0.0
- Translation: [0.200, 0.000, 0.100]
- Rotation: in Quaternion [0.000, 0.000, 0.000, 1.000]
...
```

`At time 0.0` is how static transforms appear: they are valid at all times.

Static transforms are published once on the `/tf_static` topic with **transient local** durability, so nodes that start later still receive them (see [QoS](../01-ros2-basics/topics.md#quality-of-service-qos-in-brief)).

---

## Writing a Static Broadcaster

Static frames often belong to a node, for example a driver that knows where its sensor is mounted. Let's attach two "sensor" frames to `turtle1`: a camera at the front and an antenna on top.

Create `~/ros2_ws/src/my_first_pkg/my_first_pkg/static_frames.py`:

```python
import math

import rclpy
from geometry_msgs.msg import TransformStamped
from rclpy.node import Node
from tf2_ros.static_transform_broadcaster import StaticTransformBroadcaster


def quaternion_from_euler(roll, pitch, yaw):
    cr, sr = math.cos(roll / 2), math.sin(roll / 2)
    cp, sp = math.cos(pitch / 2), math.sin(pitch / 2)
    cy, sy = math.cos(yaw / 2), math.sin(yaw / 2)
    return (
        sr * cp * cy - cr * sp * sy,
        cr * sp * cy + sr * cp * sy,
        cr * cp * sy - sr * sp * cy,
        cr * cp * cy + sr * sp * sy,
    )


class StaticFrames(Node):
    def __init__(self):
        super().__init__('static_frames')
        self.broadcaster = StaticTransformBroadcaster(self)
        self.broadcaster.sendTransform([
            self.make_transform('turtle1', 'turtle1_camera', 0.3, 0.0, 0.1, yaw=0.0),
            self.make_transform('turtle1', 'turtle1_antenna', -0.2, 0.0, 0.3, yaw=math.pi),
        ])
        self.get_logger().info('Published static frames for turtle1')

    def make_transform(self, parent, child, x, y, z, roll=0.0, pitch=0.0, yaw=0.0):
        t = TransformStamped()
        t.header.stamp = self.get_clock().now().to_msg()
        t.header.frame_id = parent
        t.child_frame_id = child
        t.transform.translation.x = x
        t.transform.translation.y = y
        t.transform.translation.z = z
        qx, qy, qz, qw = quaternion_from_euler(roll, pitch, yaw)
        t.transform.rotation.x = qx
        t.transform.rotation.y = qy
        t.transform.rotation.z = qz
        t.transform.rotation.w = qw
        return t


def main():
    rclpy.init()
    node = StaticFrames()
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

- **`StaticTransformBroadcaster(self)`** creates a publisher on `/tf_static` with the right QoS.
- **`TransformStamped`** holds one transform: the parent in `header.frame_id`, the child in `child_frame_id`, a timestamp, translation, and rotation.
- **`sendTransform([...])`** publishes a list of transforms. For static transforms, you send them **once**; there is no timer.
- **`self.get_clock().now().to_msg()`** converts the node's current time to the message format.

Register `'static_frames = my_first_pkg.static_frames:main'`, add `<depend>tf2_ros</depend>` to `package.xml`, and rebuild. You will see these frames attached to the moving turtle once the dynamic broadcaster below is running.

---

## Static Transforms in Launch Files

In practice, static transforms are usually started from a launch file:

```python
from launch_ros.actions import Node

Node(
    package='tf2_ros',
    executable='static_transform_publisher',
    arguments=[
        '--x', '0.2', '--y', '0', '--z', '0.1',
        '--roll', '0', '--pitch', '0', '--yaw', '0',
        '--frame-id', 'base_link', '--child-frame-id', 'laser',
    ],
),
```

For a real robot, though, you will rarely write static transforms one by one. They are described in the robot's URDF model, and `robot_state_publisher` publishes all of them for you (see [Module 07](../07-robot-modeling/robot-state-publisher.md)).

---

## Writing a Dynamic Broadcaster: `turtle_tf_broadcaster`

A dynamic transform changes over time and must be republished whenever it changes. Let's broadcast each turtle's pose as the transform `world → <turtle name>`.

Create `my_first_pkg/turtle_tf_broadcaster.py`:

```python
import math

import rclpy
from geometry_msgs.msg import TransformStamped
from rclpy.node import Node
from tf2_ros import TransformBroadcaster
from turtlesim.msg import Pose


class TurtleTfBroadcaster(Node):
    def __init__(self):
        super().__init__('turtle_tf_broadcaster')
        self.declare_parameter('turtlename', 'turtle1')
        self.turtlename = self.get_parameter('turtlename').value

        self.broadcaster = TransformBroadcaster(self)
        self.subscription = self.create_subscription(
            Pose, f'/{self.turtlename}/pose', self.pose_callback, 10)

    def pose_callback(self, msg):
        t = TransformStamped()
        t.header.stamp = self.get_clock().now().to_msg()
        t.header.frame_id = 'world'
        t.child_frame_id = self.turtlename

        # Turtlesim is 2D: the turtle moves in x/y and only rotates around z.
        t.transform.translation.x = msg.x
        t.transform.translation.y = msg.y
        t.transform.translation.z = 0.0
        t.transform.rotation.z = math.sin(msg.theta / 2.0)
        t.transform.rotation.w = math.cos(msg.theta / 2.0)

        self.broadcaster.sendTransform(t)


def main():
    rclpy.init()
    node = TurtleTfBroadcaster()
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

### What is different from the static broadcaster

- **`TransformBroadcaster`** publishes on `/tf` instead of `/tf_static`.
- The transform is sent **every time the pose changes**, here from the pose subscription callback. Dynamic transforms are typically published at 10–100 Hz.
- The **timestamp** matters: it says when the pose was valid. Listeners use it to look up transforms at the right moment.
- For a pure yaw rotation, the quaternion simplifies to `z = sin(θ/2)`, `w = cos(θ/2)` (see [Coordinate Frames](coordinate-frames.md#quaternions)).
- The node name and the turtle are configurable, so the same executable serves every turtle.

### Run it

Register `'turtle_tf_broadcaster = my_first_pkg.turtle_tf_broadcaster:main'`, add `<depend>turtlesim</depend>` and `<depend>geometry_msgs</depend>` to `package.xml` if they are not there yet, and rebuild.

```bash
$ ros2 run turtlesim turtlesim_node                                        # Terminal 1
$ ros2 run turtlesim turtle_teleop_key                                     # Terminal 2
$ ros2 run my_first_pkg turtle_tf_broadcaster --ros-args -p turtlename:=turtle1   # Terminal 3
$ ros2 run my_first_pkg static_frames                                      # Terminal 4
```

```bash
$ ros2 run tf2_ros tf2_echo world turtle1_camera                           # Terminal 5
```

Drive the turtle and watch the camera frame's position change, always 0.3 m ahead of the turtle in its direction of travel. TF2 combines the dynamic `world → turtle1` with the static `turtle1 → turtle1_camera` for you.

---

## Rules for a Healthy TF Tree

| Rule | Why |
|------|-----|
| Each frame has **exactly one parent** | Two nodes publishing different parents for the same child break the tree |
| Each transform has **exactly one publisher** | Two broadcasters for the same parent/child pair produce jumping, inconsistent data |
| No loops | TF2 must find a unique path between any two frames |
| Use static broadcasters for fixed transforms | Avoids flooding `/tf` and works for late-joining nodes |
| Stamp dynamic transforms with the time the data was valid | Listeners can then combine data from different times correctly |
| Frame names without a leading slash | `base_link`, not `/base_link` (ROS 2 convention) |

---

## In C++

| Python | C++ |
|--------|-----|
| `from tf2_ros import TransformBroadcaster` | `#include "tf2_ros/transform_broadcaster.h"` |
| `TransformBroadcaster(self)` | `std::make_unique<tf2_ros::TransformBroadcaster>(*this)` |
| `StaticTransformBroadcaster(self)` | `std::make_shared<tf2_ros::StaticTransformBroadcaster>(this)` |
| `broadcaster.sendTransform(t)` | `broadcaster_->sendTransform(t);` |
| `quaternion_from_euler(r, p, y)` | `tf2::Quaternion q; q.setRPY(r, p, y);` (`#include "tf2/LinearMath/Quaternion.h"`) |

Add `tf2` and `tf2_ros` to `find_package`, `ament_target_dependencies`, and `package.xml`.

> 📝 Header names in `tf2_ros` changed from `.h` to `.hpp` in recent distributions. If `#include "tf2_ros/transform_broadcaster.h"` produces a deprecation warning, use `.hpp` instead.

---

## Try It Yourself

1. Publish a static transform `world → tree` at (8, 8) from the command line and check it with `tf2_echo`.
2. Add a third static frame to `static_frames`: a sensor on the turtle's left side, rotated 90° to the left (`yaw = π/2`).
3. Spawn `turtle2` and run a second `turtle_tf_broadcaster` for it with `-p turtlename:=turtle2 -r __node:=turtle2_broadcaster`.
4. Start two broadcasters for `turtle1` at the same time. What do you observe with `tf2_echo`? Which rule did you break?
5. Write a launch file that starts turtlesim, the teleop node (in its own terminal, see [ros2 launch](../02-ros2-cli/ros2-launch.md#troubleshooting)), the turtle broadcaster, and `static_frames`.

---

⬅️ [Previous: Coordinate Frames](coordinate-frames.md) | [⬆️ Transform Library (TF2)](README.md) | [Next: TF2 Listeners](tf2-listener.md) ➡️
