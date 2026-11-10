# URDF

The **Unified Robot Description Format (URDF)** is an XML format that describes a robot as a tree of rigid parts (**links**) connected by **joints**. In this lesson you write the URDF of `my_robot`, a small two-wheeled robot, piece by piece, check it, and view it in RViz2.

## Topics Covered

- Links and joints
- Visual, collision, and inertial properties
- Joint types
- Origins, axes, and the link tree
- Building a complete mobile robot
- Checking and viewing a URDF

---

## Links and Joints

A URDF describes two things:

| Element | What it is | Becomes in TF |
|---------|------------|---------------|
| `<link>` | A rigid body: the chassis, a wheel, a sensor | A **frame** |
| `<joint>` | A connection between a parent link and a child link, and how it can move | A **transform** between their frames |

Like the TF tree in [Coordinate Frames](../06-tf2/coordinate-frames.md#the-transform-tree), the links form a **tree**: one root link, and every other link has exactly one parent joint.

```xml
<?xml version="1.0"?>
<robot name="my_robot">
  <link name="base_link"/>
  <link name="laser"/>

  <joint name="laser_joint" type="fixed">
    <parent link="base_link"/>
    <child link="laser"/>
    <origin xyz="0.1 0 0.07" rpy="0 0 0"/>
  </joint>
</robot>
```

This tiny URDF is already valid: two frames, with `laser` fixed 10 cm in front of and 7 cm above `base_link`.

---

## Links in Detail

A link can have three kinds of properties, each optional:

```xml
<link name="base_link">
  <visual>      <!-- what it looks like -->
  </visual>
  <collision>   <!-- the shape used for collision checking -->
  </collision>
  <inertial>    <!-- mass and inertia, used by physics simulation -->
  </inertial>
</link>
```

### Visual

```xml
<visual>
  <origin xyz="0 0 0" rpy="0 0 0"/>
  <geometry>
    <box size="0.4 0.3 0.1"/>
  </geometry>
  <material name="blue">
    <color rgba="0.1 0.3 0.8 1.0"/>
  </material>
</visual>
```

| Geometry | Parameters |
|----------|------------|
| `<box size="x y z"/>` | Side lengths, centered on the origin |
| `<cylinder radius="r" length="l"/>` | Axis along **z**, centered on the origin |
| `<sphere radius="r"/>` | Centered on the origin |
| `<mesh filename="package://pkg/meshes/part.stl" scale="1 1 1"/>` | A 3D model file (STL, DAE) |

The `<origin>` inside `<visual>` shifts the shape relative to the link's frame. `<material>` sets a color: RGBA values from 0 to 1. A material defined once by name can be reused by other links.

### Collision

```xml
<collision>
  <geometry>
    <box size="0.4 0.3 0.1"/>
  </geometry>
</collision>
```

The collision shape is used by simulators and motion planners. It is often the same as the visual shape but should be **simple**: when the visual is a detailed mesh, use a box or cylinder for collision, since collision checks with meshes are slow.

### Inertial

```xml
<inertial>
  <mass value="2.0"/>
  <inertia ixx="0.0167" ixy="0" ixz="0" iyy="0.0283" iyz="0" izz="0.0417"/>
</inertial>
```

Mass (kg) and the inertia tensor describe how the link responds to forces. RViz2 ignores them, but a physics simulator like Gazebo **requires** them; a link without inertia is ignored or behaves strangely. You will compute the values with formulas in the [Xacro](xacro.md#inertia-macros) lesson.

---

## Joints in Detail

```xml
<joint name="left_wheel_joint" type="continuous">
  <parent link="base_link"/>
  <child link="left_wheel"/>
  <origin xyz="-0.1 0.175 -0.05" rpy="-1.5708 0 0"/>
  <axis xyz="0 0 1"/>
</joint>
```

| Element | Meaning |
|---------|---------|
| `type` | How the joint moves (see below) |
| `<parent>`, `<child>` | The two links it connects |
| `<origin>` | Where the child frame is, relative to the parent frame, when the joint is at zero |
| `<axis>` | The axis of motion, expressed **in the child frame** |
| `<limit>` | Position, velocity, and effort limits (required for `revolute` and `prismatic`) |

### Joint types

| Type | Motion | Example |
|------|--------|---------|
| `fixed` | None | A sensor mounted on the chassis |
| `continuous` | Rotation without limits | A wheel |
| `revolute` | Rotation between limits | A robot arm joint |
| `prismatic` | Sliding along an axis, between limits | A linear actuator, a lift |
| `floating`, `planar` | 6 or 3 degrees of freedom | Rarely used |

A `revolute` joint needs limits:

```xml
<joint name="arm_joint" type="revolute">
  ...
  <axis xyz="0 1 0"/>
  <limit lower="-1.57" upper="1.57" velocity="1.0" effort="10.0"/>
</joint>
```

`lower` / `upper` are in radians (meters for `prismatic`), `velocity` in rad/s (m/s), `effort` in N·m (N).

### Origins and rotations

The joint's `<origin>` places the child frame. `xyz` is the translation in meters, `rpy` is roll, pitch, yaw in radians, using the conventions from [Coordinate Frames](../06-tf2/coordinate-frames.md#ros-conventions).

Wheels show why rotation matters. A `<cylinder>` is aligned with its link's **z** axis, but a wheel must spin around the robot's **y** axis (left-right). Rotating the wheel frame by `rpy="-1.5708 0 0"` (−90° roll) turns its z axis to point left, along the robot's y axis. The `<axis xyz="0 0 1"/>` then makes the wheel spin around that axis, and a positive rotation rolls the robot forward.

---

## The Complete Robot

Here is `my_robot`, a two-wheeled robot with a caster and a laser scanner. Save it as `~/ros2_ws/src/my_robot_description/urdf/my_robot.urdf`:

```xml
<?xml version="1.0"?>
<robot name="my_robot">

  <!-- Materials -->
  <material name="blue">
    <color rgba="0.1 0.3 0.8 1.0"/>
  </material>
  <material name="black">
    <color rgba="0.1 0.1 0.1 1.0"/>
  </material>
  <material name="grey">
    <color rgba="0.6 0.6 0.6 1.0"/>
  </material>

  <!-- Ground projection of the robot's center (REP 105) -->
  <link name="base_footprint"/>

  <!-- Chassis -->
  <link name="base_link">
    <visual>
      <geometry>
        <box size="0.4 0.3 0.1"/>
      </geometry>
      <material name="blue"/>
    </visual>
    <collision>
      <geometry>
        <box size="0.4 0.3 0.1"/>
      </geometry>
    </collision>
    <inertial>
      <mass value="2.0"/>
      <inertia ixx="0.0167" ixy="0" ixz="0" iyy="0.0283" iyz="0" izz="0.0417"/>
    </inertial>
  </link>

  <joint name="base_joint" type="fixed">
    <parent link="base_footprint"/>
    <child link="base_link"/>
    <origin xyz="0 0 0.1" rpy="0 0 0"/>
  </joint>

  <!-- Left wheel -->
  <link name="left_wheel">
    <visual>
      <geometry>
        <cylinder radius="0.05" length="0.04"/>
      </geometry>
      <material name="black"/>
    </visual>
    <collision>
      <geometry>
        <cylinder radius="0.05" length="0.04"/>
      </geometry>
    </collision>
  </link>

  <joint name="left_wheel_joint" type="continuous">
    <parent link="base_link"/>
    <child link="left_wheel"/>
    <origin xyz="-0.1 0.175 -0.05" rpy="-1.5708 0 0"/>
    <axis xyz="0 0 1"/>
  </joint>

  <!-- Right wheel -->
  <link name="right_wheel">
    <visual>
      <geometry>
        <cylinder radius="0.05" length="0.04"/>
      </geometry>
      <material name="black"/>
    </visual>
    <collision>
      <geometry>
        <cylinder radius="0.05" length="0.04"/>
      </geometry>
    </collision>
  </link>

  <joint name="right_wheel_joint" type="continuous">
    <parent link="base_link"/>
    <child link="right_wheel"/>
    <origin xyz="-0.1 -0.175 -0.05" rpy="-1.5708 0 0"/>
    <axis xyz="0 0 1"/>
  </joint>

  <!-- Caster ball at the front -->
  <link name="caster">
    <visual>
      <geometry>
        <sphere radius="0.05"/>
      </geometry>
      <material name="grey"/>
    </visual>
    <collision>
      <geometry>
        <sphere radius="0.05"/>
      </geometry>
    </collision>
  </link>

  <joint name="caster_joint" type="fixed">
    <parent link="base_link"/>
    <child link="caster"/>
    <origin xyz="0.15 0 -0.05" rpy="0 0 0"/>
  </joint>

  <!-- Laser scanner on top -->
  <link name="laser">
    <visual>
      <geometry>
        <cylinder radius="0.05" length="0.04"/>
      </geometry>
      <material name="grey"/>
    </visual>
    <collision>
      <geometry>
        <cylinder radius="0.05" length="0.04"/>
      </geometry>
    </collision>
  </link>

  <joint name="laser_joint" type="fixed">
    <parent link="base_link"/>
    <child link="laser"/>
    <origin xyz="0.1 0 0.07" rpy="0 0 0"/>
  </joint>

</robot>
```

How the numbers fit together:

- The wheels have a radius of 5 cm, so their centers are 0.05 m above the ground.
- `base_link` (the chassis center) is 0.1 m above `base_footprint`, so the wheel joints are at `z = -0.05` relative to it.
- The wheels sit at `y = ±0.175`: half the chassis width (0.15) plus half the wheel width (0.02), plus a small gap.
- The caster is a sphere of radius 0.05 whose center is also 0.05 m above the ground, so the robot stands level.
- The laser sits on top: half the chassis height (0.05) plus half the laser height (0.02) = 0.07.

Only `base_link` has an `<inertial>` for now; the [Xacro](xacro.md) lesson adds the rest without the tedious math.

---

## Checking a URDF

`check_urdf` (from `liburdfdom-tools`) parses the file and prints the link tree:

```bash
$ cd ~/ros2_ws/src/my_robot_description/urdf
$ check_urdf my_robot.urdf
robot name is: my_robot
---------- Successfully Parsed XML ---------------
root Link: base_footprint has 1 child(ren)
    child(1):  base_link
        child(1):  caster
        child(2):  laser
        child(3):  left_wheel
        child(4):  right_wheel
```

Syntax errors, unknown links in joints, or a link with two parents are reported here with a clear message, much more readably than in RViz2.

---

## Viewing the Robot

The proper launch file comes in the [robot_state_publisher](robot-state-publisher.md) lesson. For a quick look, start the three required nodes by hand, each in its own terminal:

```bash
$ cd ~/ros2_ws/src/my_robot_description/urdf
$ ros2 run robot_state_publisher robot_state_publisher --ros-args -p robot_description:="$(cat my_robot.urdf)"
```

```bash
$ ros2 run joint_state_publisher_gui joint_state_publisher_gui
```

```bash
$ rviz2
```

In RViz2:

1. Set **Fixed Frame** to `base_footprint`.
2. Click **Add** → **RobotModel**, and set its **Description Topic** to `/robot_description`.
3. Click **Add** → **TF** to see the frames.

The robot appears. The **Joint State Publisher** window has a slider for each wheel; move them and watch the wheels turn. Fixed joints have no sliders.

---

## Common Mistakes

| Symptom | Cause |
|---------|-------|
| `check_urdf` reports `Error parsing XML` | A tag is not closed, or quotes are missing |
| `link 'x' is not unique` / joint references unknown link | Duplicate link name, or a typo in `<parent>` / `<child>` |
| `Failed to find root link: Two root links found` | A link is not connected to the tree by any joint |
| Wheels stand upright like coins | Missing `rpy` rotation on the wheel joints |
| The robot model is red / white in RViz2 with `No transform` | `robot_state_publisher` or `joint_state_publisher` is not running |
| Shapes floating in the wrong place | `<origin>` in the joint vs. in `<visual>` confused, or degrees used instead of radians |

---

## Try It Yourself

1. Change the chassis color and size, and adjust the wheel and laser positions so everything still fits.
2. Add a `camera_link`: a small box (0.03 × 0.1 × 0.03) at the front of the chassis, fixed to `base_link`.
3. Turn the laser joint into a `revolute` joint around z with limits ±1.57 rad. Move it with the slider.
4. Add a second caster at the back of the robot. Does `check_urdf` still show a valid tree?
5. Deliberately give the laser link two parent joints and read the `check_urdf` error.

---

[⬆️ Robot Modeling](README.md) | [Next: Xacro](xacro.md) ➡️
