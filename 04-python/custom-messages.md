# Custom Interfaces

The standard message packages cover a lot, but sooner or later you need data that doesn't fit any of them: a robot-specific status, a request with your own fields, an action for your own task. In this lesson you define your own **message**, **service**, and **action** types in a dedicated interface package, and use the message from a Python node.

## Topics Covered

- `.msg`, `.srv`, and `.action` files
- Field types, arrays, constants, and default values
- Creating an interface package
- `CMakeLists.txt` and `package.xml` for interfaces
- Building and inspecting your interfaces
- Using custom interfaces from Python

---

## Before You Create Your Own

Always check whether a standard type already fits. Standard types work with existing tools (RViz2, rosbag2, plotting tools) and other people's nodes understand them.

| Need | Existing type |
|------|---------------|
| A position or orientation | `geometry_msgs/msg/Point`, `Pose`, `PoseStamped` |
| A velocity | `geometry_msgs/msg/Twist` |
| A single number or string | `std_msgs/msg/Float64`, `Int32`, `String` |
| A yes/no command with a reply | `std_srvs/srv/SetBool`, `Trigger` |

```bash
$ ros2 interface list | grep -i pose     # search the installed interfaces
```

Create your own type when the data is genuinely specific to your robot or application.

---

## Interface Definition Files

Interfaces are defined in simple text files, one type per file:

| Kind | Folder | Sections |
|------|--------|----------|
| Message | `msg/MyMessage.msg` | fields |
| Service | `srv/MyService.srv` | request `---` response |
| Action | `action/MyAction.action` | goal `---` result `---` feedback |

### Fields

Each line is a field: a **type** followed by a **name**.

```text
# Comments start with a hash
string name
float64 x
float64 y
bool near_wall
```

Basic types:

| Type | Python type | Notes |
|------|-------------|-------|
| `bool` | `bool` | |
| `int8`, `int16`, `int32`, `int64` | `int` | Signed integers |
| `uint8`, `uint16`, `uint32`, `uint64` | `int` | Unsigned integers |
| `float32`, `float64` | `float` | Use `float64` unless you have a reason not to |
| `string` | `str` | |
| `byte`, `char` | `bytes`, `int` | |

Other messages can be used as field types by their package and name:

```text
std_msgs/Header header             # timestamp and frame
geometry_msgs/Point position
```

### Arrays

```text
float64[] readings          # any length
float64[3] rgb              # exactly 3 elements
float64[<=10] samples       # at most 10 elements
string<=32 label            # string of at most 32 characters
```

### Constants and default values

```text
uint8 MODE_IDLE=0           # constant: UPPERCASE name, with =
uint8 MODE_DRIVING=1
uint8 mode                  # a normal field that holds one of the constants

float64 max_speed 1.0       # default value: space, no =
```

### Naming rules

- **Type names** (file names) use `CamelCase` and start with a capital letter: `TurtleStatus.msg`.
- **Field names** use `lower_snake_case` and start with a letter: `distance_to_center`.
- **Constants** use `UPPER_SNAKE_CASE`.

---

## Creating an Interface Package

Interface types are generated into Python and C++ code at build time, which requires CMake. Therefore interfaces must live in an **`ament_cmake` package**, even if all your nodes are written in Python. By convention, interface packages end in `_interfaces` or `_msgs`.

```bash
$ cd ~/ros2_ws/src
$ ros2 pkg create my_interfaces --build-type ament_cmake --license Apache-2.0
$ cd my_interfaces
$ rm -rf include src            # not needed for an interface-only package
$ mkdir msg srv action
```

### A message: `msg/TurtleStatus.msg`

```text
# Summary of a turtle's state
string name
float64 x
float64 y
float64 distance_to_center
bool near_wall
```

### A service: `srv/SetTarget.srv`

```text
# Request: where to go
float64 x
float64 y
---
# Response: whether the target was accepted
bool success
string message
```

### An action: `action/Patrol.action`

```text
# Goal: how many laps to drive
uint32 laps
---
# Result
uint32 laps_completed
---
# Feedback
uint32 current_lap
float64 progress
```

---

## `CMakeLists.txt`

Replace the generated `CMakeLists.txt` with:

```cmake
cmake_minimum_required(VERSION 3.8)
project(my_interfaces)

find_package(ament_cmake REQUIRED)
find_package(rosidl_default_generators REQUIRED)

rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/TurtleStatus.msg"
  "srv/SetTarget.srv"
  "action/Patrol.action"
)

ament_export_dependencies(rosidl_default_runtime)
ament_package()
```

- **`rosidl_default_generators`** contains the code generators that turn your files into Python and C++ classes.
- **`rosidl_generate_interfaces`** lists every interface file to generate. A file that is not listed here is simply ignored.
- **`ament_export_dependencies(rosidl_default_runtime)`** makes sure packages using your interfaces also get the runtime support they need.

If an interface uses types from other packages (for example `geometry_msgs/Point`), find them and declare them as dependencies:

```cmake
find_package(geometry_msgs REQUIRED)

rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/TurtleStatus.msg"
  DEPENDENCIES geometry_msgs
)
```

---

## `package.xml`

Add these lines to `my_interfaces/package.xml`, after `<buildtool_depend>ament_cmake</buildtool_depend>`:

```xml
<buildtool_depend>rosidl_default_generators</buildtool_depend>

<depend>action_msgs</depend>

<exec_depend>rosidl_default_runtime</exec_depend>

<member_of_group>rosidl_interface_packages</member_of_group>
```

| Line | Why |
|------|-----|
| `rosidl_default_generators` | Needed to generate the code at build time |
| `action_msgs` | Needed because the package defines an action |
| `rosidl_default_runtime` | Needed to use the generated types at run time |
| `member_of_group` | Registers the package as an interface package. The build fails without it |

If you used other message packages in your definitions (`geometry_msgs`, `std_msgs`, ...), add a `<depend>` for each of them too.

---

## Building and Inspecting

```bash
$ cd ~/ros2_ws
$ colcon build --packages-select my_interfaces
Starting >>> my_interfaces
Finished <<< my_interfaces [6.5s]

Summary: 1 package finished [6.8s]
```

In a new terminal, source the workspace and check the result with the tools from [ros2 topic](../02-ros2-cli/ros2-topic.md#ros2-interface-inspecting-message-types):

```bash
$ source ~/ros2_ws/install/setup.bash
$ ros2 interface package my_interfaces
my_interfaces/msg/TurtleStatus
my_interfaces/srv/SetTarget
my_interfaces/action/Patrol

$ ros2 interface show my_interfaces/msg/TurtleStatus
# Summary of a turtle's state
string name
float64 x
float64 y
float64 distance_to_center
bool near_wall
```

Your types now behave exactly like the standard ones. You can even publish one from the CLI:

```bash
$ ros2 topic pub --once /test my_interfaces/msg/TurtleStatus "{name: 'turtle1', x: 1.0}"
```

---

## Using Custom Interfaces from Python: `status_publisher`

Let's publish a `TurtleStatus` summary of turtlesim's turtle.

### 1. Declare the dependency

In `my_first_pkg/package.xml`:

```xml
<depend>my_interfaces</depend>
```

### 2. Write the node

Create `my_first_pkg/my_first_pkg/status_publisher.py`:

```python
import math

import rclpy
from my_interfaces.msg import TurtleStatus
from rclpy.node import Node
from turtlesim.msg import Pose

CENTER = 5.5
WALL_MARGIN = 1.5


class StatusPublisher(Node):
    def __init__(self):
        super().__init__('status_publisher')
        self.pose = None
        self.create_subscription(Pose, 'turtle1/pose', self.pose_callback, 10)
        self.publisher = self.create_publisher(TurtleStatus, 'turtle_status', 10)
        self.create_timer(0.5, self.publish_status)

    def pose_callback(self, msg):
        self.pose = msg

    def publish_status(self):
        if self.pose is None:
            return

        status = TurtleStatus()
        status.name = 'turtle1'
        status.x = float(self.pose.x)
        status.y = float(self.pose.y)
        status.distance_to_center = math.hypot(self.pose.x - CENTER, self.pose.y - CENTER)
        status.near_wall = (
            min(self.pose.x, self.pose.y) < WALL_MARGIN
            or max(self.pose.x, self.pose.y) > 11.0 - WALL_MARGIN
        )
        self.publisher.publish(status)


def main():
    rclpy.init()
    node = StatusPublisher()
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

The import follows the same pattern as every standard type: `my_interfaces/msg/TurtleStatus` becomes `from my_interfaces.msg import TurtleStatus`. Services and actions work the same way: `from my_interfaces.srv import SetTarget`, `from my_interfaces.action import Patrol`.

> 📝 `turtlesim/msg/Pose` uses `float32` fields, while `TurtleStatus` uses `float64`. Python handles the conversion, but the explicit `float(...)` makes the intent clear. Assigning a value of the wrong type (for example a string to a `float64` field) raises an error immediately.

### 3. Build and run

Register `'status_publisher = my_first_pkg.status_publisher:main'` in `setup.py`. Build both packages; `--packages-up-to` builds `my_interfaces` first because `my_first_pkg` depends on it:

```bash
$ cd ~/ros2_ws
$ colcon build --packages-up-to my_first_pkg --symlink-install
```

```bash
$ ros2 run turtlesim turtlesim_node          # Terminal 1
$ ros2 run my_first_pkg status_publisher     # Terminal 2
$ ros2 run my_first_pkg wall_bouncer         # Terminal 3
```

```bash
$ ros2 topic echo /turtle_status             # Terminal 4
name: turtle1
x: 8.21
y: 5.54
distance_to_center: 2.71
near_wall: false
---
...
```

---

## Common Mistakes

| Symptom | Cause |
|---------|-------|
| Build error mentioning `rosidl_interface_packages` or `member_of_group` | The `<member_of_group>` line is missing from `package.xml` |
| Build error about invalid field or type names | A field name starts with a capital letter, or a type name starts with a lowercase letter |
| A new interface file doesn't appear | It is not listed in `rosidl_generate_interfaces`, or you didn't rebuild |
| `ModuleNotFoundError: No module named 'my_interfaces'` | Not built, workspace not re-sourced, or the dependency is missing from `my_first_pkg/package.xml` |
| Tried to define interfaces in an `ament_python` package | Not supported: interfaces need an `ament_cmake` package |
| Build error about an unknown type such as `geometry_msgs/Point` | That package is missing from `DEPENDENCIES` in `CMakeLists.txt` or from `package.xml` |

---

## Try It Yourself

1. Add a constant pair `uint8 STATE_DRIVING=0` / `uint8 STATE_TURNING=1` and a field `uint8 state` to `TurtleStatus`, rebuild, and set it in `status_publisher` depending on whether the turtle is near a wall.
2. Write a `status_listener` node that subscribes to `/turtle_status` and logs a warning whenever `near_wall` changes from `false` to `true`.
3. Implement a server for `my_interfaces/srv/SetTarget` that teleports the turtle to the requested position using turtlesim's `/turtle1/teleport_absolute` service. Reject targets outside the window with `success: false` and a helpful `message`.
4. Add a `std_msgs/Header header` field to `TurtleStatus`, fill in `header.stamp` with `self.get_clock().now().to_msg()`, and check that `ros2 topic delay /turtle_status` now works.
5. **Challenge:** implement an action server for `my_interfaces/action/Patrol` that drives the turtle around the window `laps` times, using `wall_bouncer`-style logic, and reports the current lap as feedback.

---

⬅️ [Previous: Writing an Action Server and Client](actions.md) | [⬆️ Python (rclpy)](README.md)
