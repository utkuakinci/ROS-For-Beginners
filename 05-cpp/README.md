# 05. ROS 2 Client Libraries: C++ (rclcpp)

Modern C++ implementations of the core ROS 2 communication mechanisms.

This module rewrites the nodes from [Module 04](../04-python/README.md) in C++ with **rclcpp**, the ROS 2 C++ client library. The concepts are the same; what changes is the syntax, the build configuration (`CMakeLists.txt`), and a few C++-specific patterns. C++ is the language of choice for performance-critical parts of a robot: drivers, controllers, and perception. Most of the ROS 2 core and large libraries like Nav2 and MoveIt 2 are written in it.

## Lessons

1. [Writing a Publisher](publisher.md) — Create a C++ node that publishes messages on a topic.
2. [Writing a Subscriber](subscriber.md) — Create a C++ node that receives messages from a topic.
3. [Writing a Service Server and Client](services.md) — Implement request/response communication in C++.
4. [Writing an Action Server and Client](actions.md) — Implement a long-running, cancellable task in C++.

## What You Will Build

All nodes go into the `my_cpp_pkg` package created in [Packages](../03-workspaces-and-packages/packages.md#a-first-look-at-ament_cmake-packages):

| Lesson | Executable | Python equivalent |
|--------|------------|-------------------|
| Publisher | `talker`, `circle_driver` | [Publisher](../04-python/publisher.md) |
| Subscriber | `listener`, `wall_bouncer` | [Subscriber](../04-python/subscriber.md) |
| Services | `add_two_ints_server`, `add_two_ints_client`, `turtle_spawner` | [Services](../04-python/services.md) |
| Actions | `fibonacci_server`, `rotate_client` | [Actions](../04-python/actions.md) |

If you haven't created the package yet:

```bash
$ cd ~/ros2_ws/src
$ ros2 pkg create my_cpp_pkg --build-type ament_cmake --dependencies rclcpp --license Apache-2.0
```

## The C++ You Need

This module assumes basic C++ (variables, functions, classes, `std::string`, `std::vector`). These features appear in every rclcpp node; make sure you recognize them:

| Feature | Example | Used for |
|---------|---------|----------|
| Inheritance | `class Talker : public rclcpp::Node` | Every node derives from `rclcpp::Node` |
| Smart pointers | `std::shared_ptr<T>`, `std::make_shared<T>()` | rclcpp objects are managed by shared pointers (`SharedPtr`) |
| `auto` | `auto msg = std_msgs::msg::String();` | Avoid repeating long type names |
| Lambdas | `[this](const Msg & msg) { ... }` | Short callbacks |
| `std::bind` | `std::bind(&Talker::callback, this)` | Using a member function as a callback |
| Chrono literals | `500ms`, `1s` | Timer periods and timeouts |
| Namespaces and `using` | `using namespace std::chrono_literals;` | Shorter code |

Good free references: [learncpp.com](https://www.learncpp.com/) for the language, and [cppreference.com](https://en.cppreference.com/) for the standard library.

## Python vs C++ at a Glance

| | Python (rclpy) | C++ (rclcpp) |
|---|---|---|
| Build after a code change | Not needed with `--symlink-install` | **Always** (`colcon build`) |
| Build configuration | `setup.py` | `CMakeLists.txt` |
| Errors found | At run time | Mostly at compile time |
| Speed | Good enough for most nodes | Faster, more predictable timing |
| Typical use | Prototypes, high-level logic, tools | Drivers, controllers, perception, libraries |

Python and C++ nodes communicate seamlessly, so you can mix both in one system.

---

⬅️ [Previous: ROS 2 Client Libraries: Python (rclpy)](../04-python/README.md) | [🏠 Home](../README.md) | [Next: Transform Library (TF2)](../06-tf2/README.md) ➡️
