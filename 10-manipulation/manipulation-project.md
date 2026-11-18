# Manipulation Project

In this project you write `pick_place`, a C++ program that makes the Panda arm pick up a box from one side of a table and place it on the other. It brings together everything from this module: the planning scene, pose and named targets, the gripper, and attaching objects to the robot. The structure of the program is the same one used in real pick-and-place applications.

## Topics Covered

- Adding collision objects to the planning scene from code
- Planning with two groups: arm and gripper
- Grasp poses and approach motions
- Attaching and detaching objects
- Structuring a manipulation task as a sequence of steps
- Handling failures

## In Simple Terms

Picking up an object is the same sequence you use for a **cup on a table**: hover your hand above it, lower it, close your fingers, lift, carry, lower, open, and pull back. **Attaching** the object tells MoveIt "the cup is now part of my hand", so it is carried along and doesn't get knocked into things.

**Example:** `pick_place` runs these steps one after another, and stops with a clear message at the step that fails.

---

## The Task

```text
   side view                              top view (y to the left)
                                          ┌─────────────── table ────────────────┐
        pre-grasp ↓   pre-place ↓         │   place ▢  (0.5, 0.25)                │
          ▼             ▼                 │                                      │
   ┌──────▢─────────────────────┐         │   pick  ■  (0.5, -0.25)               │
   │        table               │         └──────────────────────────────────────┘
   └────────────────────────────┘                       ▲ robot at the origin
```

| Step | Motion | Group |
|------|--------|-------|
| 1 | Add the table and the box to the planning scene | — |
| 2 | Move to the `ready` pose, open the gripper | arm, hand |
| 3 | Move above the box (**pre-grasp**) | arm |
| 4 | Move down to the **grasp** pose | arm |
| 5 | Attach the box to the gripper, close the gripper | hand |
| 6 | Lift, move above the place position (**pre-place**), lower | arm |
| 7 | Open the gripper, detach the box | hand |
| 8 | Retreat upward, return to `ready` | arm |

Approaching from above in two steps (pre-grasp, then straight down) is the standard pattern: the final approach is short and predictable, and the gripper doesn't sweep sideways into the object.

---

## Grasp Geometry

MoveIt plans for the end effector link of the `panda_arm` group, **`panda_link8`** (the flange). The point that matters for grasping is between the fingertips, **0.1034 m** further along the flange's z axis. So to put the fingertips around a point, the flange must be 0.1034 m above it when the gripper points down.

```text
      panda_link8 (flange)   ── z = grasp height + 0.1034
           │
        ┌──┴──┐ panda_hand
        │     │
        ▌     ▐  fingers
           ●     fingertip center = grasp height
```

**Orientation:** for the gripper to point straight down, the flange is rotated 180° around x (roll = π). An extra yaw of −π/4 compensates for the 45° rotation between the flange and the hand, so the fingers close along the world y axis, across the box. In quaternion form, roll = π and yaw = −π/4 give `(x, y, z, w) = (0.924, −0.383, 0, 0)`.

> 💡 You can verify a grasp pose interactively first: drag the interactive marker in RViz2 to the pose, then read it with `ros2 run tf2_ros tf2_echo panda_link0 panda_link8`.

---

## The Program

Add a new file `~/ros2_ws/src/my_moveit_examples/src/pick_place.cpp`:

```cpp
#include <memory>
#include <string>

#include <moveit/move_group_interface/move_group_interface.h>
#include <moveit/planning_scene_interface/planning_scene_interface.h>
#include <rclcpp/rclcpp.hpp>

using moveit::planning_interface::MoveGroupInterface;
using moveit::planning_interface::PlanningSceneInterface;

namespace
{
constexpr double kTableHeight = 0.2;      // top of the table
constexpr double kBoxSize = 0.04;         // the box is a 4 cm cube
constexpr double kTcpOffset = 0.1034;     // panda_link8 to fingertip center
constexpr double kApproach = 0.10;        // pre-grasp distance above the grasp
constexpr double kPickX = 0.5, kPickY = -0.25;
constexpr double kPlaceX = 0.5, kPlaceY = 0.25;

moveit_msgs::msg::CollisionObject make_box(
  const std::string & id, const std::string & frame,
  double size_x, double size_y, double size_z, double x, double y, double z)
{
  moveit_msgs::msg::CollisionObject object;
  object.id = id;
  object.header.frame_id = frame;

  shape_msgs::msg::SolidPrimitive primitive;
  primitive.type = primitive.BOX;
  primitive.dimensions.resize(3);
  primitive.dimensions[primitive.BOX_X] = size_x;
  primitive.dimensions[primitive.BOX_Y] = size_y;
  primitive.dimensions[primitive.BOX_Z] = size_z;

  geometry_msgs::msg::Pose pose;
  pose.orientation.w = 1.0;
  pose.position.x = x;
  pose.position.y = y;
  pose.position.z = z;

  object.primitives.push_back(primitive);
  object.primitive_poses.push_back(pose);
  object.operation = object.ADD;
  return object;
}

// Flange pose that puts the fingertips at (x, y, z), gripper pointing down
geometry_msgs::msg::Pose gripper_pose(double x, double y, double z)
{
  geometry_msgs::msg::Pose pose;
  pose.position.x = x;
  pose.position.y = y;
  pose.position.z = z + kTcpOffset;
  pose.orientation.x = 0.92388;   // roll = pi, yaw = -pi/4
  pose.orientation.y = -0.38268;
  pose.orientation.z = 0.0;
  pose.orientation.w = 0.0;
  return pose;
}
}  // namespace

class PickPlace
{
public:
  explicit PickPlace(const rclcpp::Node::SharedPtr & node)
  : logger_(node->get_logger()), arm_(node, "panda_arm"), hand_(node, "hand")
  {
    arm_.setMaxVelocityScalingFactor(0.5);
    arm_.setMaxAccelerationScalingFactor(0.5);
  }

  bool run()
  {
    const double grasp_z = kTableHeight + kBoxSize / 2 + 0.01;

    setup_scene();
    return step("Move to ready", move_named(arm_, "ready")) &&
           step("Open gripper", move_named(hand_, "open")) &&
           step("Pre-grasp", move_to(gripper_pose(kPickX, kPickY, grasp_z + kApproach))) &&
           step("Grasp", move_to(gripper_pose(kPickX, kPickY, grasp_z))) &&
           step("Attach box", attach()) &&
           step("Close gripper", move_named(hand_, "close")) &&
           step("Lift", move_to(gripper_pose(kPickX, kPickY, grasp_z + kApproach))) &&
           step("Pre-place", move_to(gripper_pose(kPlaceX, kPlaceY, grasp_z + kApproach))) &&
           step("Place", move_to(gripper_pose(kPlaceX, kPlaceY, grasp_z))) &&
           step("Open gripper", move_named(hand_, "open")) &&
           step("Detach box", detach()) &&
           step("Retreat", move_to(gripper_pose(kPlaceX, kPlaceY, grasp_z + kApproach))) &&
           step("Back to ready", move_named(arm_, "ready"));
  }

private:
  void setup_scene()
  {
    const std::string frame = arm_.getPlanningFrame();
    scene_.applyCollisionObjects({
        make_box("table", frame, 0.4, 1.0, kTableHeight, 0.5, 0.0, kTableHeight / 2),
        make_box("box", frame, kBoxSize, kBoxSize, kBoxSize,
                 kPickX, kPickY, kTableHeight + kBoxSize / 2),
      });
    RCLCPP_INFO(logger_, "Added table and box to the planning scene");
  }

  bool move_named(MoveGroupInterface & group, const std::string & name)
  {
    group.setNamedTarget(name);
    return static_cast<bool>(group.move());
  }

  bool move_to(const geometry_msgs::msg::Pose & pose)
  {
    arm_.setPoseTarget(pose, "panda_link8");
    return static_cast<bool>(arm_.move());
  }

  bool attach()
  {
    // The fingers and hand may touch the box from now on
    return arm_.attachObject("box", "panda_hand",
                             {"panda_hand", "panda_leftfinger", "panda_rightfinger"});
  }

  bool detach()
  {
    return arm_.detachObject("box");
  }

  bool step(const std::string & name, bool ok)
  {
    if (ok) {
      RCLCPP_INFO(logger_, "[OK]     %s", name.c_str());
    } else {
      RCLCPP_ERROR(logger_, "[FAILED] %s - stopping", name.c_str());
    }
    return ok;
  }

  rclcpp::Logger logger_;
  MoveGroupInterface arm_;
  MoveGroupInterface hand_;
  PlanningSceneInterface scene_;
};

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  auto node = std::make_shared<rclcpp::Node>(
    "pick_place",
    rclcpp::NodeOptions().automatically_declare_parameters_from_overrides(true));

  PickPlace task(node);
  const bool success = task.run();
  RCLCPP_INFO(node->get_logger(), success ? "Pick and place done!" : "Pick and place failed");

  rclcpp::shutdown();
  return success ? 0 : 1;
}
```

### How it works

**The planning scene.** `PlanningSceneInterface::applyCollisionObjects` adds the table and the box to MoveIt's world. From now on, every plan avoids them. Each object is a `moveit_msgs/msg/CollisionObject` with an ID, a frame, a shape, and a pose. Boxes are centered on their pose, so the table's center is at half its height.

**Two groups.** `arm_` plans for the 7 arm joints, `hand_` for the fingers. The gripper uses the named poses `open` and `close` from the Panda's SRDF.

**The step chain.** `run()` chains the steps with `&&`: as soon as one returns `false`, the rest are skipped and the program reports which step failed. A real application would add recovery here, for example retrying with a different grasp.

**Attaching the box.** Before closing the gripper, `attachObject` turns the box from a world object into part of the robot, attached to `panda_hand`. Two things follow:

- The box moves with the gripper in the planning scene, and MoveIt checks it for collisions while the arm carries it.
- The listed **touch links** (hand and fingers) are allowed to touch the box, so closing the gripper around it is not a collision.

`detachObject` turns it back into a world object at its new location.

### Build

Register the executable in `my_moveit_examples/CMakeLists.txt`, next to `hello_moveit`:

```cmake
add_executable(pick_place src/pick_place.cpp)
ament_target_dependencies(pick_place rclcpp moveit_ros_planning_interface)

install(TARGETS hello_moveit pick_place
  DESTINATION lib/${PROJECT_NAME})
```

```bash
$ cd ~/ros2_ws
$ colcon build --packages-select my_moveit_examples
```

---

## Run It

```bash
$ ros2 launch moveit_resources_panda_moveit_config demo.launch.py     # Terminal 1
```

```bash
$ source ~/ros2_ws/install/setup.bash                                 # Terminal 2
$ ros2 run my_moveit_examples pick_place
[INFO] [pick_place]: Added table and box to the planning scene
[INFO] [pick_place]: [OK]     Move to ready
[INFO] [pick_place]: [OK]     Open gripper
[INFO] [pick_place]: [OK]     Pre-grasp
[INFO] [pick_place]: [OK]     Grasp
[INFO] [pick_place]: [OK]     Attach box
[INFO] [pick_place]: [OK]     Close gripper
...
[INFO] [pick_place]: Pick and place done!
```

In RViz2 you see the table and the box appear, the arm reach down, the box turn purple when it is attached, travel with the gripper to the other side, and stay there when released.

Run it a second time: the box now sits at the place position, so the pick step grasps empty air, and MoveIt may report a collision while placing. Restart the demo to reset the scene, or remove the objects first in the **Scene Objects** tab.

> 📝 The demo uses **mock hardware**: the gripper doesn't simulate contact forces, and the box only moves because it is attached in the planning scene. With a real robot or a physics simulation, grasping also depends on friction, gripper force, and an accurate object position, usually from a camera.

---

## When a Step Fails

| Failing step | Likely cause | What to try |
|--------------|--------------|-------------|
| Pre-grasp / pre-place | Pose unreachable with this orientation | Move the box closer to the robot (smaller x), or try it with the interactive marker |
| Grasp / place | Fingers or hand would collide with the box or the table | Raise `grasp_z` slightly; check the yaw so the fingers straddle the box |
| Close gripper | The box isn't attached, so closing is a collision | Make sure `attach()` comes before closing, with the finger links as touch links |
| Lift with box | The attached box would hit the table | Increase `kApproach` |
| Any step: `ABORTED` during execution | Controller problem | Check the demo terminal for errors |

Increase the log level to see details from MoveIt: `ros2 run my_moveit_examples pick_place --ros-args --log-level debug`.

---

## Ideas for Extensions

1. **Several boxes:** add three boxes at different positions and stack them on top of each other at the place position.
2. **Straight-line approach:** use `computeCartesianPath` for the grasp and place motions, so the gripper moves in a guaranteed straight line instead of any collision-free path.
3. **Obstacle:** put a tall box between pick and place. Watch MoveIt plan around it while carrying the object.
4. **Parameters:** read the pick and place positions from ROS parameters so they can be changed without recompiling.
5. **MoveIt Task Constructor:** for complex tasks, MoveIt offers the [Task Constructor](https://moveit.picknik.ai/main/doc/tutorials/pick_and_place_with_moveit_task_constructor/pick_and_place_with_moveit_task_constructor.html), which plans all stages of a pick-and-place together and finds the best combination of grasps.

---

⬅️ [Previous: MoveIt 2](moveit.md) | [⬆️ Manipulation](README.md)
