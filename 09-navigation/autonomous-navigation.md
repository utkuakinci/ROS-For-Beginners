# Autonomous Navigation

Nav2 is running and the robot reaches goals you click in RViz2. A real application, though, sends goals from **code**: a delivery robot drives to a list of rooms, a security robot patrols a route, a warehouse robot follows orders from a server. In this lesson you send goals from RViz2, the command line, and Python, and write a `patrol` node that drives `my_robot` through a series of waypoints on its own.

## Topics Covered

- Waypoints in RViz2
- Nav2's navigation actions
- Sending goals from the command line
- The Nav2 Simple Commander Python API
- Following waypoints and handling results
- Writing a patrol node

All examples assume the simulation and navigation from the [previous lesson](nav2.md#your-first-goal) are running, with the initial pose set:

```bash
$ ros2 launch my_robot_gazebo sim.launch.py use_rviz:=false
$ ros2 launch my_robot_navigation navigation.launch.py
```

---

## Waypoints in RViz2

The **Navigation 2** panel in RViz2 can queue several goals:

1. In the panel, click **Waypoint / Nav Through Poses Mode**.
2. Use **Nav2 Goal** to place several poses on the map. Each one appears as an arrow; nothing moves yet.
3. Click **Start Waypoint Following** (stops at each waypoint) or **Start Nav Through Poses** (drives through them without stopping).

This is handy for trying out a route before you write it in code.

---

## Nav2's Navigation Actions

Every way of sending goals ends up calling one of Nav2's [actions](../01-ros2-basics/actions.md):

```bash
$ ros2 action list
/navigate_to_pose
/navigate_through_poses
/follow_waypoints
...
```

| Action | Type | Behavior |
|--------|------|----------|
| `/navigate_to_pose` | `nav2_msgs/action/NavigateToPose` | Drive to one goal pose |
| `/navigate_through_poses` | `nav2_msgs/action/NavigateThroughPoses` | Drive through several poses without stopping, planned as one route |
| `/follow_waypoints` | `nav2_msgs/action/FollowWaypoints` | Navigate to each waypoint in turn, stopping at each one |

Because they are actions, you get **feedback** while the robot drives (distance remaining, current waypoint) and can **cancel** at any time, exactly as you learned in [Module 01](../01-ros2-basics/actions.md).

---

## Sending a Goal from the Command Line

```bash
$ ros2 action send_goal --feedback /navigate_to_pose nav2_msgs/action/NavigateToPose \
    "{pose: {header: {frame_id: map}, pose: {position: {x: 2.0, y: -2.0}, orientation: {w: 1.0}}}}"
Waiting for an action server to become available...
Sending goal:
     pose:
       header:
         frame_id: map
...
Goal accepted with ID: 8c0c...

Feedback:
    current_pose: ...
    navigation_time: ...
    estimated_time_remaining: ...
    number_of_recoveries: 0
    distance_remaining: 2.71
...
Result:
    ...
Goal finished with status: SUCCEEDED
```

The goal must be in the `map` frame. Orientation `{w: 1.0}` means facing along the map's x axis; for other directions, use the yaw-to-quaternion formula from [Coordinate Frames](../06-tf2/coordinate-frames.md#quaternions).

---

## The Simple Commander API

Writing an action client for Nav2 by hand is possible (see [Actions in Python](../04-python/actions.md)), but Nav2 provides a much simpler Python interface: **`nav2_simple_commander`**. Its `BasicNavigator` class wraps all the actions in plain method calls:

| Method | Purpose |
|--------|---------|
| `setInitialPose(pose)` | Set AMCL's initial pose (instead of 2D Pose Estimate) |
| `waitUntilNav2Active()` | Block until localization and navigation are ready |
| `goToPose(pose)` | Start navigating to one pose |
| `goThroughPoses(poses)` | Navigate through several poses |
| `followWaypoints(poses)` | Visit waypoints one by one |
| `isTaskComplete()` | `True` once the current task has finished |
| `getFeedback()` | The latest feedback of the current task |
| `getResult()` | `TaskResult.SUCCEEDED`, `CANCELED`, or `FAILED` |
| `cancelTask()` | Cancel the current task |

The pattern is always the same: **start a task, poll until it is complete, check the result.**

---

## Writing a Patrol Node

The patrol drives around the room's four corners, over and over. With the room from [Module 08](../08-simulation/gazebo.md#writing-a-world-file) (walls at ±3 m, obstacles near (1.5, 1.0) and (−1.2, −1.0)) and the map starting at the spawn position, the points (±2, ±2) are free and safely away from walls.

Create `~/ros2_ws/src/my_first_pkg/my_first_pkg/patrol.py`:

```python
import math

import rclpy
from geometry_msgs.msg import PoseStamped
from nav2_simple_commander.robot_navigator import BasicNavigator, TaskResult

PATROL_POINTS = [  # (x, y, yaw) in the map frame
    (2.0, -2.0, math.pi / 2),
    (2.0, 2.0, math.pi),
    (-2.0, 2.0, -math.pi / 2),
    (-2.0, -2.0, 0.0),
]
LAPS = 2


def make_pose(navigator, x, y, yaw):
    pose = PoseStamped()
    pose.header.frame_id = 'map'
    pose.header.stamp = navigator.get_clock().now().to_msg()
    pose.pose.position.x = x
    pose.pose.position.y = y
    pose.pose.orientation.z = math.sin(yaw / 2.0)
    pose.pose.orientation.w = math.cos(yaw / 2.0)
    return pose


def main():
    rclpy.init()
    navigator = BasicNavigator()

    navigator.setInitialPose(make_pose(navigator, 0.0, 0.0, 0.0))
    navigator.waitUntilNav2Active()

    waypoints = [make_pose(navigator, x, y, yaw) for x, y, yaw in PATROL_POINTS]

    for lap in range(1, LAPS + 1):
        navigator.get_logger().info(f'Starting lap {lap} of {LAPS}')
        navigator.followWaypoints(waypoints)

        last_waypoint = -1
        while not navigator.isTaskComplete():
            feedback = navigator.getFeedback()
            if feedback and feedback.current_waypoint != last_waypoint:
                last_waypoint = feedback.current_waypoint
                navigator.get_logger().info(
                    f'Heading to waypoint {last_waypoint + 1} of {len(waypoints)}')

        result = navigator.getResult()
        if result == TaskResult.SUCCEEDED:
            navigator.get_logger().info(f'Lap {lap} complete')
        elif result == TaskResult.CANCELED:
            navigator.get_logger().warn('Patrol was canceled')
            break
        else:
            navigator.get_logger().error('Patrol failed')
            break

    navigator.get_logger().info('Patrol finished')
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

### How it works

- **`BasicNavigator()`** is a node. It creates the action clients and helpers for you.
- **`setInitialPose`** replaces the manual 2D Pose Estimate. The robot spawns at the map's origin, so `(0, 0, 0)` is correct. Remove this line if you already set the pose in RViz2 or in the parameter file.
- **`waitUntilNav2Active()`** waits until AMCL and the BT navigator are active, so the first goal isn't sent too early.
- **`followWaypoints`** starts the task and returns immediately. The `while not navigator.isTaskComplete()` loop polls until it is done; `isTaskComplete` also processes incoming messages, which is why no `spin` is needed.
- **`getFeedback()`** returns the action's feedback message. For `FollowWaypoints`, `current_waypoint` is the index of the waypoint being approached.
- **`getResult()`** tells you how the task ended. Always handle failure: a blocked path or a goal inside an obstacle is normal in the real world.

### Run it

Register `'patrol = my_first_pkg.patrol:main'` in `setup.py` and add `<depend>nav2_simple_commander</depend>` to `package.xml`. Rebuild, then run it with simulation time:

```bash
$ ros2 run my_first_pkg patrol --ros-args -p use_sim_time:=true
[INFO] [basic_navigator]: Publishing Initial Pose
[INFO] [basic_navigator]: Waiting for amcl_pose to be received
[INFO] [basic_navigator]: Nav2 is ready for use!
[INFO] [basic_navigator]: Starting lap 1 of 2
[INFO] [basic_navigator]: Heading to waypoint 1 of 4
...
[INFO] [basic_navigator]: Lap 1 complete
```

The robot drives the patrol route twice. While it runs, drop a box into its path in Gazebo and watch it replan. Cancel the patrol from RViz2's **Navigation 2** panel and see how the node handles `CANCELED`.

---

## Going Further

Nav2 can do much more than this module covers. Some features to explore in the [Nav2 documentation](https://docs.nav2.org/):

| Feature | What it does |
|---------|--------------|
| **Keepout zones** | Mark areas on a separate map that the robot must never enter |
| **Speed zones** | Areas with lower speed limits |
| **Collision Monitor** | An extra safety layer that slows or stops the robot when something gets too close |
| **Docking** | Precisely drive onto a charging dock |
| **Custom behavior trees** | Change how the robot reacts, e.g. different recoveries |
| **Multiple robots** | Run several robots with namespaces |

---

## Common Problems

| Symptom | Cause / fix |
|---------|-------------|
| `waitUntilNav2Active` never returns | Nav2 or AMCL isn't active, or the node lacks `use_sim_time:=true` |
| A waypoint fails | The point is inside an obstacle or its inflation; move it into free space (check the costmap in RViz2) |
| The robot reaches waypoints but at a wrong angle | Yaw/quaternion mistake; remember `z = sin(yaw/2)`, `w = cos(yaw/2)` |
| `ModuleNotFoundError: nav2_simple_commander` | Install `ros-jazzy-nav2-simple-commander` |
| Goals in the wrong place | The map's origin is not where you expected; check positions by clicking **Publish Point** in RViz2 and echoing `/clicked_point` |

---

## Try It Yourself

1. Change `patrol` to use `goThroughPoses` instead of `followWaypoints`. How does the robot's motion change?
2. Use RViz2's **Publish Point** tool and `ros2 topic echo /clicked_point` to pick your own patrol points, then update `PATROL_POINTS`.
3. Make the patrol endless, and stop it cleanly with `Ctrl` + `C` (cancel the task before exiting).
4. Print `distance_remaining` from the feedback while using `goToPose` for each point instead of `followWaypoints`.
5. Write a "delivery" node: it reads a room name from a parameter (`kitchen`, `office`, ...), looks up the coordinates in a dictionary, drives there, waits 5 seconds, and returns to `(0, 0)`.

---

⬅️ [Previous: Nav2](nav2.md) | [⬆️ Navigation & SLAM](README.md)
