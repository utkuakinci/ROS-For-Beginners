# Writing an Action Server and Client

This lesson implements the action examples from [Module 04](../04-python/actions.md) in C++ with the **rclcpp_action** library: a `fibonacci_server` that computes a sequence step by step with feedback and cancellation, and a `rotate_client` that sends rotation goals to turtlesim.

## Topics Covered

- The `rclcpp_action` library
- Writing an action server: goal, cancel, and accepted handlers
- Running long tasks in a separate thread
- Publishing feedback, handling cancellation, returning a result
- Writing an action client with `SendGoalOptions`
- Waiting for a goal to finish

---

## Setup

Add the action library to `package.xml`:

```xml
<depend>rclcpp_action</depend>
<depend>example_interfaces</depend>
<depend>turtlesim</depend>
```

and to `CMakeLists.txt`:

```cmake
find_package(rclcpp_action REQUIRED)
```

`example_interfaces/action/Fibonacci` has a goal with `int32 order`, and a result and feedback that each contain `int32[] sequence` (check with `ros2 interface show example_interfaces/action/Fibonacci`). In C++, an action type `X` provides `X::Goal`, `X::Result`, and `X::Feedback`.

---

## Writing an Action Server

Create `~/ros2_ws/src/my_cpp_pkg/src/fibonacci_server.cpp`:

```cpp
#include <functional>
#include <memory>
#include <thread>

#include "example_interfaces/action/fibonacci.hpp"
#include "rclcpp/rclcpp.hpp"
#include "rclcpp_action/rclcpp_action.hpp"

class FibonacciServer : public rclcpp::Node
{
public:
  using Fibonacci = example_interfaces::action::Fibonacci;
  using GoalHandle = rclcpp_action::ServerGoalHandle<Fibonacci>;

  FibonacciServer()
  : Node("fibonacci_server")
  {
    using namespace std::placeholders;
    action_server_ = rclcpp_action::create_server<Fibonacci>(
      this,
      "fibonacci",
      std::bind(&FibonacciServer::handle_goal, this, _1, _2),
      std::bind(&FibonacciServer::handle_cancel, this, _1),
      std::bind(&FibonacciServer::handle_accepted, this, _1));
    RCLCPP_INFO(this->get_logger(), "Action server /fibonacci is ready");
  }

private:
  rclcpp_action::GoalResponse handle_goal(
    const rclcpp_action::GoalUUID & uuid,
    std::shared_ptr<const Fibonacci::Goal> goal)
  {
    (void)uuid;
    if (goal->order < 1 || goal->order > 45) {
      RCLCPP_WARN(this->get_logger(), "Rejecting goal with order %d", goal->order);
      return rclcpp_action::GoalResponse::REJECT;
    }
    RCLCPP_INFO(this->get_logger(), "Accepting goal with order %d", goal->order);
    return rclcpp_action::GoalResponse::ACCEPT_AND_EXECUTE;
  }

  rclcpp_action::CancelResponse handle_cancel(const std::shared_ptr<GoalHandle> goal_handle)
  {
    (void)goal_handle;
    RCLCPP_INFO(this->get_logger(), "Received cancel request");
    return rclcpp_action::CancelResponse::ACCEPT;
  }

  void handle_accepted(const std::shared_ptr<GoalHandle> goal_handle)
  {
    // Run the long task in its own thread so the executor stays free.
    std::thread{std::bind(&FibonacciServer::execute, this, std::placeholders::_1), goal_handle}
    .detach();
  }

  void execute(const std::shared_ptr<GoalHandle> goal_handle)
  {
    rclcpp::Rate loop_rate(1);  // 1 Hz: simulate a slow task
    const auto goal = goal_handle->get_goal();
    auto feedback = std::make_shared<Fibonacci::Feedback>();
    auto & sequence = feedback->sequence;
    sequence.push_back(0);
    sequence.push_back(1);
    auto result = std::make_shared<Fibonacci::Result>();

    for (int i = 1; i < goal->order && rclcpp::ok(); ++i) {
      if (goal_handle->is_canceling()) {
        result->sequence = sequence;
        goal_handle->canceled(result);
        RCLCPP_INFO(this->get_logger(), "Goal canceled");
        return;
      }

      sequence.push_back(sequence[i] + sequence[i - 1]);
      goal_handle->publish_feedback(feedback);
      RCLCPP_INFO(this->get_logger(), "Feedback: %zu numbers computed", sequence.size());
      loop_rate.sleep();
    }

    if (rclcpp::ok()) {
      result->sequence = sequence;
      goal_handle->succeed(result);
      RCLCPP_INFO(this->get_logger(), "Goal succeeded");
    }
  }

  rclcpp_action::Server<Fibonacci>::SharedPtr action_server_;
};

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<FibonacciServer>());
  rclcpp::shutdown();
  return 0;
}
```

### The three handlers

`rclcpp_action::create_server` takes three handlers instead of Python's goal, cancel, and execute callbacks:

| Handler | Question | Returns |
|---------|----------|---------|
| `handle_goal(uuid, goal)` | Do I accept this goal? | `GoalResponse::ACCEPT_AND_EXECUTE`, `ACCEPT_AND_DEFER`, or `REJECT` |
| `handle_cancel(goal_handle)` | Do I allow cancelling it? | `CancelResponse::ACCEPT` or `REJECT` |
| `handle_accepted(goal_handle)` | The goal was accepted: start working | nothing |

`(void)uuid;` marks a parameter as intentionally unused, which silences the compiler warning enabled by `-Wall -Wextra`.

### Why a separate thread?

The handlers are called by the executor and must **return quickly**: while one of them runs, the node cannot process anything else, including cancel requests. So `handle_accepted` starts the real work in a new thread and returns immediately:

```cpp
std::thread{std::bind(&FibonacciServer::execute, this, std::placeholders::_1), goal_handle}
.detach();
```

`detach()` lets the thread run independently. Meanwhile the executor keeps spinning, which is why cancel requests are handled while the goal is running, without needing the multi-threaded executor that the [Python version](../04-python/actions.md#executors-and-callback-groups) uses.

### Inside `execute`

| Call | Effect |
|------|--------|
| `goal_handle->get_goal()` | The goal sent by the client |
| `goal_handle->publish_feedback(feedback)` | Send a feedback message |
| `goal_handle->is_canceling()` | `true` once a cancel request was accepted |
| `goal_handle->succeed(result)` / `canceled(result)` / `abort(result)` | Finish the goal with a final state and result |
| `rclcpp::Rate loop_rate(1); loop_rate.sleep();` | Run a loop at a fixed rate (here 1 Hz) |

Sleeping is fine here because `execute` runs in its own thread, not in the executor. The `rclcpp::ok()` checks stop the loop cleanly when the node shuts down with `Ctrl` + `C`.

### Build and test

```cmake
add_executable(fibonacci_server src/fibonacci_server.cpp)
ament_target_dependencies(fibonacci_server rclcpp rclcpp_action example_interfaces)
```

Add it to `install(TARGETS ...)`, rebuild, and test it with the CLI:

```bash
$ ros2 run my_cpp_pkg fibonacci_server
```

```bash
$ ros2 action send_goal --feedback /fibonacci example_interfaces/action/Fibonacci "{order: 5}"
```

Press `Ctrl` + `C` during a goal to cancel it, and send `{order: 100}` to see it rejected.

---

## Writing an Action Client: `rotate_client`

Create `src/rotate_client.cpp`:

```cpp
#include <chrono>
#include <cmath>
#include <cstdlib>
#include <functional>
#include <memory>

#include "rclcpp/rclcpp.hpp"
#include "rclcpp_action/rclcpp_action.hpp"
#include "turtlesim/action/rotate_absolute.hpp"

using namespace std::chrono_literals;

class RotateClient : public rclcpp::Node
{
public:
  using RotateAbsolute = turtlesim::action::RotateAbsolute;
  using GoalHandle = rclcpp_action::ClientGoalHandle<RotateAbsolute>;

  RotateClient()
  : Node("rotate_client")
  {
    client_ = rclcpp_action::create_client<RotateAbsolute>(this, "turtle1/rotate_absolute");
  }

  void send_goal(double degrees)
  {
    RCLCPP_INFO(this->get_logger(), "Waiting for action server...");
    if (!client_->wait_for_action_server(10s)) {
      RCLCPP_ERROR(this->get_logger(), "Action server not available");
      done_ = true;
      return;
    }

    auto goal = RotateAbsolute::Goal();
    goal.theta = degrees * M_PI / 180.0;

    using namespace std::placeholders;
    auto options = rclcpp_action::Client<RotateAbsolute>::SendGoalOptions();
    options.goal_response_callback = std::bind(&RotateClient::goal_response_callback, this, _1);
    options.feedback_callback = std::bind(&RotateClient::feedback_callback, this, _1, _2);
    options.result_callback = std::bind(&RotateClient::result_callback, this, _1);

    RCLCPP_INFO(this->get_logger(), "Sending goal: rotate to %.1f degrees", degrees);
    client_->async_send_goal(goal, options);
  }

  bool done() const {return done_;}

private:
  void goal_response_callback(const GoalHandle::SharedPtr & goal_handle)
  {
    if (!goal_handle) {
      RCLCPP_WARN(this->get_logger(), "Goal rejected");
      done_ = true;
      return;
    }
    RCLCPP_INFO(this->get_logger(), "Goal accepted");
  }

  void feedback_callback(
    GoalHandle::SharedPtr,
    const std::shared_ptr<const RotateAbsolute::Feedback> feedback)
  {
    RCLCPP_INFO(this->get_logger(), "Remaining: %.1f degrees", feedback->remaining * 180.0 / M_PI);
  }

  void result_callback(const GoalHandle::WrappedResult & result)
  {
    switch (result.code) {
      case rclcpp_action::ResultCode::SUCCEEDED:
        RCLCPP_INFO(
          this->get_logger(), "Succeeded, rotated by %.1f degrees",
          result.result->delta * 180.0 / M_PI);
        break;
      case rclcpp_action::ResultCode::ABORTED:
        RCLCPP_WARN(this->get_logger(), "Goal was aborted");
        break;
      case rclcpp_action::ResultCode::CANCELED:
        RCLCPP_WARN(this->get_logger(), "Goal was canceled");
        break;
      default:
        RCLCPP_ERROR(this->get_logger(), "Unknown result code");
        break;
    }
    done_ = true;
  }

  rclcpp_action::Client<RotateAbsolute>::SharedPtr client_;
  bool done_ = false;
};

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  if (argc < 2) {
    RCLCPP_ERROR(rclcpp::get_logger("rotate_client"), "Usage: rotate_client DEGREES");
    rclcpp::shutdown();
    return 1;
  }

  auto node = std::make_shared<RotateClient>();
  node->send_goal(std::atof(argv[1]));

  rclcpp::executors::SingleThreadedExecutor executor;
  executor.add_node(node);
  while (rclcpp::ok() && !node->done()) {
    executor.spin_once(100ms);
  }

  rclcpp::shutdown();
  return 0;
}
```

### How it works

**`SendGoalOptions`** bundles the three client callbacks that follow a goal through its life (see the [diagram in the Python lesson](../04-python/actions.md#the-life-of-a-goal-step-by-step)):

| Callback | Called when | Argument |
|----------|-------------|----------|
| `goal_response_callback` | The server accepted or rejected the goal | The goal handle, or `nullptr` if rejected |
| `feedback_callback` | A feedback message arrives | The goal handle and the feedback |
| `result_callback` | The goal finished | A `WrappedResult` with `code` and `result` |

In C++ the result is requested automatically, so there is no separate `get_result_async()` step as in Python.

**`WrappedResult::code`** tells you how the goal ended: `SUCCEEDED`, `ABORTED`, or `CANCELED`. Only read `result.result` after checking the code.

**Waiting for the goal to finish.** `main()` uses an executor and calls `spin_once(100ms)` in a loop until `done()` returns `true`. Each call processes pending events (or waits up to 100 ms for one), so the loop doesn't waste CPU.

**Cancelling from code.** Keep the goal handle from `goal_response_callback` and call `client_->async_cancel_goal(goal_handle)`.

### Build and run

```cmake
add_executable(rotate_client src/rotate_client.cpp)
ament_target_dependencies(rotate_client rclcpp rclcpp_action turtlesim)
```

```bash
$ ros2 run turtlesim turtlesim_node
```

```bash
$ ros2 run my_cpp_pkg rotate_client 90
[INFO] [rotate_client]: Waiting for action server...
[INFO] [rotate_client]: Sending goal: rotate to 90.0 degrees
[INFO] [rotate_client]: Goal accepted
[INFO] [rotate_client]: Remaining: 88.2 degrees
...
[INFO] [rotate_client]: Succeeded, rotated by -90.0 degrees
```

---

## The Complete `CMakeLists.txt`

After all four lessons, the build part of `my_cpp_pkg/CMakeLists.txt` looks like this:

```cmake
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(rclcpp_action REQUIRED)
find_package(std_msgs REQUIRED)
find_package(geometry_msgs REQUIRED)
find_package(turtlesim REQUIRED)
find_package(example_interfaces REQUIRED)

add_executable(talker src/talker.cpp)
ament_target_dependencies(talker rclcpp std_msgs)

add_executable(circle_driver src/circle_driver.cpp)
ament_target_dependencies(circle_driver rclcpp geometry_msgs)

add_executable(listener src/listener.cpp)
ament_target_dependencies(listener rclcpp std_msgs)

add_executable(wall_bouncer src/wall_bouncer.cpp)
ament_target_dependencies(wall_bouncer rclcpp geometry_msgs turtlesim)

add_executable(add_two_ints_server src/add_two_ints_server.cpp)
ament_target_dependencies(add_two_ints_server rclcpp example_interfaces)

add_executable(add_two_ints_client src/add_two_ints_client.cpp)
ament_target_dependencies(add_two_ints_client rclcpp example_interfaces)

add_executable(turtle_spawner src/turtle_spawner.cpp)
ament_target_dependencies(turtle_spawner rclcpp turtlesim)

add_executable(fibonacci_server src/fibonacci_server.cpp)
ament_target_dependencies(fibonacci_server rclcpp rclcpp_action example_interfaces)

add_executable(rotate_client src/rotate_client.cpp)
ament_target_dependencies(rotate_client rclcpp rclcpp_action turtlesim)

install(TARGETS
  talker
  circle_driver
  listener
  wall_bouncer
  add_two_ints_server
  add_two_ints_client
  turtle_spawner
  fibonacci_server
  rotate_client
  DESTINATION lib/${PROJECT_NAME}
)

ament_package()
```

---

## Common Mistakes

| Symptom | Cause |
|---------|-------|
| Cancel requests are only handled after the goal finishes | The work runs inside `handle_accepted` instead of a separate thread |
| Cancelling has no effect | `execute` never checks `goal_handle->is_canceling()` |
| Template errors around `create_server` or `SendGoalOptions` | A handler or callback has the wrong signature or the wrong number of placeholders |
| `undefined reference to rclcpp_action::...` | `rclcpp_action` missing from `ament_target_dependencies` |
| Crash when reading the result | Reading `result.result` without checking `result.code` |
| The client never exits | Nothing sets `done_`, or `main()` spins forever with `rclcpp::spin` |

---

## Try It Yourself

1. Make the C++ Fibonacci server abort goals with an odd order, and check the status in the CLI.
2. Write a C++ `fibonacci_client` that prints every feedback message and the final sequence.
3. Extend `rotate_client` so it cancels the goal if it hasn't finished within 2 seconds. Hint: create a one-shot timer when the goal is accepted.
4. Write a C++ `look_around` node that makes the turtle face up, left, down, and right in sequence, sending each goal from the previous goal's result callback.
5. Use the C++ Fibonacci server with a Python client, or the other way around, to confirm that actions work across languages too.

---

⬅️ [Previous: Writing a Service Server and Client](services.md) | [⬆️ C++ (rclcpp)](README.md)
