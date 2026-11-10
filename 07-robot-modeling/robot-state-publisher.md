# robot_state_publisher

A robot model on its own is just a file. **robot_state_publisher** brings it to life: it reads the model, combines it with the current joint positions, and publishes the complete TF tree of the robot. In this lesson you learn how it works and write the launch file that displays `my_robot` with one command.

## Topics Covered

- What `robot_state_publisher` does
- Joint states and `/joint_states`
- `joint_state_publisher` and its GUI
- The `/robot_description` topic
- A launch file that loads a Xacro model
- Building and running the description package

---

## How It Works

```text
                          robot_description (URDF)
                                   │
                                   ▼
/joint_states ──────────▶ robot_state_publisher ──────▶ /tf          (moving joints)
(joint positions)                  │              └───▶ /tf_static   (fixed joints)
                                   └──────────────────▶ /robot_description
```

| Input / output | Content |
|----------------|---------|
| `robot_description` **parameter** | The URDF as a string. Required |
| `/joint_states` **subscription** | Current position of every movable joint (`sensor_msgs/msg/JointState`) |
| `/tf_static` | One transform per **fixed** joint, published once |
| `/tf` | One transform per **movable** joint, computed from the joint position and republished whenever joint states arrive |
| `/robot_description` | The URDF, republished as a topic (transient local) for RViz2 and other tools |

This is the [TF2 broadcaster](../06-tf2/tf2-broadcaster.md) from the previous module, generalized: instead of you writing one broadcaster per frame, `robot_state_publisher` derives every transform from the model.

> 📝 `robot_state_publisher` only publishes transforms **between the robot's links**. Where the robot is in the world (`odom → base_footprint`) comes from odometry, and `map → odom` from localization (see [Coordinate Frames](../06-tf2/coordinate-frames.md#standard-frames-rep-105)).

---

## Joint States

To compute the transform of a movable joint, `robot_state_publisher` needs its current position. It gets it from `/joint_states`:

```bash
$ ros2 interface show sensor_msgs/msg/JointState
std_msgs/Header header
string[] name
float64[] position
float64[] velocity
float64[] effort
```

The arrays are parallel: `position[i]` belongs to the joint `name[i]`. Positions are in radians for revolute and continuous joints, and meters for prismatic joints.

Who publishes `/joint_states`?

| Situation | Publisher |
|-----------|-----------|
| Real robot | The hardware driver or `ros2_control` reads the motor encoders |
| Simulation | The simulator (Gazebo) |
| Viewing a model without hardware | `joint_state_publisher` or `joint_state_publisher_gui` |

### `joint_state_publisher` and its GUI

`joint_state_publisher` reads the model from `/robot_description` and publishes a position for every movable joint, by default zero. The **GUI** variant opens a window with a slider per joint, which is very useful to check that joints move in the right direction and within their limits.

If no one publishes `/joint_states`, the movable links have no transform. RViz2 then shows them at the origin with the error `No transform from [left_wheel] to [base_footprint]`, while fixed links display correctly.

---

## A Launch File for the Robot

Create `~/ros2_ws/src/my_robot_description/launch/display.launch.py`:

```python
import os

from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument
from launch.conditions import IfCondition, UnlessCondition
from launch.substitutions import Command, LaunchConfiguration
from launch_ros.actions import Node
from launch_ros.parameter_descriptions import ParameterValue


def generate_launch_description():
    pkg_share = get_package_share_directory('my_robot_description')
    xacro_file = os.path.join(pkg_share, 'urdf', 'my_robot.urdf.xacro')

    gui = LaunchConfiguration('gui')
    use_laser = LaunchConfiguration('use_laser')

    robot_description = ParameterValue(
        Command(['xacro ', xacro_file, ' use_laser:=', use_laser]),
        value_type=str,
    )

    return LaunchDescription([
        DeclareLaunchArgument(
            'gui', default_value='true',
            description='Start joint_state_publisher_gui with sliders'),
        DeclareLaunchArgument(
            'use_laser', default_value='true',
            description='Include the laser scanner in the model'),

        Node(
            package='robot_state_publisher',
            executable='robot_state_publisher',
            parameters=[{'robot_description': robot_description}],
        ),
        Node(
            package='joint_state_publisher_gui',
            executable='joint_state_publisher_gui',
            condition=IfCondition(gui),
        ),
        Node(
            package='joint_state_publisher',
            executable='joint_state_publisher',
            condition=UnlessCondition(gui),
        ),
        Node(
            package='rviz2',
            executable='rviz2',
        ),
    ])
```

### How it works

```python
robot_description = ParameterValue(
    Command(['xacro ', xacro_file, ' use_laser:=', use_laser]),
    value_type=str,
)
```

- **`Command([...])`** runs a shell command when the launch file starts and uses its output. Here it runs `xacro` on the model, passing the `use_laser` argument through, and gets the expanded URDF. Note the spaces inside the strings: the pieces are joined into one command line.
- **`ParameterValue(..., value_type=str)`** tells launch to pass the result as a plain string. Without it, launch would try to parse the XML as YAML and fail.

```python
condition=IfCondition(gui)
condition=UnlessCondition(gui)
```

**Conditions** start a node only if a launch argument is true (or false). Exactly one of the two joint state publishers runs: the GUI version by default, the plain version with `gui:=false`.

---

## Build and Run

Build the package (it installs `urdf`, `launch`, and `rviz` as set up in the [module introduction](README.md#before-you-start)):

```bash
$ cd ~/ros2_ws
$ colcon build --packages-select my_robot_description --symlink-install
```

```bash
$ source ~/ros2_ws/install/setup.bash
$ ros2 launch my_robot_description display.launch.py
```

RViz2 and the slider window open. RViz2 starts empty; set it up as in the [URDF](urdf.md#viewing-the-robot) lesson (Fixed Frame `base_footprint`, add RobotModel and TF). In the [next lesson](rviz.md) you will save that setup so it loads automatically.

Try the arguments:

```bash
$ ros2 launch my_robot_description display.launch.py --show-args
$ ros2 launch my_robot_description display.launch.py use_laser:=false
$ ros2 launch my_robot_description display.launch.py gui:=false
```

---

## Inspecting What Is Published

With the launch file running, use the tools from the previous modules:

```bash
$ ros2 topic echo /joint_states --once
header:
  ...
name:
- left_wheel_joint
- right_wheel_joint
position:
- 0.0
- 0.0
...

$ ros2 run tf2_tools view_frames            # the robot's TF tree as a PDF
$ ros2 run tf2_ros tf2_echo base_footprint laser
- Translation: [0.100, 0.000, 0.220]
...
```

The laser is 0.22 m above the ground: `base_link` at 0.1, plus 0.05 (half the chassis) plus 0.02 (half the laser). Checking such numbers with `tf2_echo` is a quick way to catch modeling mistakes.

Move a wheel slider and watch `/joint_states` and `/tf` change, while `/tf_static` stays the same.

---

## Using the Model from Other Launch Files

Later modules (simulation, navigation) need the robot description too. Rather than copying the code above, they include `robot_state_publisher` with the same `Command` / `ParameterValue` pattern, or include a launch file from your description package. Keeping the model and its launch file in a dedicated `*_description` package is what makes that reuse easy.

---

## Common Mistakes

| Symptom | Cause |
|---------|-------|
| `Unable to parse the value of parameter robot_description as yaml` | Missing `ParameterValue(..., value_type=str)` around `Command` |
| `xacro: command not found` / `No such file` in the launch output | `xacro` not installed, or the path to the Xacro file is wrong (check `get_package_share_directory`) |
| `file 'display.launch.py' was not found` | The `launch` folder is not installed in `CMakeLists.txt`, or the package was not rebuilt |
| Wheels shown at the origin, `No transform from [left_wheel]` | No `/joint_states` publisher is running |
| Changes to the Xacro file have no effect | Without `--symlink-install`, rebuild after editing; with it, just restart the launch |
| `robot_state_publisher` exits with a URDF parse error | Expand the model with `xacro` and check it with `check_urdf` |

---

## Try It Yourself

1. Launch with `gui:=false`. What position do the wheels have, and why?
2. Turn the laser joint into a `revolute` joint (see the [URDF exercises](urdf.md#try-it-yourself)) and move it with the GUI slider. Watch `tf2_echo base_link laser` while you do.
3. Write a small Python node that publishes `/joint_states` itself, making both wheels rotate continuously. Run the launch file with `gui:=false` but without starting `joint_state_publisher` (comment it out), and run your node instead.
4. Add a launch argument `xacro_file` so a different model file can be displayed with the same launch file.

---

⬅️ [Previous: Xacro](xacro.md) | [⬆️ Robot Modeling](README.md) | [Next: RViz2](rviz.md) ➡️
