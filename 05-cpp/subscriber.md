# Writing a Subscriber

This lesson rewrites the subscribers from [Module 04](../04-python/subscriber.md) in C++: a `listener` for the talker, and a `wall_bouncer` that reads the turtle's pose and steers it away from the walls. You will learn the callback signatures rclcpp accepts and how to store the latest message safely.

## Topics Covered

- Creating a subscription
- Callback signatures: references, shared pointers, lambdas
- Combining a subscriber, a publisher, and a timer
- Storing the latest message
- Throttled logging
- QoS for subscribers

---

## A Minimal Subscriber: `listener`

Create `~/ros2_ws/src/my_cpp_pkg/src/listener.cpp`:

```cpp
#include <functional>
#include <memory>

#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"

using std::placeholders::_1;

class Listener : public rclcpp::Node
{
public:
  Listener()
  : Node("listener")
  {
    subscription_ = this->create_subscription<std_msgs::msg::String>(
      "chatter", 10, std::bind(&Listener::topic_callback, this, _1));
  }

private:
  void topic_callback(const std_msgs::msg::String & msg) const
  {
    RCLCPP_INFO(this->get_logger(), "I heard: '%s'", msg.data.c_str());
  }

  rclcpp::Subscription<std_msgs::msg::String>::SharedPtr subscription_;
};

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<Listener>());
  rclcpp::shutdown();
  return 0;
}
```

### How it works

```cpp
subscription_ = this->create_subscription<std_msgs::msg::String>(
  "chatter", 10, std::bind(&Listener::topic_callback, this, _1));
```

Note the argument order: **topic, QoS, callback**. (In Python it is topic, callback, QoS.) The callback takes one argument, the message, so `std::bind` needs the placeholder `_1` to pass it through.

```cpp
void topic_callback(const std_msgs::msg::String & msg) const
```

The message arrives as a **const reference**: you can read it without copying it, but not modify it. The method is `const` because it doesn't change the node's state.

As with publishers, the subscription `SharedPtr` must be stored as a member, or the subscription disappears when the constructor ends.

### Callback signatures

rclcpp accepts several callback forms. The two you will use most:

```cpp
// Const reference: read the message, no ownership
void callback(const std_msgs::msg::String & msg);

// Shared pointer: keep the message after the callback returns
void callback(const std_msgs::msg::String::SharedPtr msg);
```

Use the **reference** when you only process the message inside the callback, and the **shared pointer** when you want to store it, as `wall_bouncer` does below.

A lambda avoids `std::bind` and placeholders entirely:

```cpp
subscription_ = this->create_subscription<std_msgs::msg::String>(
  "chatter", 10,
  [this](const std_msgs::msg::String & msg) {
    RCLCPP_INFO(this->get_logger(), "I heard: '%s'", msg.data.c_str());
  });
```

### Build and run

Add to `CMakeLists.txt`:

```cmake
add_executable(listener src/listener.cpp)
ament_target_dependencies(listener rclcpp std_msgs)
```

and add `listener` to the `install(TARGETS ...)` list. Rebuild, then run it with either talker:

```bash
$ ros2 run my_cpp_pkg talker          # or: ros2 run my_first_pkg talker
```

```bash
$ ros2 run my_cpp_pkg listener
[INFO] [1730800101.124567890] [listener]: I heard: 'Hello World: 3'
...
```

---

## Sense, Decide, Act: `wall_bouncer`

Create `src/wall_bouncer.cpp`:

```cpp
#include <chrono>
#include <memory>

#include "geometry_msgs/msg/twist.hpp"
#include "rclcpp/rclcpp.hpp"
#include "turtlesim/msg/pose.hpp"

using namespace std::chrono_literals;
using std::placeholders::_1;

class WallBouncer : public rclcpp::Node
{
public:
  WallBouncer()
  : Node("wall_bouncer")
  {
    this->declare_parameter("margin", 1.5);

    pose_subscription_ = this->create_subscription<turtlesim::msg::Pose>(
      "turtle1/pose", 10, std::bind(&WallBouncer::pose_callback, this, _1));
    cmd_publisher_ = this->create_publisher<geometry_msgs::msg::Twist>("turtle1/cmd_vel", 10);
    timer_ = this->create_wall_timer(100ms, std::bind(&WallBouncer::control_loop, this));
  }

private:
  void pose_callback(const turtlesim::msg::Pose::SharedPtr msg)
  {
    // Only store the latest pose; the decision is made in the control loop.
    pose_ = msg;
  }

  void control_loop()
  {
    if (!pose_) {
      RCLCPP_INFO_THROTTLE(
        this->get_logger(), *this->get_clock(), 2000, "Waiting for the first pose...");
      return;
    }

    const double margin = this->get_parameter("margin").as_double();
    const bool near_wall =
      pose_->x < margin || pose_->x > 11.0 - margin ||
      pose_->y < margin || pose_->y > 11.0 - margin;

    auto cmd = geometry_msgs::msg::Twist();
    if (near_wall) {
      cmd.linear.x = 0.5;
      cmd.angular.z = 2.0;
      RCLCPP_INFO_THROTTLE(
        this->get_logger(), *this->get_clock(), 1000,
        "Wall ahead at (%.1f, %.1f), turning", pose_->x, pose_->y);
    } else {
      cmd.linear.x = 2.0;
    }
    cmd_publisher_->publish(cmd);
  }

  rclcpp::Subscription<turtlesim::msg::Pose>::SharedPtr pose_subscription_;
  rclcpp::Publisher<geometry_msgs::msg::Twist>::SharedPtr cmd_publisher_;
  rclcpp::TimerBase::SharedPtr timer_;
  turtlesim::msg::Pose::SharedPtr pose_;
};

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<WallBouncer>());
  rclcpp::shutdown();
  return 0;
}
```

### What is new compared to Python

**Storing the message.** The callback receives a `SharedPtr` and keeps it in `pose_`. Storing the pointer is cheap (no copy) and keeps the message alive after the callback returns.

**Checking for missing data.** A `SharedPtr` is empty (`nullptr`) until the first message arrives, so `if (!pose_)` plays the role of Python's `if self.pose is None`.

**Throttled logging.** `RCLCPP_INFO_THROTTLE` needs the node's clock (`*this->get_clock()`) and a period in **milliseconds**.

**Thread safety.** With the default single-threaded executor, `pose_callback` and `control_loop` never run at the same time, so sharing `pose_` between them is safe without a mutex. The same rules as in [Python](../04-python/subscriber.md#how-callbacks-and-spinning-work) apply: keep callbacks short and never sleep in them.

### Build and run

```cmake
find_package(turtlesim REQUIRED)

add_executable(wall_bouncer src/wall_bouncer.cpp)
ament_target_dependencies(wall_bouncer rclcpp geometry_msgs turtlesim)
```

Add `wall_bouncer` to `install(TARGETS ...)` and `<depend>turtlesim</depend>` to `package.xml`. Rebuild and run:

```bash
$ ros2 run turtlesim turtlesim_node
```

```bash
$ ros2 run my_cpp_pkg wall_bouncer
$ ros2 param set /wall_bouncer margin 3.0             # in another terminal
```

---

## QoS for Subscribers

The rules are the same as in [Python](../04-python/subscriber.md#qos-for-subscribers): a subscriber's QoS must be compatible with the publisher's, and sensor topics are usually best effort. rclcpp provides a ready-made sensor data profile:

```cpp
#include "sensor_msgs/msg/laser_scan.hpp"

scan_subscription_ = this->create_subscription<sensor_msgs::msg::LaserScan>(
  "scan", rclcpp::SensorDataQoS(),
  std::bind(&MyNode::scan_callback, this, _1));
```

For full control, build a profile explicitly:

```cpp
auto qos = rclcpp::QoS(rclcpp::KeepLast(1)).reliable().transient_local();
```

---

## Common Mistakes

| Symptom | Cause |
|---------|-------|
| The callback is never called | Topic name, type, or QoS mismatch, or the subscription `SharedPtr` was not stored as a member |
| Long template error mentioning `create_subscription` | The callback signature doesn't match the message type, or `_1` is missing from `std::bind` |
| Crash (segmentation fault) in the control loop | Using `pose_` before the first message arrived. Check `if (!pose_)` |
| Arguments in the wrong order | C++ is `(topic, qos, callback)`, Python is `(topic, callback, qos)` |
| `RCLCPP_INFO_THROTTLE` doesn't compile | It needs the clock as its second argument: `*this->get_clock()` |

---

## Try It Yourself

1. Rewrite the listener's subscription with a lambda.
2. Write a C++ `pose_reporter` that logs the turtle's position and its heading in degrees, at most once per second.
3. Run the C++ `wall_bouncer` for `turtle1` and the Python `wall_bouncer` for a spawned `turtle2` (using remapping, as in the [Python exercise](../04-python/subscriber.md#try-it-yourself)). Can you tell them apart from the outside?
4. Change `pose_callback` to take a `const turtlesim::msg::Pose & msg` and store a copy instead (`turtlesim::msg::Pose pose_;` plus a `bool has_pose_` flag). Which version do you find clearer?

---

⬅️ [Previous: Writing a Publisher](publisher.md) | [⬆️ C++ (rclcpp)](README.md) | [Next: Writing a Service Server and Client](services.md) ➡️
