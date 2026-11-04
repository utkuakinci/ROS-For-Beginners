# Writing a Publisher

In this lesson you write the C++ versions of the two publishers from [Module 04](../04-python/publisher.md): a `talker` that sends text and a `circle_driver` that drives the turtle. You will learn the structure of an rclcpp node and, just as important, how to add a node to `CMakeLists.txt` so it gets built and installed.

## Topics Covered

- The anatomy of an rclcpp node
- Creating a publisher and a timer
- Building a node with `CMakeLists.txt`
- Logging in C++
- Parameters in C++
- Comparing rclcpp with rclpy

---

## A Minimal Publisher: `talker`

Create `~/ros2_ws/src/my_cpp_pkg/src/talker.cpp`:

```cpp
#include <chrono>
#include <memory>
#include <string>

#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"

using namespace std::chrono_literals;

class Talker : public rclcpp::Node
{
public:
  Talker()
  : Node("talker"), count_(0)
  {
    publisher_ = this->create_publisher<std_msgs::msg::String>("chatter", 10);
    timer_ = this->create_wall_timer(500ms, std::bind(&Talker::timer_callback, this));
  }

private:
  void timer_callback()
  {
    auto msg = std_msgs::msg::String();
    msg.data = "Hello World: " + std::to_string(count_++);
    RCLCPP_INFO(this->get_logger(), "Publishing: '%s'", msg.data.c_str());
    publisher_->publish(msg);
  }

  rclcpp::Publisher<std_msgs::msg::String>::SharedPtr publisher_;
  rclcpp::TimerBase::SharedPtr timer_;
  size_t count_;
};

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<Talker>());
  rclcpp::shutdown();
  return 0;
}
```

### How it works

```cpp
#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"
```

Message types are generated C++ classes. The header name is the type name in `lower_snake_case`: `std_msgs/msg/String` → `std_msgs/msg/string.hpp`, and the class is `std_msgs::msg::String`. (`geometry_msgs/msg/TwistStamped` → `geometry_msgs/msg/twist_stamped.hpp`.)

```cpp
class Talker : public rclcpp::Node
{
public:
  Talker()
  : Node("talker"), count_(0)
```

The node is a class derived from `rclcpp::Node`. The constructor passes the node name to the base class in the initializer list, just like `super().__init__('talker')` in Python.

```cpp
publisher_ = this->create_publisher<std_msgs::msg::String>("chatter", 10);
```

The message type is a **template argument** (`<...>`) instead of a normal argument. `10` is the queue depth, as in Python. The result is a `SharedPtr` that you must store as a member: if it goes out of scope, the publisher is destroyed.

```cpp
timer_ = this->create_wall_timer(500ms, std::bind(&Talker::timer_callback, this));
```

`create_wall_timer` takes a period (`500ms` comes from `std::chrono_literals`) and a callback. `std::bind(&Talker::timer_callback, this)` turns the member function into a callable object bound to this instance. A lambda works just as well:

```cpp
timer_ = this->create_wall_timer(500ms, [this]() { timer_callback(); });
```

```cpp
RCLCPP_INFO(this->get_logger(), "Publishing: '%s'", msg.data.c_str());
```

Logging uses macros with `printf`-style formatting. `%s` needs a C string, hence `.c_str()`. See [Logging in C++](#logging-in-c) below.

```cpp
rclcpp::Publisher<std_msgs::msg::String>::SharedPtr publisher_;
rclcpp::TimerBase::SharedPtr timer_;
```

Member variables end with an underscore, following the ROS 2 C++ style guide.

```cpp
int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<Talker>());
  rclcpp::shutdown();
  return 0;
}
```

The same steps as in Python: initialize, create the node, spin until `Ctrl` + `C`, shut down. Passing `argc` and `argv` to `init` lets `--ros-args` work.

---

## Building the Node: `CMakeLists.txt`

C++ code must be compiled, and `CMakeLists.txt` tells CMake how. For each executable, you need three things: find its dependencies, build it, and install it.

Open `~/ros2_ws/src/my_cpp_pkg/CMakeLists.txt` and edit it so it looks like this (keep any generated lines you don't see here, such as the compiler warning options and the `BUILD_TESTING` block):

```cmake
cmake_minimum_required(VERSION 3.8)
project(my_cpp_pkg)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# 1. Find dependencies
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)

# 2. Build the executable and link its dependencies
add_executable(talker src/talker.cpp)
ament_target_dependencies(talker rclcpp std_msgs)

# 3. Install it where `ros2 run` looks for executables
install(TARGETS
  talker
  DESTINATION lib/${PROJECT_NAME}
)

ament_package()
```

| Command | Purpose |
|---------|---------|
| `find_package(std_msgs REQUIRED)` | Locate the package; the build fails early with a clear message if it is missing |
| `add_executable(talker src/talker.cpp)` | Compile `talker.cpp` into an executable named `talker` |
| `ament_target_dependencies(talker rclcpp std_msgs)` | Add the include paths and libraries of these packages to `talker` |
| `install(TARGETS ... DESTINATION lib/${PROJECT_NAME})` | Copy the executable to `install/my_cpp_pkg/lib/my_cpp_pkg/` |
| `ament_package()` | Generate the files other tools need. Must be the **last** line |

> 📝 Newer ROS 2 distributions recommend `target_link_libraries(talker PUBLIC rclcpp::rclcpp ${std_msgs_TARGETS})` instead of `ament_target_dependencies`. Both work on Humble and Jazzy; this guide uses `ament_target_dependencies` because it is shorter and appears in most existing code.

Every package you `find_package` must also be declared in `package.xml`:

```xml
<depend>rclcpp</depend>
<depend>std_msgs</depend>
```

### Build and run

```bash
$ cd ~/ros2_ws
$ colcon build --packages-select my_cpp_pkg
Starting >>> my_cpp_pkg
Finished <<< my_cpp_pkg [12.4s]

Summary: 1 package finished [12.7s]
```

```bash
$ source ~/ros2_ws/install/setup.bash
$ ros2 run my_cpp_pkg talker
[INFO] [1730800000.123456789] [talker]: Publishing: 'Hello World: 0'
[INFO] [1730800000.623456789] [talker]: Publishing: 'Hello World: 1'
...
```

The C++ talker and the Python listener from [Module 04](../04-python/subscriber.md) work together: `ros2 run my_first_pkg listener`.

> ⚠️ Unlike Python with `--symlink-install`, C++ code must be **rebuilt after every change**. If your change has no effect, you probably forgot to run `colcon build`.

When compilation fails, scroll up to the **first** error. C++ compilers often print many lines of follow-up errors caused by a single mistake.

---

## Logging in C++

| Macro | Example |
|-------|---------|
| `RCLCPP_DEBUG`, `RCLCPP_INFO`, `RCLCPP_WARN`, `RCLCPP_ERROR`, `RCLCPP_FATAL` | `RCLCPP_INFO(get_logger(), "Speed: %.2f", speed);` |
| `..._STREAM` variants | `RCLCPP_INFO_STREAM(get_logger(), "Speed: " << speed);` |
| `..._THROTTLE` variants | `RCLCPP_WARN_THROTTLE(get_logger(), *get_clock(), 1000, "Low battery");` (at most once per 1000 ms) |
| `..._ONCE` variants | `RCLCPP_INFO_ONCE(get_logger(), "First message received");` |

Common format specifiers: `%d` (int), `%zu` (`size_t`), `%f` / `%.2f` (double), `%s` (C string). The `_STREAM` variants avoid format specifiers altogether and accept anything that works with `<<`.

---

## Driving the Turtle: `circle_driver`

Create `src/circle_driver.cpp`:

```cpp
#include <chrono>
#include <memory>

#include "geometry_msgs/msg/twist.hpp"
#include "rclcpp/rclcpp.hpp"

using namespace std::chrono_literals;

class CircleDriver : public rclcpp::Node
{
public:
  CircleDriver()
  : Node("circle_driver")
  {
    this->declare_parameter("linear_speed", 2.0);
    this->declare_parameter("angular_speed", 1.0);

    publisher_ = this->create_publisher<geometry_msgs::msg::Twist>("turtle1/cmd_vel", 10);
    timer_ = this->create_wall_timer(100ms, std::bind(&CircleDriver::timer_callback, this));
  }

private:
  void timer_callback()
  {
    auto msg = geometry_msgs::msg::Twist();
    msg.linear.x = this->get_parameter("linear_speed").as_double();
    msg.angular.z = this->get_parameter("angular_speed").as_double();
    publisher_->publish(msg);
  }

  rclcpp::Publisher<geometry_msgs::msg::Twist>::SharedPtr publisher_;
  rclcpp::TimerBase::SharedPtr timer_;
};

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<CircleDriver>());
  rclcpp::shutdown();
  return 0;
}
```

What is new compared to Python:

- `declare_parameter("linear_speed", 2.0)` takes its type from the default value, as in Python. `2.0` is a `double`; writing `2` would make it an integer parameter.
- `get_parameter(...).as_double()` returns the value with an explicit type. Other accessors include `as_int()`, `as_bool()`, `as_string()`, and `as_double_array()`.

Add it to `CMakeLists.txt`:

```cmake
find_package(geometry_msgs REQUIRED)

add_executable(circle_driver src/circle_driver.cpp)
ament_target_dependencies(circle_driver rclcpp geometry_msgs)

install(TARGETS
  talker
  circle_driver
  DESTINATION lib/${PROJECT_NAME}
)
```

Add `<depend>geometry_msgs</depend>` to `package.xml`, rebuild, and run it next to turtlesim:

```bash
$ ros2 run turtlesim turtlesim_node
```

```bash
$ ros2 run my_cpp_pkg circle_driver
$ ros2 param set /circle_driver angular_speed 0.5      # in another terminal
```

It behaves exactly like the Python version, and the same CLI tools work on it.

---

## rclpy vs rclcpp: Side by Side

| Task | Python | C++ |
|------|--------|-----|
| Node class | `class Talker(Node)` | `class Talker : public rclcpp::Node` |
| Node name | `super().__init__('talker')` | `: Node("talker")` |
| Publisher | `self.create_publisher(String, 'chatter', 10)` | `create_publisher<std_msgs::msg::String>("chatter", 10)` |
| Timer | `self.create_timer(0.5, self.cb)` | `create_wall_timer(500ms, std::bind(&Talker::cb, this))` |
| Message | `msg = String()` | `auto msg = std_msgs::msg::String();` |
| Publish | `self.publisher.publish(msg)` | `publisher_->publish(msg);` |
| Log | `self.get_logger().info(f'x={x}')` | `RCLCPP_INFO(get_logger(), "x=%f", x);` |
| Parameter | `self.get_parameter('p').value` | `get_parameter("p").as_double()` |
| Register executable | `entry_points` in `setup.py` | `add_executable` + `install` in `CMakeLists.txt` |

---

## Common Mistakes

| Symptom | Cause |
|---------|-------|
| `fatal error: std_msgs/msg/string.hpp: No such file or directory` | Missing `find_package` / `ament_target_dependencies` for that package, or wrong header name |
| `No executable found` | The executable is not in `install(TARGETS ...)`, or you didn't rebuild |
| Code changes have no effect | C++ needs `colcon build` after every change |
| Node starts but never publishes | The timer or publisher `SharedPtr` was stored in a local variable and destroyed at the end of the constructor |
| `ros2 param set` fails with a "wrong parameter type" error | Setting an integer (`1`) on a `double` parameter. Use `1.0` |
| Long, confusing template errors | Usually a wrong message type or callback signature. Read the first error line carefully |

---

## Try It Yourself

1. Add a `message` parameter to the C++ talker, like in the Python exercise.
2. Rewrite the talker's timer callback with a lambda instead of `std::bind`.
3. Use `RCLCPP_INFO_STREAM` to log the turtle's linear and angular speed in `circle_driver`, throttled to once per second.
4. Write a C++ `square_driver` (see the [Python exercise](../04-python/publisher.md#try-it-yourself)).
5. Introduce a deliberate typo in a type name, build, and practice finding the first relevant error in the compiler output.

---

[⬆️ C++ (rclcpp)](README.md) | [Next: Writing a Subscriber](subscriber.md) ➡️
