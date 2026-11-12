# Sensors

A robot without sensors is blind. In this lesson you give `my_robot` three simulated sensors: a **LiDAR**, a **camera**, and an **IMU**. Gazebo generates their data from the virtual world, the bridge forwards it to ROS 2, and you visualize it in RViz2. To your ROS 2 nodes, the data looks exactly like data from real sensors.

## Topics Covered

- How simulated sensors work
- A 2D LiDAR (`gpu_lidar`)
- A camera and its optical frame
- An IMU
- Adding sensor noise
- Bridging sensor topics
- Viewing sensor data in RViz2

---

## How Simulated Sensors Work

A sensor is declared inside a `<gazebo reference="link">` tag, which attaches it to a link of the robot:

```xml
<gazebo reference="laser">
  <sensor name="lidar" type="gpu_lidar">
    <topic>/scan</topic>                <!-- Gazebo topic for the data -->
    <gz_frame_id>laser</gz_frame_id>    <!-- frame_id written into the messages -->
    <update_rate>10</update_rate>        <!-- Hz -->
    ...                                  <!-- sensor-specific settings -->
  </sensor>
</gazebo>
```

Three things must line up for the data to be usable in ROS 2:

| Requirement | Why |
|-------------|-----|
| The world loads the right **systems** (`Sensors`, `Imu`) | Without them, sensors produce no data. Your `my_world.sdf` from the [Gazebo](gazebo.md#writing-a-world-file) lesson already includes both |
| `gz_frame_id` matches a **TF frame** from your URDF | RViz2 and other nodes place the data using TF |
| The topic is **bridged** | Gazebo topics are invisible to ROS 2 otherwise |

> 📝 Without `<gz_frame_id>`, Gazebo writes a scoped name such as `my_robot/base_footprint/lidar` into the messages, which doesn't exist in your TF tree.

All sensors below go into `my_robot.gazebo.xacro`, inside the `<robot>` element.

---

## LiDAR

`my_robot` already has a `laser` link. Attach a 360° 2D laser scanner to it:

```xml
<gazebo reference="laser">
  <sensor name="lidar" type="gpu_lidar">
    <topic>/scan</topic>
    <gz_frame_id>laser</gz_frame_id>
    <update_rate>10</update_rate>
    <always_on>true</always_on>
    <visualize>true</visualize>
    <lidar>
      <scan>
        <horizontal>
          <samples>360</samples>
          <resolution>1</resolution>
          <min_angle>${-pi}</min_angle>
          <max_angle>${pi}</max_angle>
        </horizontal>
      </scan>
      <range>
        <min>0.12</min>
        <max>10.0</max>
        <resolution>0.01</resolution>
      </range>
      <noise>
        <type>gaussian</type>
        <mean>0.0</mean>
        <stddev>0.01</stddev>
      </noise>
    </lidar>
  </sensor>
</gazebo>
```

| Setting | Meaning |
|---------|---------|
| `type="gpu_lidar"` | A laser scanner computed on the graphics card |
| `samples`, `min_angle`, `max_angle` | 360 rays over a full circle: one measurement per degree |
| `range` | Minimum and maximum measurable distance in meters |
| `update_rate` | Scans per second |
| `visualize` | Draw the rays in the Gazebo window |

These values roughly match common low-cost 2D LiDARs (for example the RPLIDAR series).

The range minimum of 0.12 m matters: the laser sits on top of the chassis, and the minimum range keeps it from detecting the robot's own body.

---

## Camera

The camera needs a link of its own. Add it to `my_robot.urdf.xacro`, for example after the laser:

```xml
<!-- Camera at the front of the chassis -->
<link name="camera_link">
  <visual>
    <geometry>
      <box size="0.03 0.08 0.03"/>
    </geometry>
    <material name="black"/>
  </visual>
  <collision>
    <geometry>
      <box size="0.03 0.08 0.03"/>
    </geometry>
  </collision>
  <xacro:box_inertia m="0.05" x="0.03" y="0.08" z="0.03"/>
</link>

<joint name="camera_joint" type="fixed">
  <parent link="base_link"/>
  <child link="camera_link"/>
  <origin xyz="${base_length / 2 + 0.015} 0 0.02" rpy="0 0 0"/>
</joint>

<!-- Optical frame: z forward, x right, y down (camera convention) -->
<link name="camera_optical_frame"/>

<joint name="camera_optical_joint" type="fixed">
  <parent link="camera_link"/>
  <child link="camera_optical_frame"/>
  <origin xyz="0 0 0" rpy="${-pi / 2} 0 ${-pi / 2}"/>
</joint>
```

Why two frames? `camera_link` follows the robot convention (x forward, z up). Image processing libraries, however, expect the **optical** convention: z pointing into the image, x right, y down ([Coordinate Frames](../06-tf2/coordinate-frames.md#units-and-axes-rep-103)). The extra frame, rotated by `rpy="-π/2 0 -π/2"`, provides it. Camera messages use the optical frame.

Now the sensor, in `my_robot.gazebo.xacro`:

```xml
<gazebo reference="camera_link">
  <sensor name="camera" type="camera">
    <topic>/camera/image_raw</topic>
    <gz_frame_id>camera_optical_frame</gz_frame_id>
    <update_rate>30</update_rate>
    <always_on>true</always_on>
    <camera>
      <horizontal_fov>1.047</horizontal_fov>
      <image>
        <width>640</width>
        <height>480</height>
        <format>R8G8B8</format>
      </image>
      <clip>
        <near>0.05</near>
        <far>50.0</far>
      </clip>
    </camera>
  </sensor>
</gazebo>
```

| Setting | Meaning |
|---------|---------|
| `horizontal_fov` | Field of view in radians: 1.047 ≈ 60° |
| `image` | Resolution and pixel format |
| `clip` | Nearest and farthest visible distance |

Gazebo also publishes the camera's calibration on `/camera/camera_info` automatically.

---

## IMU

An **IMU** (Inertial Measurement Unit) measures angular velocity and linear acceleration, and estimates orientation. It is usually mounted near the robot's center:

```xml
<gazebo reference="base_link">
  <sensor name="imu" type="imu">
    <topic>/imu</topic>
    <gz_frame_id>base_link</gz_frame_id>
    <update_rate>100</update_rate>
    <always_on>true</always_on>
  </sensor>
</gazebo>
```

The world must load the `Imu` system (`gz-sim-imu-system`), which `my_world.sdf` does.

---

## Sensor Noise

Real sensors are never perfect, and code that only ever saw perfect data often fails on hardware. Gazebo can add noise. You saw it for the LiDAR above; the camera accepts the same block inside `<camera>`, and the IMU can have noise per axis:

```xml
<imu>
  <angular_velocity>
    <z>
      <noise type="gaussian">
        <mean>0.0</mean>
        <stddev>0.002</stddev>
      </noise>
    </z>
  </angular_velocity>
</imu>
```

Start with small values. Once your nodes work, increase the noise to test how robust they are.

---

## Bridging the Sensor Topics

Add the sensor topics to `my_robot_gazebo/config/bridge.yaml`:

```yaml
- ros_topic_name: "/scan"
  gz_topic_name: "/scan"
  ros_type_name: "sensor_msgs/msg/LaserScan"
  gz_type_name: "gz.msgs.LaserScan"
  direction: GZ_TO_ROS

- ros_topic_name: "/camera/image_raw"
  gz_topic_name: "/camera/image_raw"
  ros_type_name: "sensor_msgs/msg/Image"
  gz_type_name: "gz.msgs.Image"
  direction: GZ_TO_ROS

- ros_topic_name: "/camera/camera_info"
  gz_topic_name: "/camera/camera_info"
  ros_type_name: "sensor_msgs/msg/CameraInfo"
  gz_type_name: "gz.msgs.CameraInfo"
  direction: GZ_TO_ROS

- ros_topic_name: "/imu"
  gz_topic_name: "/imu"
  ros_type_name: "sensor_msgs/msg/Imu"
  gz_type_name: "gz.msgs.IMU"
  direction: GZ_TO_ROS
```

> 💡 For high-resolution or high-rate images, the dedicated `ros_gz_image` package (`ros2 run ros_gz_image image_bridge /camera/image_raw`) is more efficient than the generic bridge and supports compressed transports.

Rebuild (or just restart, with `--symlink-install`) and launch the simulation:

```bash
$ ros2 launch my_robot_gazebo sim.launch.py
```

In Gazebo you see the blue laser rays around the robot. Check the ROS side:

```bash
$ ros2 topic hz /scan
average rate: 10.000

$ ros2 topic echo /scan --once --no-arr
header:
  stamp: ...
  frame_id: laser
angle_min: -3.1415...
angle_max: 3.1415...
...

$ ros2 topic echo /imu --once --field angular_velocity
```

---

## Viewing Sensor Data in RViz2

```bash
$ rviz2 --ros-args -p use_sim_time:=true
```

1. **Fixed Frame:** `odom`.
2. **Add** RobotModel (Description Topic `/robot_description`) and TF.
3. **Add → By topic → /scan → LaserScan.** Set **Size (m)** to `0.03`. The walls and obstacles of the room appear as points around the robot.
4. **Add → By topic → /camera/image_raw → Image.** A panel shows the camera view.
5. **Add → Imu** display (from `rviz_imu_plugin`, if installed: `sudo apt install ros-jazzy-rviz-imu-plugin`), topic `/imu`.

Drive with `teleop_twist_keyboard` and watch the scan update as the robot moves. Notice that the scan points stay aligned with the walls while the robot turns: RViz2 transforms them from `laser` into `odom` using the TF tree, exactly as described in [TF2 Listeners](../06-tf2/tf2-listener.md#transforming-data-between-frames).

> 📝 If a display stays empty although `ros2 topic hz` shows data, compare the publisher's QoS (`ros2 topic info /scan --verbose`) with the display's **Reliability Policy** setting (see [RViz2](../07-robot-modeling/rviz.md#displaying-sensor-data-a-fake-laser)).

---

## Common Problems

| Symptom | Cause / fix |
|---------|-------------|
| No sensor topics in `gz topic -l` | The `Sensors` (or `Imu`) system is missing from the world |
| LiDAR data in Gazebo, nothing in ROS 2 | Topic not in `bridge.yaml`, or a type mismatch |
| RViz2: `Message removed because it is too old` / frame errors | `gz_frame_id` missing or not matching a TF frame, or RViz2 runs without `use_sim_time` |
| The laser detects the robot itself (points very close) | `range/min` too small, or the laser sits inside the chassis |
| Camera image is black | Missing light in the world, or rendering problems (see [Gazebo troubleshooting](gazebo.md#troubleshooting)) |
| Simulation gets very slow after adding sensors | Lower `update_rate`, `samples`, or image resolution |

---

## Try It Yourself

1. Reduce the LiDAR to a 180° field of view in front of the robot (`min_angle = -π/2`, `max_angle = π/2`). How does the scan look in RViz2?
2. Increase the LiDAR noise to `stddev 0.05` and watch the walls become "fuzzy".
3. Write a Python node that subscribes to `/scan` and logs the distance to the closest obstacle and its angle in degrees, once per second.
4. Write a node that subscribes to `/imu` and logs the robot's yaw rate (`angular_velocity.z`) while you turn with teleop. Compare it with the angular velocity you command.
5. Use `xacro` arguments to make the camera optional, like the laser (`use_camera:=false`).

---

⬅️ [Previous: Spawning Robots](spawning-robots.md) | [⬆️ Simulation](README.md) | [Next: Simulation Project](simulation-project.md) ➡️
