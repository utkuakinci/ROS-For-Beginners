# Coordinate Frames

Before using TF2, you need a clear picture of what a coordinate frame is, what a transform between two frames contains, and which frames almost every ROS robot has. This lesson covers those ideas and shows them in action with a ready-made TF2 demo.

## Topics Covered

- What a coordinate frame is and why robots need many
- Transforms: translation and rotation
- Representing rotations: Euler angles and quaternions
- The transform tree
- ROS conventions for axes, units, and standard frames
- A first TF2 demo with turtlesim

---

## Why Frames?

Imagine a mobile robot with a LiDAR mounted 20 cm in front of its center. The LiDAR detects an obstacle **1.5 m straight ahead of the sensor**. Where is the obstacle relative to the robot's center? Where is it on the map?

```text
       map
        │      the robot is at (4, 2) on the map, facing 90°
        ▼
   ┌─────────┐
   │base_link│──── 0.2 m ───▶ laser ──── 1.5 m ───▶ ● obstacle
   └─────────┘
```

Each question needs a different **frame of reference**. Every sensor measures in its own frame, every actuator acts in its own frame, and the planner thinks in map coordinates. Converting between them by hand, for every message, while the robot moves, would be error-prone and repetitive. TF2 does it for you.

---

## Frames and Transforms

A **coordinate frame** is an origin plus three axes (x, y, z). Every frame has a name, such as `map`, `base_link`, or `laser`.

A **transform** describes where one frame is relative to another. It has two parts:

| Part | Meaning | Represented as |
|------|---------|----------------|
| **Translation** | How far the child frame's origin is from the parent's, along x, y, z | 3 numbers, in meters |
| **Rotation** | How the child frame's axes are turned relative to the parent's | A quaternion (4 numbers) |

In ROS 2, a transform is a `geometry_msgs/msg/TransformStamped`:

```bash
$ ros2 interface show geometry_msgs/msg/TransformStamped
std_msgs/Header header          # header.frame_id = the PARENT frame, header.stamp = time
string child_frame_id           # the CHILD frame
Transform transform
        Vector3 translation
                float64 x
                float64 y
                float64 z
        Quaternion rotation
                float64 x 0
                float64 y 0
                float64 z 0
                float64 w 1
```

> 💡 Read a transform as: "the **child** frame is located at this position and orientation, **expressed in the parent** frame". With it, TF2 can convert any point from the child frame into the parent frame and back.

---

## Representing Rotations

### Euler angles: roll, pitch, yaw

The most intuitive way to describe an orientation is with three rotations:

| Angle | Rotation around | Example |
|-------|-----------------|---------|
| **Roll** | x axis (forward) | A car tilting sideways in a curve |
| **Pitch** | y axis (left) | A car nose-up on a ramp |
| **Yaw** | z axis (up) | A car turning left or right |

Angles are in **radians**. A ground robot usually only changes its yaw.

Euler angles are easy for humans but have mathematical problems (for example *gimbal lock*, where two axes line up and a degree of freedom is lost) and are awkward to combine.

### Quaternions

ROS therefore stores rotations as **quaternions**: four numbers `(x, y, z, w)`. You don't need to understand the math to use them. Remember these facts:

- The **identity** (no rotation) is `x=0, y=0, z=0, w=1`. A quaternion of all zeros is **invalid**.
- A valid quaternion has length 1: `x² + y² + z² + w² = 1`.
- For a rotation of `yaw` around the z axis only: `z = sin(yaw / 2)`, `w = cos(yaw / 2)`, `x = y = 0`.

In code, convert from Euler angles instead of writing quaternions by hand. This small function is all you need, and the following lessons use it:

```python
import math


def quaternion_from_euler(roll, pitch, yaw):
    """Convert Euler angles (radians) to a quaternion (x, y, z, w)."""
    cr, sr = math.cos(roll / 2), math.sin(roll / 2)
    cp, sp = math.cos(pitch / 2), math.sin(pitch / 2)
    cy, sy = math.cos(yaw / 2), math.sin(yaw / 2)
    return (
        sr * cp * cy - cr * sp * sy,
        cr * sp * cy + sr * cp * sy,
        cr * cp * sy - sr * sp * cy,
        cr * cp * cy + sr * sp * sy,
    )


print(quaternion_from_euler(0.0, 0.0, math.pi / 2))   # 90° yaw → (0.0, 0.0, 0.707, 0.707)
```

> 📝 The `tf_transformations` package (`sudo apt install ros-jazzy-tf-transformations`) provides `quaternion_from_euler`, `euler_from_quaternion`, and many more helpers if you need them. In C++, use `tf2::Quaternion` and its `setRPY()` method.

---

## The Transform Tree

TF2 organizes all frames in a **tree**: every frame has exactly **one parent** (except the root), and there are no loops. A typical mobile robot:

```text
map
 └── odom
      └── base_footprint
           └── base_link
                ├── left_wheel_link
                ├── right_wheel_link
                ├── laser
                └── camera_link
                     └── camera_optical_frame
```

Because it is a tree, TF2 can find the transform between **any** two frames by walking up from one to their common ancestor and down to the other, chaining the transforms along the way. You never need to publish `laser → map` directly: TF2 computes it from `laser → base_link → base_footprint → odom → map`.

Transforms in the tree come in two kinds:

| Kind | Changes over time? | Examples | Published on |
|------|--------------------|----------|--------------|
| **Static** | No | Where a sensor is mounted on the robot | `/tf_static` |
| **Dynamic** | Yes | The robot's position in the world, a rotating wheel | `/tf` |

TF2 also keeps a short **history** of every transform (10 seconds by default). This lets you ask "where was the laser relative to the map *at the moment this scan was taken*?", which matters when the robot is moving.

---

## ROS Conventions

ROS defines conventions that every package follows, so data fits together.

### Units and axes ([REP 103](https://www.ros.org/reps/rep-0103.html))

- Distances in **meters**, angles in **radians**, time in **seconds**.
- Right-handed coordinate frames. For a robot body: **x forward, y left, z up**.
- Rotations are positive **counter-clockwise** when looking down the axis toward the origin. A positive yaw turns the robot to the **left**.

> 💡 The right-hand rule: point your right thumb along x, index finger along y, and your middle finger shows z.

Camera *optical* frames are the one common exception: z forward (into the image), x right, y down. That is why cameras often have both a `camera_link` and a `camera_optical_frame`.

### Standard frames ([REP 105](https://www.ros.org/reps/rep-0105.html))

| Frame | Meaning | Who publishes `parent → frame` |
|-------|---------|-------------------------------|
| `map` | A fixed world frame. Positions are globally correct but may jump when localization corrects itself | — (root) |
| `odom` | A world frame that moves smoothly but drifts over time | Localization (`map → odom`) |
| `base_footprint` | The robot's center projected onto the ground | Odometry (`odom → base_footprint`) |
| `base_link` | Rigidly attached to the robot body | The robot description (`base_footprint → base_link`) |

You will meet these frames again in [Robot Modeling](../07-robot-modeling/README.md) and [Navigation](../09-navigation/README.md).

---

## A First TF2 Demo

The `turtle_tf2_py` package contains a demo in which a second turtle follows the first using TF2:

```bash
$ ros2 launch turtle_tf2_py turtle_tf2_demo.launch.py
```

In a second terminal, drive `turtle1`:

```bash
$ ros2 run turtlesim turtle_teleop_key
```

`turtle2` follows wherever you go. Behind the scenes:

1. A **broadcaster** node converts each turtle's pose into a transform: `world → turtle1` and `world → turtle2`.
2. A **listener** node asks TF2 "where is `turtle1` relative to `turtle2`?" and steers `turtle2` toward it.

In a third terminal, see the transform the listener uses:

```bash
$ ros2 run tf2_ros tf2_echo turtle2 turtle1
At time 1731240000.123456789
- Translation: [2.104, -0.315, 0.000]
- Rotation: in Quaternion [0.000, 0.000, 0.588, 0.809]
...
```

The translation is turtle1's position as seen from turtle2: 2.1 m ahead and 0.3 m to the right. You will write both the broadcaster and the listener yourself in the next two lessons.

---

## Try It Yourself

1. Using the right-hand rule, determine the direction of z when x points east and y points north.
2. Use `quaternion_from_euler` to compute the quaternions for a yaw of 0°, 90°, 180°, and -90°. Check that `z² + w² = 1` for each.
3. Draw the transform tree of a robot arm on a mobile base with a camera on the gripper. Which transforms are static and which are dynamic?
4. In the demo, stop driving and run `tf2_echo turtle2 turtle1` again. How do the values change as turtle2 catches up?
5. Run `tf2_echo world turtle1` and compare the translation with `ros2 topic echo /turtle1/pose --once`.

---

[⬆️ Transform Library (TF2)](README.md) | [Next: TF2 Broadcasters](tf2-broadcaster.md) ➡️
