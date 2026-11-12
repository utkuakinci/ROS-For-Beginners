# Simulation Project

Time to put the module together. In this project you finish the simulation setup so that one command starts Gazebo, the robot, the bridge, and RViz2, and then write `scan_avoider`: a node that uses the simulated LiDAR to explore the room on its own without hitting anything. It is your first complete **sense → decide → act** robot behavior running on realistic sensor data.

## Topics Covered

- A configurable launch file for the whole simulation
- Starting RViz2 with simulation time
- Processing `LaserScan` data in Python
- A reactive obstacle avoidance node
- Recording a simulation run
- From simulation to a real robot

---

## The Goal

```text
                 /scan                          /cmd_vel
 ┌──────────┐ ─────────▶ ┌──────────────┐ ─────────────▶ ┌──────────┐
 │  Gazebo  │            │ scan_avoider │                │  Gazebo  │
 │  LiDAR   │            │  (your node) │                │ DiffDrive│
 └──────────┘            └──────────────┘                └──────────┘
```

When you are done, this is all it takes to watch the robot explore:

```bash
$ ros2 launch my_robot_gazebo sim.launch.py
$ ros2 run my_first_pkg scan_avoider --ros-args -p use_sim_time:=true
```

---

## Step 1: A Configurable Launch File

Extend `my_robot_gazebo/launch/sim.launch.py` from [Spawning Robots](spawning-robots.md#the-launch-file) with launch arguments for the world, the spawn position, and RViz2:

```python
import os

from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument, IncludeLaunchDescription
from launch.conditions import IfCondition
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.substitutions import Command, LaunchConfiguration
from launch_ros.actions import Node
from launch_ros.parameter_descriptions import ParameterValue


def generate_launch_description():
    description_share = get_package_share_directory('my_robot_description')
    gazebo_share = get_package_share_directory('my_robot_gazebo')

    xacro_file = os.path.join(description_share, 'urdf', 'my_robot.urdf.xacro')
    bridge_config = os.path.join(gazebo_share, 'config', 'bridge.yaml')
    rviz_config = os.path.join(gazebo_share, 'rviz', 'sim.rviz')

    world = LaunchConfiguration('world')
    x = LaunchConfiguration('x')
    y = LaunchConfiguration('y')
    use_rviz = LaunchConfiguration('use_rviz')

    robot_description = ParameterValue(Command(['xacro ', xacro_file]), value_type=str)

    return LaunchDescription([
        DeclareLaunchArgument(
            'world', default_value=os.path.join(gazebo_share, 'worlds', 'my_world.sdf'),
            description='Full path to the world file'),
        DeclareLaunchArgument('x', default_value='0.0', description='Spawn x position'),
        DeclareLaunchArgument('y', default_value='0.0', description='Spawn y position'),
        DeclareLaunchArgument('use_rviz', default_value='true', description='Start RViz2'),

        IncludeLaunchDescription(
            PythonLaunchDescriptionSource(os.path.join(
                get_package_share_directory('ros_gz_sim'), 'launch', 'gz_sim.launch.py')),
            launch_arguments={'gz_args': ['-r ', world]}.items(),
        ),

        Node(
            package='robot_state_publisher',
            executable='robot_state_publisher',
            parameters=[{'robot_description': robot_description, 'use_sim_time': True}],
        ),

        Node(
            package='ros_gz_sim',
            executable='create',
            arguments=['-topic', 'robot_description', '-name', 'my_robot',
                       '-x', x, '-y', y, '-z', '0.05'],
        ),

        Node(
            package='ros_gz_bridge',
            executable='parameter_bridge',
            parameters=[{'config_file': bridge_config, 'use_sim_time': True}],
        ),

        Node(
            package='rviz2',
            executable='rviz2',
            arguments=['-d', rviz_config],
            parameters=[{'use_sim_time': True}],
            condition=IfCondition(use_rviz),
        ),
    ])
```

New compared to the previous version:

- **Launch arguments** for the world and spawn position, so the same file works for any world. Note `'gz_args': ['-r ', world]`: a list mixes plain text with a `LaunchConfiguration`, and launch joins them at run time.
- **RViz2** with `use_sim_time` and a saved configuration, started only when `use_rviz` is true.

### The RViz2 configuration

Add an `rviz` folder to the package and install it:

```bash
$ mkdir ~/ros2_ws/src/my_robot_gazebo/rviz
```

```cmake
install(DIRECTORY worlds config launch rviz
  DESTINATION share/${PROJECT_NAME}
)
```

The configuration file must exist before RViz2 can load it. Create it once: start the simulation with `use_rviz:=false`, start `rviz2 --ros-args -p use_sim_time:=true` by hand, set it up (Fixed Frame `odom`; RobotModel, TF, LaserScan on `/scan`, Image on `/camera/image_raw`; see [Sensors](sensors.md#viewing-sensor-data-in-rviz2)), and save it as `~/ros2_ws/src/my_robot_gazebo/rviz/sim.rviz`. Add `<exec_depend>rviz2</exec_depend>` to `package.xml`, rebuild, and from now on:

```bash
$ ros2 launch my_robot_gazebo sim.launch.py
$ ros2 launch my_robot_gazebo sim.launch.py x:=-2.0 y:=2.0 use_rviz:=false
```

---

## Step 2: Understanding `LaserScan`

```bash
$ ros2 interface show sensor_msgs/msg/LaserScan
std_msgs/Header header
float32 angle_min            # start angle of the scan [rad]
float32 angle_max            # end angle of the scan [rad]
float32 angle_increment      # angle between measurements [rad]
float32 time_increment
float32 scan_time
float32 range_min            # minimum valid range [m]
float32 range_max            # maximum valid range [m]
float32[] ranges             # distances [m]
float32[] intensities
```

`ranges[i]` is the distance measured at angle `angle_min + i * angle_increment`, in the `laser` frame: angle 0 is straight ahead (x), positive angles are to the left. With the sensor from the previous lesson, there are 360 values from -π to π.

```text
                 0 rad (front)
                     │
      +π/2 (left) ───●─── -π/2 (right)
                     │
                  ±π (back)
```

Not every value is a valid distance. When a ray hits nothing within `range_max`, the value is `inf`; invalid readings can be `nan`. Always filter them out.

---

## Step 3: The `scan_avoider` Node

The strategy is simple and surprisingly effective:

1. Look at three **sectors** of the scan: front, left, and right.
2. If the front is clear, drive forward.
3. Otherwise, stop moving forward and turn toward the side with more space.

Create `~/ros2_ws/src/my_first_pkg/my_first_pkg/scan_avoider.py`:

```python
import math

import rclpy
from geometry_msgs.msg import Twist
from rclpy.node import Node
from rclpy.qos import qos_profile_sensor_data
from sensor_msgs.msg import LaserScan


class ScanAvoider(Node):
    def __init__(self):
        super().__init__('scan_avoider')
        self.declare_parameter('safe_distance', 0.6)
        self.declare_parameter('linear_speed', 0.3)
        self.declare_parameter('angular_speed', 0.8)
        self.declare_parameter('front_angle_deg', 30.0)

        self.scan = None
        self.create_subscription(LaserScan, 'scan', self.scan_callback, qos_profile_sensor_data)
        self.cmd_publisher = self.create_publisher(Twist, 'cmd_vel', 10)
        self.create_timer(0.1, self.control_loop)

    def scan_callback(self, msg):
        self.scan = msg

    def sector_min(self, scan, start_deg, end_deg):
        """Smallest valid range between two angles (degrees, 0 = front, positive = left)."""
        start, end = math.radians(start_deg), math.radians(end_deg)
        closest = math.inf
        for i, r in enumerate(scan.ranges):
            angle = scan.angle_min + i * scan.angle_increment
            if start <= angle <= end and scan.range_min < r < scan.range_max:
                closest = min(closest, r)
        return closest

    def control_loop(self):
        if self.scan is None:
            return

        safe = self.get_parameter('safe_distance').value
        front_deg = self.get_parameter('front_angle_deg').value

        front = self.sector_min(self.scan, -front_deg, front_deg)
        left = self.sector_min(self.scan, front_deg, 90.0)
        right = self.sector_min(self.scan, -90.0, -front_deg)

        cmd = Twist()
        if front > safe:
            cmd.linear.x = self.get_parameter('linear_speed').value
            # Gently steer away from side walls that get too close
            if left < safe:
                cmd.angular.z = -0.3
            elif right < safe:
                cmd.angular.z = 0.3
        else:
            turn = self.get_parameter('angular_speed').value
            cmd.angular.z = turn if left > right else -turn
            self.get_logger().info(
                f'Obstacle {front:.2f} m ahead, turning {"left" if left > right else "right"}',
                throttle_duration_sec=1.0)

        self.cmd_publisher.publish(cmd)


def main():
    rclpy.init()
    node = ScanAvoider()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.cmd_publisher.publish(Twist())  # stop the robot
        node.destroy_node()
        rclpy.try_shutdown()


if __name__ == '__main__':
    main()
```

### What to notice

- **The structure is `wall_bouncer` from [Module 04](../04-python/subscriber.md#sense-decide-act-wall_bouncer)**: store the latest message, decide in a fixed-rate control loop. Only the input changed, from a perfect pose to realistic, noisy sensor data.
- **`qos_profile_sensor_data`** subscribes with best-effort reliability, which works with both best-effort and reliable sensor publishers.
- **Invalid readings are skipped**: values outside `range_min`..`range_max`, including `inf` and `nan`, never count as obstacles.
- **The robot stops on exit**: publishing an empty `Twist` in `finally` prevents the robot from driving on after `Ctrl` + `C`. Always do this in nodes that command motion.
- **No knowledge of the world**: the node doesn't know where the walls are. It reacts only to what it senses. This is called a **reactive** behavior.

### Run it

Register `'scan_avoider = my_first_pkg.scan_avoider:main'` in `setup.py`, add `<depend>sensor_msgs</depend>` to `package.xml` if missing, and rebuild:

```bash
$ cd ~/ros2_ws
$ colcon build --packages-select my_first_pkg my_robot_description my_robot_gazebo --symlink-install
```

```bash
$ ros2 launch my_robot_gazebo sim.launch.py
```

```bash
$ ros2 run my_first_pkg scan_avoider --ros-args -p use_sim_time:=true
[INFO] [scan_avoider]: Obstacle 0.58 m ahead, turning left
...
```

The robot drives through the room, turning away from walls and obstacles. Tune it while it runs:

```bash
$ ros2 param set /scan_avoider safe_distance 1.0
$ ros2 param set /scan_avoider linear_speed 0.5
```

---

## Step 4: Record a Run

`rosbag2` records topics so you can replay and analyze a run later, without the simulator. Record the most important topics for a minute while the robot explores:

```bash
$ ros2 bag record -o explore_run /scan /odom /tf /tf_static /cmd_vel
```

Stop with `Ctrl` + `C`, then inspect and replay it:

```bash
$ ros2 bag info explore_run
$ ros2 bag play explore_run
```

Recording and replaying data is covered in detail in [rosbag2](../11-debugging/rosbag.md).

---

## Checklist

If something doesn't work, go through the chain from Gazebo to your node:

| Check | Command |
|-------|---------|
| Gazebo publishes the scan | `gz topic -l \| grep scan`, `gz topic -e -t /scan -n 1` |
| The bridge forwards it | `ros2 topic hz /scan` |
| The scan is in the right frame | `ros2 topic echo /scan --once --no-arr` → `frame_id: laser` |
| The node receives it and publishes commands | `ros2 topic echo /cmd_vel` |
| The bridge forwards commands to Gazebo | `gz topic -e -t /cmd_vel` |
| All nodes use simulation time | `ros2 param get /scan_avoider use_sim_time` → `True` |

---

## From Simulation to a Real Robot

Your `scan_avoider` doesn't know it runs in a simulator. On a real robot with a LiDAR driver publishing `/scan` and a motor driver listening on `/cmd_vel`, the same node works unchanged. Only the launch file differs: instead of Gazebo, the spawner, and the bridge, it starts the hardware drivers, and `use_sim_time` is `false`.

Expect some differences in practice: real sensors are noisier, wheels slip, and motors respond with delay. Test in simulation first, then on hardware at low speed.

---

## Ideas for Extensions

1. **Wall following:** make the robot keep a constant distance to the wall on its right side, using a proportional controller on the right-sector distance.
2. **Smoother driving:** instead of switching between "drive" and "turn", make the speed proportional to the front distance and the turn rate proportional to the difference between left and right.
3. **A harder world:** create a world with a narrow corridor and a dead end. Does the reactive strategy get stuck? Why?
4. **Camera:** subscribe to `/camera/image_raw` and log the average brightness of the image. Make the robot stop when it sees the red box (hint: count pixels with a high red value).
5. Continue with [Project 02: Obstacle Avoidance](../projects/02-obstacle-avoidance/README.md), which builds on this node.

---

⬅️ [Previous: Sensors](sensors.md) | [⬆️ Simulation](README.md)
