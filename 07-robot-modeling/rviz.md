# RViz2

**RViz2** is the 3D visualization tool of ROS 2. It shows what the robot *knows*: its model, its frames, sensor data, maps, planned paths. It is not a simulator; it only displays data published by other nodes. In this lesson you learn the interface, set up a view for `my_robot`, save it so the launch file loads it automatically, and display sensor data from a small fake laser scanner.

## Topics Covered

- The RViz2 interface
- The Fixed Frame and Global Status
- Common displays
- Saving and loading configurations
- Displaying sensor data
- Views and tools

---

## The Interface

Start the robot from the previous lesson:

```bash
$ ros2 launch my_robot_description display.launch.py
```

```text
┌──────────────┬────────────────────────────────────────┬──────────────┐
│  Displays    │                                        │  Views       │
│  ─────────   │                                        │              │
│  ☑ Global    │             3D view                    │  Orbit       │
│    Options   │                                        │  (current)   │
│  ☑ Grid      │                                        │              │
│  ☑ RobotModel│                                        │              │
│  ☑ TF        │                                        │              │
│              │                                        │              │
│  [Add] ...   │                                        │              │
├──────────────┴────────────────────────────────────────┴──────────────┤
│  Time: ROS Time ... Wall Time ...                       [Reset]      │
└──────────────────────────────────────────────────────────────────────┘
       Toolbar (top): Interact · Move Camera · Select · Focus · Measure ·
                      2D Pose Estimate · 2D Goal Pose · Publish Point
```

| Area | Purpose |
|------|---------|
| **Displays** panel (left) | The list of things being drawn. Each display has settings and a status |
| **3D view** (center) | The scene |
| **Views** panel (right) | Camera type and position |
| **Toolbar** (top) | Mouse tools: move the camera, select, measure, send poses |
| **Time** panel (bottom) | ROS time and wall time; **Reset** clears all displays |

Camera controls in the default **Orbit** view: left-drag rotates, middle-drag (or `Shift` + left-drag) pans, scroll zooms.

---

## Fixed Frame and Global Status

**Global Options → Fixed Frame** is the most important setting in RViz2. All data is transformed into this frame before it is drawn, using TF.

| Fixed Frame | Use when |
|-------------|----------|
| `base_footprint` / `base_link` | Looking at the robot model itself. The robot stays still in the view |
| `odom` | Watching the robot move relative to where it started |
| `map` | Navigation: the robot moves on the map |

If the Fixed Frame doesn't exist in the TF tree, nothing can be drawn. **Global Status** at the top of the Displays panel then shows an error such as `Fixed Frame [map] does not exist`. Expand any display with a warning or error to read its status messages; they usually say exactly what is missing (a topic, a transform, a frame).

---

## Setting Up a View for `my_robot`

1. **Fixed Frame:** type `base_footprint`.
2. **Add → RobotModel.** Set **Description Topic** to `/robot_description`. The robot appears.
3. **Add → TF.** Enable **Show Names**. Reduce **Marker Scale** if the axes hide the robot.
4. In the RobotModel display, set **Alpha** to `0.5` to see the frames inside the chassis.
5. Move the wheel sliders in the Joint State Publisher window and watch the wheel frames rotate.

### Saving the configuration

**File → Save Config As**, and save to `~/ros2_ws/src/my_robot_description/rviz/display.rviz`. The file stores every display, its settings, and the camera position.

Load it automatically from the launch file by adding the `-d` argument to the RViz2 node in `launch/display.launch.py`:

```python
rviz_config = os.path.join(pkg_share, 'rviz', 'display.rviz')

Node(
    package='rviz2',
    executable='rviz2',
    arguments=['-d', rviz_config],
),
```

Rebuild the package so the new file is installed, then launch again: RViz2 opens with your view already set up.

> 💡 `.rviz` files are plain YAML. Keep them in the package and in Git, like any other configuration. When you change the view, save it again with **File → Save Config** (`Ctrl` + `S`).

---

## Common Displays

**Add** → **By display type** lists every display; **By topic** lists only the topics currently available with a matching display, which is often faster.

| Display | Shows | Message type |
|---------|-------|--------------|
| **Grid** | A reference grid on the ground | — |
| **RobotModel** | The robot from its URDF | `/robot_description` + TF |
| **TF** | Frames as axes (red = x, green = y, blue = z) | TF |
| **Axes** | One frame's axes | TF |
| **LaserScan** | 2D laser points | `sensor_msgs/msg/LaserScan` |
| **PointCloud2** | 3D points from depth cameras or 3D LiDAR | `sensor_msgs/msg/PointCloud2` |
| **Image** | A camera image in a panel | `sensor_msgs/msg/Image` |
| **Map** | An occupancy grid map | `nav_msgs/msg/OccupancyGrid` |
| **Path** | A planned path | `nav_msgs/msg/Path` |
| **Odometry** | Odometry poses as arrows | `nav_msgs/msg/Odometry` |
| **Pose** / **PoseArray** | One or many poses | `geometry_msgs/msg/PoseStamped` / `PoseArray` |
| **Marker** / **MarkerArray** | Custom shapes, text, and arrows from your own nodes | `visualization_msgs/msg/Marker` / `MarkerArray` |

Every display of sensor data needs two things: messages on its topic, **and** a TF path from the message's `header.frame_id` to the Fixed Frame.

---

## Displaying Sensor Data: A Fake Laser

`my_robot` has a `laser` link but no real sensor yet; that comes with simulation in [Module 08](../08-simulation/sensors.md). To practice, write a node that publishes a made-up scan in the `laser` frame.

Create `~/ros2_ws/src/my_first_pkg/my_first_pkg/fake_scan.py`:

```python
import math

import rclpy
from rclpy.node import Node
from sensor_msgs.msg import LaserScan


class FakeScan(Node):
    def __init__(self):
        super().__init__('fake_scan')
        self.publisher = self.create_publisher(LaserScan, 'scan', 10)
        self.timer = self.create_timer(0.1, self.publish_scan)
        self.phase = 0.0

    def publish_scan(self):
        scan = LaserScan()
        scan.header.stamp = self.get_clock().now().to_msg()
        scan.header.frame_id = 'laser'
        scan.angle_min = -math.pi
        scan.angle_max = math.pi
        scan.angle_increment = math.radians(1.0)
        scan.range_min = 0.1
        scan.range_max = 10.0

        count = int((scan.angle_max - scan.angle_min) / scan.angle_increment)
        # A wavy "wall" around the robot that slowly rotates
        scan.ranges = [
            1.5 + 0.3 * math.sin(4 * (scan.angle_min + i * scan.angle_increment) + self.phase)
            for i in range(count)
        ]
        self.phase += 0.05
        self.publisher.publish(scan)


def main():
    rclpy.init()
    node = FakeScan()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.try_shutdown()


if __name__ == '__main__':
    main()
```

The important line is `scan.header.frame_id = 'laser'`: it tells every consumer, including RViz2, that these distances are measured **from the `laser` frame**. Register `'fake_scan = my_first_pkg.fake_scan:main'`, add `<depend>sensor_msgs</depend>` to `package.xml`, and rebuild.

With `display.launch.py` running:

```bash
$ ros2 run my_first_pkg fake_scan
```

In RViz2, **Add → By topic → /scan → LaserScan**. Set **Size (m)** to `0.03` and **Color Transformer** to `FlatColor`. A wavy ring of points appears around the robot, centered on the laser. Now change the laser's position in the Xacro file (for example `xyz="0.15 0 ..."`), restart the launch, and watch the whole scan move with it: RViz2 places the data using TF.

> 📝 Sensor drivers often publish with **best effort** QoS. If a LaserScan display shows nothing although the topic is active, change the display's **Topic → Reliability Policy** to `Best Effort`.

---

## Views

The **Views** panel switches between camera types:

| View | Use |
|------|-----|
| **Orbit** | Rotate around a focal point. The default, good for robots |
| **TopDownOrtho** | A 2D map view from above, without perspective. Good for navigation |
| **FPS** | First-person camera that moves freely |
| **ThirdPersonFollower** | Follows a frame, for example the robot as it drives |

You can save several views in the panel and switch between them.

---

## Tools

| Tool | Shortcut | What it does |
|------|----------|--------------|
| **Interact** | `I` | Interact with interactive markers (used by MoveIt 2) |
| **Move Camera** | `M` | Rotate, pan, and zoom the view |
| **Select** | `S` | Select objects to inspect their properties |
| **Focus Camera** | `C` | Center the view on a clicked point |
| **Measure** | `N` | Measure the distance between two clicked points |
| **2D Pose Estimate** | `P` | Publish an initial pose on `/initialpose` (used by localization) |
| **2D Goal Pose** | `G` | Publish a goal on `/goal_pose` (used by Nav2) |
| **Publish Point** | `U` | Publish a clicked point on `/clicked_point` |

The last three publish messages: RViz2 is not only a viewer but also a simple way to send poses to your nodes. Try it: run `ros2 topic echo /clicked_point`, choose **Publish Point**, and click on the grid.

---

## Common Problems

| Symptom | Cause |
|---------|-------|
| `Fixed Frame [x] does not exist` | Wrong frame name, or nothing publishes TF for it yet |
| RobotModel: `No transform from [link] to [frame]` | Missing `/joint_states` or `robot_state_publisher` |
| RobotModel shows nothing, no error | Description Topic not set to `/robot_description` |
| LaserScan / PointCloud2 empty although the topic publishes | QoS mismatch (try Best Effort), or no TF from the data's `frame_id` to the Fixed Frame |
| `Message removed because it is too old` / `queue is full` | Timestamps don't match TF time; check `use_sim_time` and system clocks |
| Everything is too slow | Large point clouds or images; reduce the display's queue size or decay time |

---

## Try It Yourself

1. Change the fake laser's ranges to describe a square room (2 m × 2 m) around the robot.
2. Add an **Axes** display for the `laser` frame and a **Grid** with a 0.1 m cell size. Save the configuration.
3. Use **Measure** to check the distance from the laser to the scan points, and compare it with the ranges you published.
4. Switch to the **TopDownOrtho** view and save a second view in the Views panel.
5. Write a node that publishes a `visualization_msgs/msg/Marker` of type `TEXT_VIEW_FACING` showing the robot's name above `base_link`, and display it in RViz2.

---

⬅️ [Previous: robot_state_publisher](robot-state-publisher.md) | [⬆️ Robot Modeling](README.md)
