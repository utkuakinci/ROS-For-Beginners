# Xacro

The URDF from the previous lesson works, but it already shows the problems of plain XML: the two wheels are almost identical copies, the same numbers appear in several places, and changing the chassis size means recalculating every joint by hand. **Xacro** (XML Macros) solves this with properties, math, macros, and file includes. Nearly every real robot description is written in Xacro.

## Topics Covered

- Properties and math expressions
- Macros with parameters
- Inertia macros
- Splitting a description into several files
- Arguments and conditional blocks
- Converting Xacro to URDF

---

## How Xacro Works

A `.urdf.xacro` file is a URDF with extra tags. The `xacro` tool **expands** it into plain URDF before anything else reads it:

```text
my_robot.urdf.xacro ──▶ xacro ──▶ URDF (XML) ──▶ robot_state_publisher, RViz2, Gazebo
```

```bash
$ xacro my_robot.urdf.xacro > my_robot.urdf
```

To use Xacro tags, declare the namespace in the `<robot>` tag:

```xml
<?xml version="1.0"?>
<robot name="my_robot" xmlns:xacro="http://www.ros.org/wiki/xacro">
  ...
</robot>
```

---

## Properties and Math

A **property** is a named constant. Use it with `${name}`:

```xml
<xacro:property name="wheel_radius" value="0.05"/>
<xacro:property name="wheel_width" value="0.04"/>

<cylinder radius="${wheel_radius}" length="${wheel_width}"/>
```

Anything inside `${...}` is evaluated as a Python expression, so you can compute values:

```xml
<xacro:property name="base_width" value="0.3"/>
<xacro:property name="wheel_gap" value="0.005"/>

<origin xyz="0 ${base_width / 2 + wheel_width / 2 + wheel_gap} 0" rpy="${-pi / 2} 0 0"/>
```

`pi` is predefined, as are functions like `sin`, `cos`, and `sqrt`. Now every dimension is defined **once**, and positions follow automatically when you change it.

---

## Macros

A **macro** is a reusable block of XML with parameters, like a function:

```xml
<xacro:macro name="wheel" params="prefix y_reflect">
  <link name="${prefix}_wheel">
    <visual>
      <geometry>
        <cylinder radius="${wheel_radius}" length="${wheel_width}"/>
      </geometry>
      <material name="black"/>
    </visual>
  </link>

  <joint name="${prefix}_wheel_joint" type="continuous">
    <parent link="base_link"/>
    <child link="${prefix}_wheel"/>
    <origin xyz="${-wheel_x_offset} ${y_reflect * wheel_y_offset} ${-base_height / 2}"
            rpy="${-pi / 2} 0 0"/>
    <axis xyz="0 0 1"/>
  </joint>
</xacro:macro>

<!-- Use the macro twice -->
<xacro:wheel prefix="left" y_reflect="1"/>
<xacro:wheel prefix="right" y_reflect="-1"/>
```

The `y_reflect` parameter mirrors the wheel to the other side. Two calls replace two nearly identical blocks of 20 lines each, and a fix in the macro fixes both wheels.

---

## Inertia Macros

Every link needs an `<inertial>` for simulation, and computing inertia tensors by hand is tedious and error-prone. For the basic shapes, standard formulas exist, so you write them **once** as macros.

Create `~/ros2_ws/src/my_robot_description/urdf/inertial_macros.xacro`:

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro">

  <!-- Solid box with side lengths x, y, z -->
  <xacro:macro name="box_inertia" params="m x y z">
    <inertial>
      <mass value="${m}"/>
      <inertia ixx="${m * (y * y + z * z) / 12}" ixy="0" ixz="0"
               iyy="${m * (x * x + z * z) / 12}" iyz="0"
               izz="${m * (x * x + y * y) / 12}"/>
    </inertial>
  </xacro:macro>

  <!-- Solid cylinder with radius r and length h, axis along z -->
  <xacro:macro name="cylinder_inertia" params="m r h">
    <inertial>
      <mass value="${m}"/>
      <inertia ixx="${m * (3 * r * r + h * h) / 12}" ixy="0" ixz="0"
               iyy="${m * (3 * r * r + h * h) / 12}" iyz="0"
               izz="${m * r * r / 2}"/>
    </inertial>
  </xacro:macro>

  <!-- Solid sphere with radius r -->
  <xacro:macro name="sphere_inertia" params="m r">
    <inertial>
      <mass value="${m}"/>
      <inertia ixx="${2 * m * r * r / 5}" ixy="0" ixz="0"
               iyy="${2 * m * r * r / 5}" iyz="0"
               izz="${2 * m * r * r / 5}"/>
    </inertial>
  </xacro:macro>

</robot>
```

Using one is a single line inside a link:

```xml
<xacro:box_inertia m="2.0" x="0.4" y="0.3" z="0.1"/>
```

> 📝 These formulas assume the shape is centered on the link's origin, which is true for every link in `my_robot`. If a visual is offset, give the `<inertial>` the same `<origin>`.

---

## Including Other Files

`xacro:include` inserts another Xacro file, which keeps large descriptions organized (one file per part, one for macros, one for simulation settings):

```xml
<xacro:include filename="inertial_macros.xacro"/>
```

A relative path is resolved relative to the including file. To include a file from another package, use `$(find package_name)`:

```xml
<xacro:include filename="$(find my_robot_description)/urdf/inertial_macros.xacro"/>
```

`$(find ...)` looks up the installed package, so it only works after the package has been built and the workspace sourced.

---

## Arguments and Conditions

**Arguments** are values passed in from outside, for example from a launch file. **Conditions** include a block only if a value is true:

```xml
<xacro:arg name="use_laser" default="true"/>

<xacro:if value="$(arg use_laser)">
  <!-- laser link and joint -->
</xacro:if>
```

```bash
$ xacro my_robot.urdf.xacro use_laser:=false
```

There is also `<xacro:unless value="...">` for the opposite case. Arguments let one description cover several robot variants, such as with and without a sensor, or real hardware vs simulation.

---

## The Complete `my_robot.urdf.xacro`

Here is the robot from the [URDF](urdf.md#the-complete-robot) lesson, rewritten with Xacro and with inertia for every link. Save it as `~/ros2_ws/src/my_robot_description/urdf/my_robot.urdf.xacro`:

```xml
<?xml version="1.0"?>
<robot name="my_robot" xmlns:xacro="http://www.ros.org/wiki/xacro">

  <xacro:include filename="inertial_macros.xacro"/>

  <xacro:arg name="use_laser" default="true"/>

  <!-- Dimensions (meters) and masses (kilograms) -->
  <xacro:property name="base_length" value="0.4"/>
  <xacro:property name="base_width" value="0.3"/>
  <xacro:property name="base_height" value="0.1"/>
  <xacro:property name="base_mass" value="2.0"/>

  <xacro:property name="wheel_radius" value="0.05"/>
  <xacro:property name="wheel_width" value="0.04"/>
  <xacro:property name="wheel_mass" value="0.2"/>
  <xacro:property name="wheel_gap" value="0.005"/>
  <xacro:property name="wheel_x_offset" value="0.1"/>
  <xacro:property name="wheel_y_offset" value="${base_width / 2 + wheel_width / 2 + wheel_gap}"/>

  <xacro:property name="caster_mass" value="0.1"/>

  <xacro:property name="laser_radius" value="0.05"/>
  <xacro:property name="laser_length" value="0.04"/>
  <xacro:property name="laser_mass" value="0.1"/>

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

  <!-- Base -->
  <link name="base_footprint"/>

  <link name="base_link">
    <visual>
      <geometry>
        <box size="${base_length} ${base_width} ${base_height}"/>
      </geometry>
      <material name="blue"/>
    </visual>
    <collision>
      <geometry>
        <box size="${base_length} ${base_width} ${base_height}"/>
      </geometry>
    </collision>
    <xacro:box_inertia m="${base_mass}" x="${base_length}" y="${base_width}" z="${base_height}"/>
  </link>

  <joint name="base_joint" type="fixed">
    <parent link="base_footprint"/>
    <child link="base_link"/>
    <!-- The chassis bottom is level with the wheel axles -->
    <origin xyz="0 0 ${wheel_radius + base_height / 2}" rpy="0 0 0"/>
  </joint>

  <!-- Wheels -->
  <xacro:macro name="wheel" params="prefix y_reflect">
    <link name="${prefix}_wheel">
      <visual>
        <geometry>
          <cylinder radius="${wheel_radius}" length="${wheel_width}"/>
        </geometry>
        <material name="black"/>
      </visual>
      <collision>
        <geometry>
          <cylinder radius="${wheel_radius}" length="${wheel_width}"/>
        </geometry>
      </collision>
      <xacro:cylinder_inertia m="${wheel_mass}" r="${wheel_radius}" h="${wheel_width}"/>
    </link>

    <joint name="${prefix}_wheel_joint" type="continuous">
      <parent link="base_link"/>
      <child link="${prefix}_wheel"/>
      <origin xyz="${-wheel_x_offset} ${y_reflect * wheel_y_offset} ${-base_height / 2}"
              rpy="${-pi / 2} 0 0"/>
      <axis xyz="0 0 1"/>
    </joint>
  </xacro:macro>

  <xacro:wheel prefix="left" y_reflect="1"/>
  <xacro:wheel prefix="right" y_reflect="-1"/>

  <!-- Caster: same radius as the wheels so the robot stands level -->
  <link name="caster">
    <visual>
      <geometry>
        <sphere radius="${wheel_radius}"/>
      </geometry>
      <material name="grey"/>
    </visual>
    <collision>
      <geometry>
        <sphere radius="${wheel_radius}"/>
      </geometry>
    </collision>
    <xacro:sphere_inertia m="${caster_mass}" r="${wheel_radius}"/>
  </link>

  <joint name="caster_joint" type="fixed">
    <parent link="base_link"/>
    <child link="caster"/>
    <origin xyz="${base_length / 2 - wheel_radius} 0 ${-base_height / 2}" rpy="0 0 0"/>
  </joint>

  <!-- Laser scanner (optional) -->
  <xacro:if value="$(arg use_laser)">
    <link name="laser">
      <visual>
        <geometry>
          <cylinder radius="${laser_radius}" length="${laser_length}"/>
        </geometry>
        <material name="grey"/>
      </visual>
      <collision>
        <geometry>
          <cylinder radius="${laser_radius}" length="${laser_length}"/>
        </geometry>
      </collision>
      <xacro:cylinder_inertia m="${laser_mass}" r="${laser_radius}" h="${laser_length}"/>
    </link>

    <joint name="laser_joint" type="fixed">
      <parent link="base_link"/>
      <child link="laser"/>
      <origin xyz="0.1 0 ${base_height / 2 + laser_length / 2}" rpy="0 0 0"/>
    </joint>
  </xacro:if>

</robot>
```

Compare it with the plain URDF: every number now has a name, the wheels are defined once, every link has a computed inertia, and the laser can be switched off.

---

## Converting and Checking

```bash
$ cd ~/ros2_ws/src/my_robot_description/urdf
$ xacro my_robot.urdf.xacro > /tmp/my_robot.urdf
$ check_urdf /tmp/my_robot.urdf
robot name is: my_robot
---------- Successfully Parsed XML ---------------
root Link: base_footprint has 1 child(ren)
    child(1):  base_link
    ...
```

Look at `/tmp/my_robot.urdf`: all properties and macros have been replaced by plain values. When the model looks wrong, reading the expanded URDF is often the fastest way to find the mistake.

View it with the same commands as in the [URDF](urdf.md#viewing-the-robot) lesson, replacing `cat` with `xacro`:

```bash
$ ros2 run robot_state_publisher robot_state_publisher \
    --ros-args -p robot_description:="$(xacro my_robot.urdf.xacro)"
```

It should look exactly like before. You no longer need `my_robot.urdf`; from now on the Xacro file is the single source of truth.

---

## Common Mistakes

| Symptom | Cause |
|---------|-------|
| Xacro tags appear unchanged in the output, or `Unknown tag` errors | Missing `xmlns:xacro="http://www.ros.org/wiki/xacro"` in `<robot>` |
| `name 'wheel_raduis' is not defined` | Typo in a property name inside `${...}` |
| `macro 'xacro:wheel' is not defined` or similar | The macro is defined after it is used, or the include is missing |
| `Include tag's filename spec "..." matched no files` | Wrong include path; relative paths are relative to the including file |
| `$(find my_robot_description)` fails | The package hasn't been built and sourced yet |
| Wrong values in the robot | Check the expanded URDF with `xacro file.urdf.xacro` |

---

## Try It Yourself

1. Change `base_length` to 0.5 and `wheel_radius` to 0.07. Check in RViz2 that everything still fits together.
2. Move the materials into a separate `materials.xacro` and include it.
3. Write a macro `caster` with a parameter `x` and use it to add a second caster at the back.
4. Add an argument `use_camera` (default `false`) and a camera link inside an `xacro:if` block.
5. Expand the Xacro file with `use_laser:=false` and confirm with `check_urdf` that the `laser` link is gone.

---

⬅️ [Previous: URDF](urdf.md) | [⬆️ Robot Modeling](README.md) | [Next: robot_state_publisher](robot-state-publisher.md) ➡️
