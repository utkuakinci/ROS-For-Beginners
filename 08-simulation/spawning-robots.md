# Spawning Robots

In this lesson you put `my_robot` from [Module 07](../07-robot-modeling/README.md) into your Gazebo world and drive it from ROS 2. You add Gazebo plugins that make the wheels drivable and publish odometry, write a launch file that starts Gazebo, spawns the robot, and bridges the topics, and drive the robot with the keyboard.

## Topics Covered

- What Gazebo needs from a URDF
- Gazebo-specific tags: `<gazebo>` and plugins
- The differential drive and joint state publisher systems
- Spawning a robot with `ros_gz_sim create`
- Bridging topics with a configuration file
- A launch file for the whole simulation
- Driving the robot and checking odometry and TF

---

## What Gazebo Needs from a URDF

When you spawn a URDF robot, Gazebo converts it to SDF. A model that looks fine in RViz2 can still fail in simulation, because Gazebo needs more:

| Requirement | Why | `my_robot` |
|-------------|-----|------------|
| `<inertial>` on every link with geometry | Without mass, physics cannot move the link; it may be ignored | ✅ Added with the [inertia macros](../07-robot-modeling/xacro.md#inertia-macros) |
| `<collision>` on links that touch things | Without it, the robot falls through the floor or passes through walls | ✅ |
| Plugins for actuators and sensors | Joints don't move by themselves; sensors need to be declared | ❌ Added in this lesson |

> 📝 During conversion, links connected by **fixed** joints are merged into their parent (for example `laser` into `base_link`). This is normal and doesn't affect TF, which still comes from `robot_state_publisher`. It is why Gazebo's entity tree shows fewer links than your URDF.

---

## Gazebo Tags in the URDF

Gazebo-specific information goes into `<gazebo>` tags inside the URDF. ROS tools ignore them; Gazebo reads them during conversion.

| Form | Applies to |
|------|-----------|
| `<gazebo> ... </gazebo>` | The whole robot model (used for plugins) |
| `<gazebo reference="link_name"> ... </gazebo>` | One link (used for friction, sensors) |

Keep them in a separate file so the description stays readable. Create `~/ros2_ws/src/my_robot_description/urdf/my_robot.gazebo.xacro`:

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro">

  <!-- The caster should slide freely, not brake the robot -->
  <gazebo reference="caster">
    <mu1>0.0</mu1>
    <mu2>0.0</mu2>
  </gazebo>

  <gazebo>
    <!-- Drive the two wheels from Twist commands and publish odometry -->
    <plugin filename="gz-sim-diff-drive-system" name="gz::sim::systems::DiffDrive">
      <left_joint>left_wheel_joint</left_joint>
      <right_joint>right_wheel_joint</right_joint>
      <wheel_separation>${2 * wheel_y_offset}</wheel_separation>
      <wheel_radius>${wheel_radius}</wheel_radius>
      <max_linear_acceleration>1.0</max_linear_acceleration>
      <topic>/cmd_vel</topic>
      <odom_topic>/odom</odom_topic>
      <tf_topic>/tf</tf_topic>
      <frame_id>odom</frame_id>
      <child_frame_id>base_footprint</child_frame_id>
      <odom_publish_frequency>30</odom_publish_frequency>
    </plugin>

    <!-- Publish the positions of all joints, so the wheels turn in RViz2 -->
    <plugin filename="gz-sim-joint-state-publisher-system"
            name="gz::sim::systems::JointStatePublisher">
      <topic>/joint_states</topic>
    </plugin>
  </gazebo>

</robot>
```

Include it at the end of `my_robot.urdf.xacro`, just before `</robot>`. Because it is included, it can use the properties defined there, such as `wheel_radius`:

```xml
  <xacro:include filename="my_robot.gazebo.xacro"/>

</robot>
```

### The plugins

**DiffDrive** turns velocity commands into wheel motion, like the motor controller of a real differential-drive robot:

| Setting | Meaning |
|---------|---------|
| `left_joint`, `right_joint` | The wheel joints to drive |
| `wheel_separation` | Distance between the wheels: 2 × 0.175 = 0.35 m |
| `wheel_radius` | Must match the model, or odometry will be wrong |
| `topic` | Where it listens for `Twist` commands |
| `odom_topic`, `tf_topic` | Where it publishes odometry and the `odom → base_footprint` transform |
| `frame_id`, `child_frame_id` | The frame names used in odometry and TF ([REP 105](../06-tf2/coordinate-frames.md#standard-frames-rep-105)) |

**JointStatePublisher** publishes the simulated wheel positions. It replaces the `joint_state_publisher_gui` sliders you used in Module 07: in simulation, joint states come from physics.

The friction values `mu1` / `mu2` of 0 make the caster a frictionless slider, a common simplification for ball casters.

---

## The Bridge Configuration

The robot now needs several bridged topics. Instead of a long command line, list them in a YAML file. Create `~/ros2_ws/src/my_robot_gazebo/config/bridge.yaml`:

```yaml
- ros_topic_name: "/clock"
  gz_topic_name: "/clock"
  ros_type_name: "rosgraph_msgs/msg/Clock"
  gz_type_name: "gz.msgs.Clock"
  direction: GZ_TO_ROS

- ros_topic_name: "/cmd_vel"
  gz_topic_name: "/cmd_vel"
  ros_type_name: "geometry_msgs/msg/Twist"
  gz_type_name: "gz.msgs.Twist"
  direction: ROS_TO_GZ

- ros_topic_name: "/odom"
  gz_topic_name: "/odom"
  ros_type_name: "nav_msgs/msg/Odometry"
  gz_type_name: "gz.msgs.Odometry"
  direction: GZ_TO_ROS

- ros_topic_name: "/tf"
  gz_topic_name: "/tf"
  ros_type_name: "tf2_msgs/msg/TFMessage"
  gz_type_name: "gz.msgs.Pose_V"
  direction: GZ_TO_ROS

- ros_topic_name: "/joint_states"
  gz_topic_name: "/joint_states"
  ros_type_name: "sensor_msgs/msg/JointState"
  gz_type_name: "gz.msgs.Model"
  direction: GZ_TO_ROS
```

| Topic | Direction | Purpose |
|-------|-----------|---------|
| `/clock` | Gazebo → ROS | Simulation time for `use_sim_time` |
| `/cmd_vel` | ROS → Gazebo | Velocity commands to the DiffDrive plugin |
| `/odom` | Gazebo → ROS | Odometry from the wheels |
| `/tf` | Gazebo → ROS | The `odom → base_footprint` transform |
| `/joint_states` | Gazebo → ROS | Wheel positions for `robot_state_publisher` |

---

## The Launch File

Create `~/ros2_ws/src/my_robot_gazebo/launch/sim.launch.py`:

```python
import os

from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch.actions import IncludeLaunchDescription
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.substitutions import Command
from launch_ros.actions import Node
from launch_ros.parameter_descriptions import ParameterValue


def generate_launch_description():
    description_share = get_package_share_directory('my_robot_description')
    gazebo_share = get_package_share_directory('my_robot_gazebo')

    xacro_file = os.path.join(description_share, 'urdf', 'my_robot.urdf.xacro')
    world_file = os.path.join(gazebo_share, 'worlds', 'my_world.sdf')
    bridge_config = os.path.join(gazebo_share, 'config', 'bridge.yaml')

    robot_description = ParameterValue(Command(['xacro ', xacro_file]), value_type=str)

    gazebo = IncludeLaunchDescription(
        PythonLaunchDescriptionSource(os.path.join(
            get_package_share_directory('ros_gz_sim'), 'launch', 'gz_sim.launch.py')),
        launch_arguments={'gz_args': f'-r {world_file}'}.items(),
    )

    robot_state_publisher = Node(
        package='robot_state_publisher',
        executable='robot_state_publisher',
        parameters=[{'robot_description': robot_description, 'use_sim_time': True}],
    )

    spawn_robot = Node(
        package='ros_gz_sim',
        executable='create',
        arguments=['-topic', 'robot_description', '-name', 'my_robot', '-z', '0.05'],
    )

    bridge = Node(
        package='ros_gz_bridge',
        executable='parameter_bridge',
        parameters=[{'config_file': bridge_config, 'use_sim_time': True}],
    )

    return LaunchDescription([gazebo, robot_state_publisher, spawn_robot, bridge])
```

What each part does:

| Part | Role |
|------|------|
| `gz_sim.launch.py` with `-r <world>` | Starts Gazebo with your world, already running |
| `robot_state_publisher` | Publishes the model on `/robot_description` and the robot's internal TF, with simulation time |
| `ros_gz_sim create` | Reads the model from the `/robot_description` topic and spawns it as `my_robot`, 5 cm above the ground |
| `parameter_bridge` | Connects the topics listed in `bridge.yaml` |

Declare the dependencies in `my_robot_gazebo/package.xml`:

```xml
<exec_depend>my_robot_description</exec_depend>
<exec_depend>robot_state_publisher</exec_depend>
<exec_depend>ros_gz_sim</exec_depend>
<exec_depend>ros_gz_bridge</exec_depend>
<exec_depend>xacro</exec_depend>
```

---

## Build and Run

```bash
$ cd ~/ros2_ws
$ colcon build --packages-select my_robot_description my_robot_gazebo --symlink-install
```

```bash
$ source ~/ros2_ws/install/setup.bash
$ ros2 launch my_robot_gazebo sim.launch.py
```

Gazebo opens with the room, and `my_robot` appears in the middle.

### Drive it

```bash
$ ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

`teleop_twist_keyboard` publishes on `/cmd_vel`. Use `i` to drive forward, `j` / `l` to turn, `,` to reverse, and `k` to stop. Drive into a wall: the robot is stopped by the collision.

### Check the ROS side

```bash
$ ros2 topic list
/clock
/cmd_vel
/joint_states
/odom
/robot_description
/tf
/tf_static
...

$ ros2 topic echo /odom --field pose.pose.position
$ ros2 run tf2_tools view_frames
```

The TF tree now starts at `odom`: Gazebo's DiffDrive publishes `odom → base_footprint`, and `robot_state_publisher` publishes everything below it.

```text
odom ──(Gazebo DiffDrive)──▶ base_footprint ──(robot_state_publisher)──▶ base_link ──▶ wheels, laser, caster
```

Open RViz2 with simulation time:

```bash
$ rviz2 --ros-args -p use_sim_time:=true
```

Set **Fixed Frame** to `odom` and add **RobotModel** and **TF**. Drive the robot: it moves in RViz2 exactly as in Gazebo, and its wheels turn.

---

## Common Problems

| Symptom | Cause / fix |
|---------|-------------|
| The robot doesn't appear | `create` couldn't read the model. Check the launch output for URDF errors; check `ros2 topic echo /robot_description --once` |
| The robot falls through the floor or explodes | Missing or unrealistic `<inertial>` / `<collision>` values |
| The robot doesn't move with teleop | `/cmd_vel` not bridged, or the plugin topic differs from the bridge. Check `gz topic -l` and `gz topic -e -t /cmd_vel` |
| The robot drives backwards or spins in place | Wheel joint axes point the wrong way, or the left and right joints are swapped |
| RViz2: `Fixed Frame [odom] does not exist` | `/tf` not bridged, or `tf_topic` missing in the DiffDrive plugin |
| RViz2: wheels at the origin, `No transform` | `/joint_states` not bridged |
| TF errors about extrapolation | A node is missing `use_sim_time:=true` |
| Gazebo keeps running after `Ctrl` + `C` | Stop leftovers with `pkill -f "gz sim"` |

---

## Try It Yourself

1. Change `max_linear_acceleration` to `0.2` and feel the difference when driving.
2. Spawn the robot in a different place by adding `-x 1.0 -y -1.0 -Y 1.57` to the `create` arguments.
3. Add RViz2 to `sim.launch.py` with `use_sim_time` and the configuration file from [Module 07](../07-robot-modeling/rviz.md#saving-the-configuration) (change its Fixed Frame to `odom`).
4. Drive the robot in a square back to its start and compare `/odom` with the real position shown in Gazebo. How large is the odometry error?
5. Write a launch argument `world` so `sim.launch.py` can start other world files.

---

⬅️ [Previous: Gazebo](gazebo.md) | [⬆️ Simulation](README.md) | [Next: Sensors](sensors.md) ➡️
