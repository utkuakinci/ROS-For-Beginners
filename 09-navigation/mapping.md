# Mapping (SLAM)

A robot that should navigate needs a map. **SLAM** (Simultaneous Localization and Mapping) builds one while the robot explores: it uses the laser scans to draw the environment and, at the same time, works out where the robot is in the map it is drawing. In this lesson you map your Gazebo world with **slam_toolbox** and save the result for the following lessons.

## Topics Covered

- What SLAM is and why it is hard
- Occupancy grid maps
- Running `slam_toolbox` in simulation
- Watching the map grow in RViz2
- Driving to build a good map
- Saving the map and understanding the map files

## In Simple Terms

SLAM is like **sketching a map of a dark house with a flashlight**, while also keeping track of where you are on your own sketch. Small mistakes add up, until you come back to a place you recognize ("the front door again!") and fix the whole drawing at once: that's **loop closure**.

**Example:** you drive `my_robot` around the room with the keyboard, the map grows in RViz2, and at the end you save it as an image file.

---

## What Is SLAM?

Mapping would be easy if the robot knew exactly where it was: just draw each scan at the robot's position. Localization would be easy with a map: compare the scan with the map. SLAM has to do **both at once**, which is a chicken-and-egg problem.

Why not simply use odometry for the position? Because odometry **drifts**: small wheel slip and measurement errors add up, and after a few meters the estimate is noticeably wrong. Draw scans with drifting odometry and walls appear doubled and bent.

SLAM corrects this by **matching** each new scan against the map built so far, and by recognizing places it has seen before (**loop closure**), which lets it correct the accumulated drift of the whole trajectory.

In ROS 2, the result shows up in two places:

| Output | Meaning |
|--------|---------|
| `/map` topic (`nav_msgs/msg/OccupancyGrid`) | The map built so far |
| `map → odom` transform | The correction SLAM applies to odometry (see [REP 105](../06-tf2/coordinate-frames.md#standard-frames-rep-105)) |

```text
map ──(slam_toolbox)──▶ odom ──(Gazebo DiffDrive)──▶ base_footprint ──(robot_state_publisher)──▶ ...
```

SLAM publishes `map → odom`, not `map → base_footprint`, so the tree keeps one parent per frame: odometry stays smooth and continuous, and SLAM corrects its drift on top.

---

## Occupancy Grid Maps

ROS navigation uses **occupancy grids**: the world is divided into square cells, and each cell is:

| Value | Meaning | Shown as |
|-------|---------|----------|
| 0 | Free | White / light grey |
| 100 | Occupied (obstacle) | Black |
| -1 | Unknown (never observed) | Dark grey |

```bash
$ ros2 interface show nav_msgs/msg/OccupancyGrid
std_msgs/Header header
MapMetaData info
        builtin_interfaces/Time map_load_time
        float32 resolution        # meters per cell
        uint32 width              # cells
        uint32 height             # cells
        geometry_msgs/Pose origin # real-world pose of cell (0, 0)
int8[] data
```

The **resolution** sets the cell size; 0.05 m (5 cm) is a common default for indoor robots.

---

## Running slam_toolbox

slam_toolbox expects the frames and topics your simulation already provides:

| What | slam_toolbox default | `my_robot` |
|------|----------------------|------------|
| Laser topic | `/scan` | ✅ |
| Odometry frame | `odom` | ✅ |
| Robot base frame | `base_footprint` | ✅ |
| Map frame (published) | `map` | — |

**Terminal 1**, the simulation:

```bash
$ ros2 launch my_robot_gazebo sim.launch.py use_rviz:=false
```

**Terminal 2**, SLAM:

```bash
$ ros2 launch slam_toolbox online_async_launch.py use_sim_time:=true
[INFO] [async_slam_toolbox_node]: Node using stack size 40000000
...
[INFO] [async_slam_toolbox_node]: Registering sensor: [Custom Described Lidar]
```

`online_async` means: map live while the robot drives (*online*), and if processing falls behind, skip scans rather than queue them (*async*). It is the recommended mode for most robots.

**Terminal 3**, RViz2:

```bash
$ rviz2 --ros-args -p use_sim_time:=true
```

1. **Fixed Frame:** `map`.
2. **Add → By topic → /map → Map.** The map starts as a small patch around the robot.
3. Add **RobotModel**, **TF**, and **LaserScan** (`/scan`) as before.

> 📝 `/map` is published with **transient local** durability so late subscribers get it. If the Map display stays empty, check its **Durability Policy** setting.

---

## Building the Map

**Terminal 4**, drive:

```bash
$ ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

Drive slowly through the whole room and watch the map grow: free space appears in white, walls in black, and areas the laser hasn't seen stay grey.

Tips for a good map:

- **Drive slowly**, especially when turning. Fast rotations are the hardest case for scan matching.
- **Cover everything.** Unknown (grey) areas are treated as obstacles or unknowns by the planner later.
- **Close loops.** Return to where you started at the end. slam_toolbox recognizes the place and corrects the accumulated error.
- **Avoid moving objects** while mapping. Anything that moves gets drawn as a static obstacle.

Check the transform SLAM publishes while you drive:

```bash
$ ros2 run tf2_ros tf2_echo map odom
```

At the start it is (almost) zero. As odometry drifts, SLAM's correction grows.

---

## Saving the Map

When the map looks complete, save it with the Nav2 map saver while SLAM is still running:

```bash
$ ros2 run nav2_map_server map_saver_cli -f ~/ros2_ws/src/my_robot_navigation/maps/my_map \
    --ros-args -p use_sim_time:=true
[INFO] [map_saver]: Saving map from 'map' topic to '/home/utku/ros2_ws/src/my_robot_navigation/maps/my_map' file
[INFO] [map_saver]: Map saved successfully
```

Two files are created:

```text
maps/
├── my_map.pgm     ← the image: one pixel per cell
└── my_map.yaml    ← metadata
```

```yaml
image: my_map.pgm
mode: trinary
resolution: 0.05
origin: [-3.21, -3.14, 0]
negate: 0
occupied_thresh: 0.65
free_thresh: 0.25
```

| Field | Meaning |
|-------|---------|
| `image` | The image file, relative to the YAML file |
| `resolution` | Meters per pixel |
| `origin` | Position (x, y) and yaw of the image's **bottom-left** pixel in the `map` frame |
| `occupied_thresh`, `free_thresh` | Pixel darkness above which a cell counts as occupied, and below which it counts as free |
| `negate`, `mode` | How pixel values are interpreted; leave the defaults |

Open `my_map.pgm` with any image viewer: it is a grayscale picture of your room. You can even clean it up in an image editor, for example to remove a stray obstacle or close a gap.

Rebuild the package so the map is installed, and stop SLAM: from the next lesson on, the robot uses this saved map.

```bash
$ cd ~/ros2_ws
$ colcon build --packages-select my_robot_navigation --symlink-install
```

> 💡 slam_toolbox can also save its full internal state (*serialize* the pose graph), which lets you continue mapping later. Add the **SlamToolboxPlugin** panel in RViz2 (**Panels → Add New Panel**) to save and load it, or to switch slam_toolbox into localization mode.

---

## Common Problems

| Symptom | Cause / fix |
|---------|-------------|
| No `/map` topic, or slam_toolbox logs `Message Filter dropping message` repeatedly | Missing TF `odom → base_footprint` or `base_footprint → laser`; check with `tf2_echo` |
| The map stays empty in RViz2 | Fixed Frame is not `map`, or the Map display's durability doesn't match |
| Walls are doubled or bent | Driving too fast, or odometry is poor; drive slowly and close loops |
| `Failed to compute odom pose` or extrapolation errors | `use_sim_time` missing on slam_toolbox or other nodes |
| `map_saver_cli` times out | SLAM is not running, or `/map` isn't published yet |

---

## Try It Yourself

1. Map the room twice: once driving slowly, once fast with quick turns. Compare the two maps.
2. Let `scan_avoider` from the [Simulation Project](../08-simulation/simulation-project.md#step-3-the-scan_avoider-node) drive the robot while SLAM runs. Does it explore the room completely?
3. Open `my_map.pgm` and `my_map.yaml`. Using `origin` and `resolution`, compute the map coordinates of the image's top-right corner.
4. Add an obstacle to `my_world.sdf`, map the world again, and compare the maps.
5. Watch `tf2_echo map odom` for a few minutes of driving. How large does the correction get?

---

[⬆️ Navigation & SLAM](README.md) | [Next: Localization](localization.md) ➡️
