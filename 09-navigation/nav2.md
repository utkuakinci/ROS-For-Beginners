# Nav2

**Nav2** is the navigation stack of ROS 2: a set of servers that together take a goal pose and drive the robot there, planning a path around known obstacles and reacting to new ones on the way. In this lesson you learn how Nav2 is organized, adapt its parameters to `my_robot`, start it with your own launch file, and send a first goal.

## Topics Covered

- The Nav2 architecture: servers and behavior trees
- Costmaps and their layers
- Planners and controllers
- Adapting the parameter file to your robot
- A launch file for navigation
- Your first navigation goal
- Reading Nav2 in RViz2

---

## Architecture

Nav2 is made of several **servers**, each a [lifecycle node](localization.md#lifecycle-nodes) with one job, coordinated by a **behavior tree**:

```text
                         NavigateToPose (action)
                                  │
                         ┌────────▼────────┐
                         │  BT Navigator   │  runs a behavior tree:
                         └──┬─────┬─────┬──┘  plan → follow → recover
              ComputePath   │     │     │   Spin / BackUp / Wait
             ┌──────────────▼┐  ┌─▼─────────────┐  ┌▼────────────────┐
             │ Planner Server│  │Controller Svr │  │ Behavior Server │
             │ global costmap│  │ local costmap │  │                 │
             └───────────────┘  └───────┬───────┘  └─────────────────┘
                                        │ cmd_vel
                               Velocity Smoother ──▶ /cmd_vel ──▶ robot
```

| Server | Job |
|--------|-----|
| **BT Navigator** | Receives navigation goals as actions and runs the **behavior tree** that decides what to do next |
| **Planner Server** | Computes a **global path** from the robot to the goal on the global costmap |
| **Controller Server** | Follows the path: computes velocity commands many times per second, avoiding obstacles on the local costmap |
| **Behavior Server** | Recovery behaviors when stuck: spin in place, back up, wait |
| **Smoother Server** | Smooths planned paths |
| **Velocity Smoother** | Limits acceleration so commands are smooth for the motors |
| **Waypoint Follower** | Drives through a list of waypoints one by one |
| **Lifecycle Manager** | Starts all the servers in the right order |

### Behavior trees

A **behavior tree** describes the navigation logic as a tree of small actions and conditions. The default tree does roughly this:

```text
Navigate with recovery
├── Repeat until goal reached:
│     ├── Compute a path to the goal (replan about once per second)
│     └── Follow the path
└── If that fails: clear costmaps → spin → wait → back up → try again
```

Because the logic is a tree in an XML file and not hard-coded, you can change Nav2's behavior (replan more often, add a different recovery) without touching C++ code. For this guide, the default tree is all you need.

---

## Costmaps

A **costmap** is a grid like the occupancy map, but each cell holds a **cost** (0–254) for the robot to be there. Nav2 keeps two:

| Costmap | Covers | Used by | Updated from |
|---------|--------|---------|--------------|
| **Global costmap** | The whole map | Planner | The saved map plus sensor data |
| **Local costmap** | A small window (e.g. 3 m × 3 m) that moves with the robot | Controller | Sensor data |

Each costmap is built from **layers**:

| Layer | Adds |
|-------|------|
| **Static layer** | The obstacles of the saved map |
| **Obstacle layer** (or voxel layer) | Obstacles currently seen by the laser, including ones not on the map |
| **Inflation layer** | A cost gradient around every obstacle, so paths keep a safe distance |

```text
  wall │███│▓▓▓│▒▒▒│░░░│   │   │
       lethal  inscribed  decreasing cost   free
```

Cells closer than the robot's radius to an obstacle are **lethal**: the robot's center can't go there without its body hitting something. The inflation layer then adds decreasing cost further out, so the planner prefers paths through the middle of free space.

---

## Planners and Controllers

Both are **plugins**: you choose them in the parameter file.

| Planners (global path) | Characteristics |
|------------------------|-----------------|
| **NavFn** | Classic grid search (Dijkstra / A*). Simple, fast, good for round robots |
| **Smac Planner 2D** | Modern A*-based grid planner |
| **Smac Hybrid-A\*** | Respects turning limits; for car-like robots |
| **Theta\*** | Any-angle paths with fewer zig-zags |

| Controllers (path following) | Characteristics |
|------------------------------|-----------------|
| **DWB** | Samples many velocity commands, scores each with configurable critics |
| **Regulated Pure Pursuit** | Follows the path like a car "chasing a point" ahead; simple and robust |
| **MPPI** | Optimizes over many predicted trajectories; smooth and capable, more CPU |

The defaults in `nav2_params.yaml` work for a differential-drive robot like `my_robot`. Check which ones are configured under `planner_server` → `GridBased` → `plugin` and `controller_server` → `FollowPath` → `plugin`.

---

## Adapting the Parameters to `my_robot`

Open `~/ros2_ws/src/my_robot_navigation/config/nav2_params.yaml` (copied in the [Localization](localization.md#your-nav2-parameter-file) lesson). The defaults are tuned for a different, round robot. A few settings must match yours.

### 1. The robot's footprint

`my_robot` is rectangular, so describe its outline as a polygon instead of a radius. In **both** `local_costmap` and `global_costmap`, replace the `robot_radius` line with a `footprint`:

```yaml
local_costmap:
  local_costmap:
    ros__parameters:
      ...
      # robot_radius: 0.22
      footprint: "[[0.24, 0.21], [0.24, -0.21], [-0.21, -0.21], [-0.21, 0.21]]"
```

The points are corners in meters, in the `base_link` frame: the chassis is 0.4 × 0.3 m, the wheels stick out to ±0.2 m, the camera reaches 0.23 m forward, plus a small safety margin.

### 2. Inflation

In both costmaps, the `inflation_layer` controls how far the planner stays from obstacles:

```yaml
inflation_layer:
  plugin: "nav2_costmap_2d::InflationLayer"
  cost_scaling_factor: 3.0
  inflation_radius: 0.55
```

`inflation_radius` must be larger than the robot's circumscribed radius (about 0.3 m for `my_robot`). Larger values keep the robot further from walls, but in a small room they can close narrow passages entirely. For the 6 × 6 m room, the default is fine.

### 3. Speed limits

Keep the robot slow while learning. In `velocity_smoother`:

```yaml
velocity_smoother:
  ros__parameters:
    max_velocity: [0.3, 0.0, 1.0]       # [x m/s, y m/s, theta rad/s]
    min_velocity: [-0.3, 0.0, -1.0]
```

The controller has its own speed settings (for example `vx_max` for MPPI, `max_vel_x` for DWB, `desired_linear_vel` for Regulated Pure Pursuit). Lower them to the same value.

### 4. Frames and topics

The defaults already match `my_robot`, but it is worth knowing where they are:

| Setting | Default | Where |
|---------|---------|-------|
| `robot_base_frame` | `base_link` | costmaps, `bt_navigator`, `behavior_server` |
| `global_frame` | `map` (global costmap), `odom` (local costmap) | costmaps |
| `odom_topic` | `/odom` | `bt_navigator`, `velocity_smoother` |
| laser source `topic` | `/scan` | `obstacle_layer` / `voxel_layer` → `scan` |

> 📝 Nav2 in Jazzy publishes `geometry_msgs/msg/Twist` on `/cmd_vel` by default, which matches the bridge from [Module 08](../08-simulation/spawning-robots.md#the-bridge-configuration). Newer distributions switch to `TwistStamped`; if your robot doesn't move, compare `ros2 topic info /cmd_vel --verbose` on both sides.

---

## A Launch File for Navigation

`nav2_bringup` provides `bringup_launch.py`, which starts localization (map server + AMCL) **and** all navigation servers. Wrap it in your own launch file so the map and parameters are filled in automatically. Create `~/ros2_ws/src/my_robot_navigation/launch/navigation.launch.py`:

```python
import os

from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument, IncludeLaunchDescription
from launch.conditions import IfCondition
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.substitutions import LaunchConfiguration
from launch_ros.actions import Node


def generate_launch_description():
    nav_share = get_package_share_directory('my_robot_navigation')
    bringup_share = get_package_share_directory('nav2_bringup')

    map_file = LaunchConfiguration('map')
    params_file = LaunchConfiguration('params_file')
    use_rviz = LaunchConfiguration('use_rviz')

    return LaunchDescription([
        DeclareLaunchArgument(
            'map', default_value=os.path.join(nav_share, 'maps', 'my_map.yaml'),
            description='Full path to the map YAML file'),
        DeclareLaunchArgument(
            'params_file', default_value=os.path.join(nav_share, 'config', 'nav2_params.yaml'),
            description='Full path to the Nav2 parameter file'),
        DeclareLaunchArgument('use_rviz', default_value='true', description='Start RViz2'),

        IncludeLaunchDescription(
            PythonLaunchDescriptionSource(
                os.path.join(bringup_share, 'launch', 'bringup_launch.py')),
            launch_arguments={
                'map': map_file,
                'params_file': params_file,
                'use_sim_time': 'true',
                'autostart': 'true',
            }.items(),
        ),

        Node(
            package='rviz2',
            executable='rviz2',
            arguments=['-d', os.path.join(bringup_share, 'rviz', 'nav2_default_view.rviz')],
            parameters=[{'use_sim_time': True}],
            condition=IfCondition(use_rviz),
        ),
    ])
```

Add the dependencies to `package.xml`, then rebuild:

```xml
<exec_depend>nav2_bringup</exec_depend>
<exec_depend>slam_toolbox</exec_depend>
<exec_depend>rviz2</exec_depend>
```

```bash
$ cd ~/ros2_ws
$ colcon build --packages-select my_robot_navigation --symlink-install
```

---

## Your First Goal

**Terminal 1**, the simulation:

```bash
$ ros2 launch my_robot_gazebo sim.launch.py use_rviz:=false
```

**Terminal 2**, navigation:

```bash
$ ros2 launch my_robot_navigation navigation.launch.py
...
[lifecycle_manager_navigation]: Managed nodes are active
```

In RViz2:

1. Set the initial pose with **2D Pose Estimate** (unless you configured `set_initial_pose`).
2. Click **Nav2 Goal** in the toolbar, then click a free spot on the map and drag to set the final orientation.

The robot plans a path (a line from the robot to the goal), starts driving, and stops at the goal facing the chosen direction. 🎉

### Reading Nav2 in RViz2

The `nav2_default_view.rviz` configuration shows:

| Display | What you see |
|---------|--------------|
| **Map** | The saved map |
| **Global Costmap** | Inflated obstacles over the whole map (colored halo around walls) |
| **Local Costmap** | The small window around the robot, updated from the laser |
| **Path** | The global plan from the planner |
| **Amcl Particle Swarm** | AMCL's particles |
| **Navigation 2** panel | Navigation status, a cancel button, and waypoint mode |

### Obstacles that aren't on the map

Insert a box in Gazebo (toolbar → box) in the robot's path while it drives. The laser sees it, the obstacle layer marks it in both costmaps, and the planner routes around it. This is the difference from a robot that only follows the map.

---

## Navigating While Mapping

Nav2 also works without a saved map, with SLAM running instead of AMCL. `bringup_launch.py` supports this directly:

```bash
$ ros2 launch nav2_bringup bringup_launch.py slam:=True use_sim_time:=true \
    params_file:=$HOME/ros2_ws/src/my_robot_navigation/config/nav2_params.yaml
```

Now you can send goals into unexplored areas: the map grows as the robot drives. This is how robots explore new environments autonomously.

---

## Common Problems

| Symptom | Cause / fix |
|---------|-------------|
| `Managed nodes are active` never appears | A server failed to configure. Scroll up to the first error, often a typo in the YAML |
| Goal rejected or `Failed to create plan` | The goal or the robot's pose is inside an obstacle or inflated area, or outside the map |
| The robot plans a path but doesn't move | `/cmd_vel` doesn't reach Gazebo: check `ros2 topic echo /cmd_vel` and the bridge |
| The robot oscillates or stops near walls | Footprint or inflation too large for the space, or speeds too high |
| Repeated `Spin` / `BackUp` recoveries | The controller can't follow the path; check footprint, speed limits, and localization quality |
| TF or timing errors | A node without `use_sim_time`, or the CPU is overloaded (check the Real Time Factor in Gazebo) |

---

## Try It Yourself

1. Change `inflation_radius` to 1.0 in the global costmap and look at the costmap in RViz2. Can the robot still reach every part of the room?
2. Switch the controller plugin (for example from MPPI to Regulated Pure Pursuit, using the configuration from the [Nav2 documentation](https://docs.nav2.org/configuration/index.html)) and compare how the robot drives.
3. Send a goal into the box obstacle. What does Nav2 report?
4. Navigate while mapping with `slam:=True` and explore the whole room with Nav2 goals only.
5. Watch the behavior tree's progress: `ros2 topic echo /behavior_tree_log` while the robot navigates.

---

⬅️ [Previous: Localization](localization.md) | [⬆️ Navigation & SLAM](README.md) | [Next: Autonomous Navigation](autonomous-navigation.md) ➡️
