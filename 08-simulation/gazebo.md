# Gazebo

**Gazebo** is an open-source 3D robot simulator. It computes physics (gravity, friction, collisions), simulates sensors (cameras, LiDAR, IMU), and renders the scene. In this lesson you start Gazebo, learn its interface, write your own world file, and connect Gazebo to ROS 2 with the **bridge**.

## Topics Covered

- What a simulator does (and doesn't do)
- Starting Gazebo and its example worlds
- The Gazebo interface
- Worlds and models in SDF
- Writing a world file
- Gazebo topics and the `gz` command-line tool
- Connecting Gazebo and ROS 2 with `ros_gz_bridge`
- Simulation time
- Troubleshooting

---

## What a Simulator Does

| Simulator | ROS 2 nodes |
|-----------|-------------|
| Computes how the robot moves under the commands it receives | Send commands (`/cmd_vel`) |
| Generates sensor data from the virtual world | Receive sensor data (`/scan`, `/camera/image_raw`) |
| Keeps the simulation clock | Use simulation time instead of the wall clock |

The key idea: your ROS 2 nodes should **not know** whether they talk to a simulator or to a real robot. The same `/cmd_vel` and `/scan` topics exist in both cases. Code tested in simulation then runs on hardware with little or no change.

A simulator is never perfect. Friction, sensor noise, and motor behavior always differ from reality, so final testing still happens on real hardware.

---

## Starting Gazebo

```bash
$ gz sim shapes.sdf
```

A window opens with a few simple shapes. `shapes.sdf` is one of many example worlds shipped with Gazebo. The simulation starts **paused**; press the ▶ (play) button at the bottom left to start physics.

Start a world already running with `-r`:

```bash
$ gz sim -r diff_drive.sdf
```

This world contains two small vehicles. You will drive one of them from ROS 2 at the end of this lesson.

> 💡 Running `gz sim` without arguments opens a quick-start dialog listing example worlds and models from [Gazebo Fuel](https://app.gazebosim.org/fuel), an online library of ready-made models.

---

## The Interface

```text
┌─────────────────────────────────────────────────────────────────┐
│  Toolbar: select · translate · rotate · insert box/sphere/...  │
├────────────────────────────────────────────────┬────────────────┤
│                                                │ Entity Tree    │
│                    3D scene                    │ ────────────   │
│                                                │ Component      │
│                                                │ Inspector      │
├────────────────────────────────────────────────┴────────────────┤
│ ▶ ⏸  step            Sim time: 12.3 s   Real time factor: 99 %  │
└─────────────────────────────────────────────────────────────────┘
```

| Element | Purpose |
|---------|---------|
| **Play / Pause / Step** | Start, stop, and advance physics one step at a time |
| **Real Time Factor (RTF)** | Simulation speed relative to real time. 100 % = real time; lower means your computer can't keep up |
| **Entity Tree** | All models, links, lights, and sensors in the world |
| **Component Inspector** | Properties of the selected entity: pose, mass, etc. |
| **Toolbar** | Move and rotate objects, insert simple shapes |
| **⋮ menu (top right)** | Add GUI plugins, for example *Teleop* or *Image Display* |

Camera controls: left-drag pans, right-drag (or `Shift` + left-drag) orbits, scroll zooms.

Try it: insert a box from the toolbar, drag it into the air with the translate tool, and press play. It falls.

---

## Worlds and Models in SDF

Gazebo describes everything in **SDF** (Simulation Description Format), an XML format similar to URDF but more expressive:

| | URDF | SDF |
|---|---|---|
| Describes | One robot | A whole **world**: robots, objects, lights, physics settings |
| Used by | ROS tools (`robot_state_publisher`, RViz2) | Gazebo |
| Structure | Tree of links and joints | Models with links and joints, plus world elements |

You keep writing your robot in URDF/Xacro: Gazebo converts it to SDF automatically when it is spawned (see [Spawning Robots](spawning-robots.md)). **Worlds**, however, are written directly in SDF.

---

## Writing a World File

Create the simulation package:

```bash
$ cd ~/ros2_ws/src
$ ros2 pkg create my_robot_gazebo --build-type ament_cmake --license Apache-2.0
$ cd my_robot_gazebo
$ rm -rf include src
$ mkdir worlds config launch
```

Install the folders in `CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.8)
project(my_robot_gazebo)

find_package(ament_cmake REQUIRED)

install(DIRECTORY worlds config launch
  DESTINATION share/${PROJECT_NAME}
)

ament_package()
```

Now create `worlds/my_world.sdf`: a ground plane, a light, some walls, and a few obstacles.

```xml
<?xml version="1.0" ?>
<sdf version="1.8">
  <world name="my_world">

    <!-- Physics: 1 ms steps, run in real time -->
    <physics name="1ms" type="ignored">
      <max_step_size>0.001</max_step_size>
      <real_time_factor>1.0</real_time_factor>
    </physics>

    <!-- Systems (plugins) the world needs -->
    <plugin filename="gz-sim-physics-system" name="gz::sim::systems::Physics"/>
    <plugin filename="gz-sim-user-commands-system" name="gz::sim::systems::UserCommands"/>
    <plugin filename="gz-sim-scene-broadcaster-system" name="gz::sim::systems::SceneBroadcaster"/>
    <plugin filename="gz-sim-sensors-system" name="gz::sim::systems::Sensors">
      <render_engine>ogre2</render_engine>
    </plugin>
    <plugin filename="gz-sim-imu-system" name="gz::sim::systems::Imu"/>

    <light type="directional" name="sun">
      <cast_shadows>true</cast_shadows>
      <pose>0 0 10 0 0 0</pose>
      <diffuse>0.8 0.8 0.8 1</diffuse>
      <specular>0.2 0.2 0.2 1</specular>
      <direction>-0.5 0.1 -0.9</direction>
    </light>

    <model name="ground_plane">
      <static>true</static>
      <link name="link">
        <collision name="collision">
          <geometry>
            <plane><normal>0 0 1</normal><size>100 100</size></plane>
          </geometry>
        </collision>
        <visual name="visual">
          <geometry>
            <plane><normal>0 0 1</normal><size>100 100</size></plane>
          </geometry>
          <material>
            <ambient>0.8 0.8 0.8 1</ambient>
            <diffuse>0.8 0.8 0.8 1</diffuse>
          </material>
        </visual>
      </link>
    </model>

    <!-- A 6 m x 6 m room: four walls -->
    <model name="wall_north">
      <static>true</static>
      <pose>0 3 0.25 0 0 0</pose>
      <link name="link">
        <collision name="collision"><geometry><box><size>6 0.1 0.5</size></box></geometry></collision>
        <visual name="visual"><geometry><box><size>6 0.1 0.5</size></box></geometry></visual>
      </link>
    </model>
    <model name="wall_south">
      <static>true</static>
      <pose>0 -3 0.25 0 0 0</pose>
      <link name="link">
        <collision name="collision"><geometry><box><size>6 0.1 0.5</size></box></geometry></collision>
        <visual name="visual"><geometry><box><size>6 0.1 0.5</size></box></geometry></visual>
      </link>
    </model>
    <model name="wall_east">
      <static>true</static>
      <pose>3 0 0.25 0 0 0</pose>
      <link name="link">
        <collision name="collision"><geometry><box><size>0.1 6 0.5</size></box></geometry></collision>
        <visual name="visual"><geometry><box><size>0.1 6 0.5</size></box></geometry></visual>
      </link>
    </model>
    <model name="wall_west">
      <static>true</static>
      <pose>-3 0 0.25 0 0 0</pose>
      <link name="link">
        <collision name="collision"><geometry><box><size>0.1 6 0.5</size></box></geometry></collision>
        <visual name="visual"><geometry><box><size>0.1 6 0.5</size></box></geometry></visual>
      </link>
    </model>

    <!-- Obstacles -->
    <model name="box_obstacle">
      <static>true</static>
      <pose>1.5 1.0 0.25 0 0 0.4</pose>
      <link name="link">
        <collision name="collision"><geometry><box><size>0.5 0.5 0.5</size></box></geometry></collision>
        <visual name="visual">
          <geometry><box><size>0.5 0.5 0.5</size></box></geometry>
          <material><ambient>0.8 0.2 0.2 1</ambient><diffuse>0.8 0.2 0.2 1</diffuse></material>
        </visual>
      </link>
    </model>
    <model name="cylinder_obstacle">
      <static>true</static>
      <pose>-1.2 -1.0 0.25 0 0 0</pose>
      <link name="link">
        <collision name="collision"><geometry><cylinder><radius>0.3</radius><length>0.5</length></cylinder></geometry></collision>
        <visual name="visual">
          <geometry><cylinder><radius>0.3</radius><length>0.5</length></cylinder></geometry>
          <material><ambient>0.2 0.7 0.2 1</ambient><diffuse>0.2 0.7 0.2 1</diffuse></material>
        </visual>
      </link>
    </model>

  </world>
</sdf>
```

Key points:

- **`<plugin>`** elements load **systems**, Gazebo's building blocks. Without `Physics`, nothing moves; without `Sensors` and `Imu`, the sensors in the [Sensors](sensors.md) lesson produce no data. The example worlds load these for you; your own world must list them.
- **`<static>true</static>`** fixes a model in place: walls and obstacles don't move or fall.
- **`<pose>x y z roll pitch yaw</pose>`** places a model. Boxes are centered on their origin, so a 0.5 m tall wall is placed at `z = 0.25` to stand on the ground.

Open it directly with Gazebo:

```bash
$ gz sim ~/ros2_ws/src/my_robot_gazebo/worlds/my_world.sdf
```

---

## Gazebo Topics

Gazebo has its **own** communication system, *Gazebo Transport*, with its own topics and message types. It is separate from ROS 2: `ros2 topic list` does not show Gazebo topics. The `gz` tool inspects them:

```bash
$ gz sim -r diff_drive.sdf
```

```bash
$ gz topic -l
/clock
/model/vehicle_blue/cmd_vel
/model/vehicle_blue/odometry
/model/vehicle_green/cmd_vel
...

$ gz topic -e -t /model/vehicle_blue/odometry          # echo
$ gz topic -t /model/vehicle_blue/cmd_vel -m gz.msgs.Twist -p "linear: {x: 0.5}"   # publish
```

The blue vehicle starts moving.

---

## Connecting Gazebo and ROS 2: `ros_gz_bridge`

The **bridge** translates messages between Gazebo Transport and ROS 2, topic by topic:

```text
 ROS 2 nodes ◀──▶  ros_gz_bridge  ◀──▶  Gazebo
 /cmd_vel           (translates)         /cmd_vel
 geometry_msgs/msg/Twist                 gz.msgs.Twist
```

The quickest way to create a bridge is on the command line:

```bash
ros2 run ros_gz_bridge parameter_bridge <topic>@<ROS type><direction><Gazebo type>
```

| Direction symbol | Meaning |
|------------------|---------|
| `@` | Both ways |
| `[` | Gazebo → ROS 2 only |
| `]` | ROS 2 → Gazebo only |

With `diff_drive.sdf` running, bridge the blue vehicle's velocity command from ROS 2 to Gazebo:

```bash
$ ros2 run ros_gz_bridge parameter_bridge \
    /model/vehicle_blue/cmd_vel@geometry_msgs/msg/Twist]gz.msgs.Twist
```

Now drive it with a normal ROS 2 command:

```bash
$ ros2 topic pub -r 10 /model/vehicle_blue/cmd_vel geometry_msgs/msg/Twist "{linear: {x: 1.0}, angular: {z: 0.5}}"
```

The vehicle drives in a circle, commanded from ROS 2. This is the whole idea of ROS–Gazebo integration; the following lessons only add more topics and a launch file.

Common type pairs:

| ROS 2 type | Gazebo type |
|------------|-------------|
| `geometry_msgs/msg/Twist` | `gz.msgs.Twist` |
| `nav_msgs/msg/Odometry` | `gz.msgs.Odometry` |
| `sensor_msgs/msg/LaserScan` | `gz.msgs.LaserScan` |
| `sensor_msgs/msg/Image` | `gz.msgs.Image` |
| `sensor_msgs/msg/Imu` | `gz.msgs.IMU` |
| `sensor_msgs/msg/JointState` | `gz.msgs.Model` |
| `tf2_msgs/msg/TFMessage` | `gz.msgs.Pose_V` |
| `rosgraph_msgs/msg/Clock` | `gz.msgs.Clock` |

The full list is in the [ros_gz_bridge documentation](https://github.com/gazebosim/ros_gz/tree/jazzy/ros_gz_bridge).

---

## Simulation Time

Gazebo keeps its own clock, **simulation time**, which starts at 0 and can run slower or faster than real time, or stop when paused. ROS 2 nodes must use this clock in simulation, otherwise timestamps from sensors and from TF don't match.

Two things are needed:

1. **Bridge the clock** from Gazebo to ROS 2:

   ```bash
   $ ros2 run ros_gz_bridge parameter_bridge /clock@rosgraph_msgs/msg/Clock[gz.msgs.Clock
   ```

2. **Set `use_sim_time:=true`** on every ROS 2 node that runs with the simulation (`robot_state_publisher`, RViz2, your own nodes):

   ```bash
   $ ros2 run my_first_pkg talker --ros-args -p use_sim_time:=true
   ```

With `use_sim_time`, `self.get_clock().now()` returns simulation time, and timers follow it: pause Gazebo, and the node's timers stop too.

> ⚠️ Mixing nodes with and without `use_sim_time` is a classic source of TF errors such as `extrapolation into the past`. In simulation, set it on **all** nodes.

---

## Starting Gazebo from a Launch File

The `ros_gz_sim` package provides a launch file that starts Gazebo. `gz_args` passes arguments to `gz sim`:

```bash
$ ros2 launch ros_gz_sim gz_sim.launch.py gz_args:="-r empty.sdf"
```

In your own launch files, you will include it as shown in the [next lesson](spawning-robots.md).

---

## Troubleshooting

| Symptom | Cause / fix |
|---------|-------------|
| Gazebo window black, crashes on start, or extremely slow | Graphics driver issue. Update GPU drivers; in a VM or WSL2, try `export LIBGL_ALWAYS_SOFTWARE=1` (slow but works) |
| Real Time Factor far below 100 % | The computer can't keep up. Close other programs, simplify the world, reduce sensor rates |
| Nothing moves | The simulation is paused (press play or use `-r`), or the `Physics` system is missing from your world |
| `ros2 topic list` doesn't show Gazebo topics | Gazebo topics need a bridge |
| The bridge runs but no data arrives | Wrong topic name or type pair, or wrong direction symbol. Compare with `gz topic -l` and `gz topic -i -t <topic>` |
| Gazebo processes keep running after closing | Kill them: `pkill -f "gz sim"` |

---

## Try It Yourself

1. Open `shapes.sdf`, insert a sphere above the box, and watch it fall and roll.
2. Add a third obstacle to `my_world.sdf` and a doorway in one of the walls (split the wall into two shorter boxes).
3. With `diff_drive.sdf`, bridge `/model/vehicle_blue/odometry` to ROS 2 (`nav_msgs/msg/Odometry`, direction `[`) and echo it with `ros2 topic echo`.
4. Drive the green vehicle from ROS 2 with `teleop_twist_keyboard`, remapping its output: `ros2 run teleop_twist_keyboard teleop_twist_keyboard --ros-args -r cmd_vel:=/model/vehicle_green/cmd_vel` (after bridging that topic).
5. Pause the simulation while echoing the bridged `/clock`. What happens to the published time?

---

[⬆️ Simulation](README.md) | [Next: Spawning Robots](spawning-robots.md) ➡️
