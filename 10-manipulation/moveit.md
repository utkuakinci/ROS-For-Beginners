# MoveIt 2

**MoveIt 2** is the motion planning framework of ROS 2. Given a goal ("move the gripper there" or "go to this joint configuration"), it solves inverse kinematics, plans a collision-free trajectory around the robot itself and obstacles in its environment, and sends the trajectory to the arm's controllers. In this lesson you plan motions interactively in RViz2 with the Panda arm, learn how MoveIt is organized, and write your first C++ program that moves the arm.

## Topics Covered

- What MoveIt does
- The `move_group` architecture
- Planning in RViz2 with the MotionPlanning plugin
- Planning groups and named poses
- The planning scene and collision objects
- Creating a configuration for your own robot: the Setup Assistant
- Your first MoveIt program in C++

## In Simple Terms

MoveIt is a **GPS for robot arms**. You say "put the hand there", and it works out how every joint must move, finding a route that doesn't hit the table, the robot itself, or anything else it knows about. The **planning scene** is MoveIt's knowledge of the furniture in the room.

**Example:** you drag the gripper marker in RViz2, click Plan, and MoveIt shows a smooth motion of all seven Panda joints to get there.

---

## What MoveIt Does

| Capability | Meaning |
|------------|---------|
| **Kinematics** | Forward and inverse kinematics for the arm, using a plugin (KDL by default) |
| **Motion planning** | Finding a collision-free path from the current state to the goal |
| **Collision checking** | Making sure the arm doesn't hit itself or objects in the **planning scene** |
| **Trajectory processing** | Adding timing that respects velocity and acceleration limits |
| **Execution** | Sending trajectories to the controllers and monitoring them |

You describe **what** you want; MoveIt works out **how**.

---

## Architecture

The central node is **`move_group`**. Everything else talks to it:

```text
  RViz2 MotionPlanning plugin ─┐
  Your C++ / Python program ───┼──▶ move_group ──▶ planners (OMPL, Pilz, ...)
  Command line / services ─────┘        │      ──▶ kinematics plugin (KDL, ...)
                                        │      ──▶ planning scene (robot + world)
                                        ▼
                         FollowJointTrajectory action
                                        ▼
                      ros2_control: joint_trajectory_controller ──▶ arm
```

`move_group` gets its knowledge about the robot from a **MoveIt configuration package** (for the Panda: `moveit_resources_panda_moveit_config`), which contains:

| File | Content |
|------|---------|
| URDF | The robot's links and joints |
| **SRDF** (Semantic Robot Description) | Planning groups, named poses, end effectors, and which link pairs never collide |
| `kinematics.yaml` | Which IK solver each group uses |
| `joint_limits.yaml` | Velocity and acceleration limits for planning |
| `ompl_planning.yaml` etc. | Planner settings |
| `moveit_controllers.yaml` | Which controllers execute trajectories |
| Launch files | `demo.launch.py` and others |

**Planners** are plugins. The default is **OMPL**, a library of sampling-based planners (such as RRTConnect) that find collision-free paths quickly in complex spaces. **Pilz** offers simple industrial motions: straight lines (LIN), point-to-point (PTP), and circles.

---

## Planning in RViz2

```bash
$ ros2 launch moveit_resources_panda_moveit_config demo.launch.py
```

This starts `move_group`, `robot_state_publisher`, `ros2_control` with **mock hardware** (simulated motors that follow commands instantly), and RViz2 with the **MotionPlanning** plugin.

In RViz2 you see the Panda twice:

- The **current state**: where the arm is.
- An orange **goal state** with an **interactive marker** (arrows and rings) at the gripper.

### Plan and execute

1. Drag the interactive marker's arrows and rings to move the goal. The orange arm follows: MoveIt solves IK live as you drag.
2. In the **MotionPlanning** panel → **Planning** tab, click **Plan**. A preview animation shows the planned trajectory.
3. Click **Execute**. The arm moves along the trajectory. (Or use **Plan & Execute**.)

### Planning groups and named poses

A **planning group** is a set of joints that is planned together. The Panda has:

| Group | Joints |
|-------|--------|
| `panda_arm` | The 7 arm joints |
| `hand` | The gripper fingers |
| `panda_arm_hand` | Both together |

Choose the group under **Planning Group**. In **Goal State**, pick a **named pose** stored in the SRDF, such as `ready` or `extended` for the arm, or `open` / `close` for the hand, then plan and execute.

> 💡 Enable **Query Goal State** / **Query Start State** and **Show Trail** in the panel's displays to see the motion more clearly. In the **Context** tab you can choose a different planner, for example from the Pilz pipeline.

---

## The Planning Scene

The **planning scene** is MoveIt's model of the world: the robot's current state plus all **collision objects** around it. Plans always avoid collisions in the scene.

Try it: in the **Scene Objects** tab, add a **Box**, move it between the arm and its goal with the marker, and click **Publish**. Plan again: MoveIt now plans around the box. Make the box so large that no path exists, and planning fails with a message in the panel and the terminal.

Collision objects come from three sources:

- Added by you (RViz2 or code), like the box above. You will do this from code in the [project](manipulation-project.md).
- Attached to the robot: an object held by the gripper moves with it and is checked for collisions too.
- Detected by sensors: MoveIt can build an occupancy map from depth cameras (the *Octomap*).

---

## Configuring Your Own Robot: the Setup Assistant

The Panda configuration was created with the **MoveIt Setup Assistant**, a graphical tool that generates a configuration package from a URDF:

```bash
$ ros2 launch moveit_setup_assistant setup_assistant.launch.py
```

| Step | What you define |
|------|----------------|
| Load URDF | Your robot's description (Xacro works too) |
| Self-Collisions | Generates a list of link pairs that can never collide, which speeds up planning |
| Virtual Joints | How the robot is attached to the world (e.g. fixed to `world`) |
| Planning Groups | Which joints belong to the arm, which to the gripper |
| Robot Poses | Named poses like `home` |
| End Effectors | Which group is the gripper and which link it is attached to |
| ros2_control / Controllers | The controllers that execute trajectories |
| Generate | Writes the `<robot>_moveit_config` package |

Try it with `my_arm` from the [previous lesson](robot-arms.md#building-my_arm): load `my_arm.urdf.xacro`, generate self-collisions, create a group `arm` with `joint1` to `joint3` and the `tool` link as the tip, add a `home` pose, and generate the package into `~/ros2_ws/src/my_arm_moveit_config`. After building, `ros2 launch my_arm_moveit_config demo.launch.py` lets you plan with your own arm. (With only 3 joints, the interactive marker can't reach arbitrary orientations; plan to joint-space goals or named poses instead.)

---

## Your First MoveIt Program

RViz2 is great for exploring, but applications move the arm from code. The **MoveGroupInterface** C++ class is the standard way to talk to `move_group`.

### Create the package

```bash
$ cd ~/ros2_ws/src
$ ros2 pkg create my_moveit_examples --build-type ament_cmake --license Apache-2.0 \
    --dependencies rclcpp moveit_ros_planning_interface --node-name hello_moveit
```

### Write the program

Replace `my_moveit_examples/src/hello_moveit.cpp` with:

```cpp
#include <memory>

#include <moveit/move_group_interface/move_group_interface.h>
#include <rclcpp/rclcpp.hpp>

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  auto const node = std::make_shared<rclcpp::Node>(
    "hello_moveit",
    rclcpp::NodeOptions().automatically_declare_parameters_from_overrides(true));
  auto const logger = rclcpp::get_logger("hello_moveit");

  // Connect to move_group for the arm's planning group
  using moveit::planning_interface::MoveGroupInterface;
  auto move_group_interface = MoveGroupInterface(node, "panda_arm");

  // The target pose for the end effector (panda_link8), in the planning frame (panda_link0)
  auto const target_pose = [] {
      geometry_msgs::msg::Pose msg;
      msg.orientation.w = 1.0;
      msg.position.x = 0.28;
      msg.position.y = -0.2;
      msg.position.z = 0.5;
      return msg;
    }();
  move_group_interface.setPoseTarget(target_pose);

  // Plan
  MoveGroupInterface::Plan plan;
  auto const success = static_cast<bool>(move_group_interface.plan(plan));

  // Execute
  if (success) {
    RCLCPP_INFO(logger, "Planning succeeded, executing");
    move_group_interface.execute(plan);
  } else {
    RCLCPP_ERROR(logger, "Planning failed!");
  }

  rclcpp::shutdown();
  return 0;
}
```

> 📝 Newer MoveIt versions also provide the header with a `.hpp` extension (`move_group_interface.hpp`). If you get a deprecation warning, switch to it.

### How it works

| Line | Meaning |
|------|---------|
| `automatically_declare_parameters_from_overrides(true)` | Lets MoveIt load the robot description parameters it needs |
| `MoveGroupInterface(node, "panda_arm")` | Connects to `move_group` and selects the planning group |
| `setPoseTarget(pose)` | Sets the goal for the group's end effector; MoveIt solves IK for you |
| `plan(plan)` | Asks `move_group` for a trajectory; returns an error code that converts to `bool` |
| `execute(plan)` | Sends the trajectory to the controllers and waits until the motion is done |

Other ways to set a goal:

```cpp
move_group_interface.setNamedTarget("ready");                    // a named pose from the SRDF
move_group_interface.setJointValueTarget(std::vector<double>{    // joint angles in radians
  0.0, -0.785, 0.0, -2.356, 0.0, 1.571, 0.785});
move_group_interface.move();                                    // plan and execute in one call
```

### Build and run

`ros2 pkg create` already set up `CMakeLists.txt` and `package.xml` with the dependencies. Build and run it while the demo is running:

```bash
$ cd ~/ros2_ws
$ colcon build --packages-select my_moveit_examples
```

```bash
$ ros2 launch moveit_resources_panda_moveit_config demo.launch.py     # Terminal 1
```

```bash
$ source ~/ros2_ws/install/setup.bash                                 # Terminal 2
$ ros2 run my_moveit_examples hello_moveit
[INFO] [hello_moveit]: Planning succeeded, executing
```

The arm moves to the target pose in RViz2. 🎉

> 💡 **Python:** MoveIt also offers a Python API, **`moveit_py`** (`ros-jazzy-moveit-py`). It needs a bit more configuration to start; see the [MoveIt documentation](https://moveit.picknik.ai/main/doc/examples/motion_planning_python_api/motion_planning_python_api_tutorial.html) when you are comfortable with the C++ interface.

---

## Common Problems

| Symptom | Cause / fix |
|---------|-------------|
| `Planning failed!` | The pose is unreachable, in collision, or the orientation is impossible. Try a pose you reached with the interactive marker |
| `Unable to find a valid IK solution` / `No motion plan found` | Same as above; also check that the pose is in the planning frame (`panda_link0`) |
| The program hangs at `MoveGroupInterface` | `move_group` isn't running, or robot description parameters can't be loaded |
| `execute` fails with a controller error | The controllers aren't running; check the demo's terminal |
| Build error `moveit/...: No such file` | `moveit_ros_planning_interface` missing in `find_package` / `ament_target_dependencies` |

---

## Try It Yourself

1. In RViz2, move the arm to `ready`, then to `extended`, and back. Try the Pilz `LIN` planner from the **Context** tab and compare the motion with OMPL.
2. Change `hello_moveit` to move to the named pose `ready` first, then to the target pose.
3. Send the arm through three poses in a row. What happens if one of them is unreachable?
4. Use `move_group_interface.getCurrentPose()` to log the end effector pose before and after the motion. (Hint: this needs a spinning node; see the MoveIt tutorials on the "MoveIt Visual Tools" step for how to spin in a separate thread.)
5. Create a MoveIt configuration for `my_arm` with the Setup Assistant and plan with it in RViz2.

---

⬅️ [Previous: Robot Arms](robot-arms.md) | [⬆️ Manipulation](README.md) | [Next: Manipulation Project](manipulation-project.md) ➡️
