# Writing a Service Server and Client

This lesson implements the service examples from [Module 04](../04-python/services.md) in C++: an `add_two_ints` server and client, and a `turtle_spawner` node that calls turtlesim's `/spawn` service from a running node. The rule from the Python lesson applies unchanged: **never wait for a response inside a callback**.

## Topics Covered

- Writing a service server
- Writing a one-shot service client
- Futures and `spin_until_future_complete`
- Calling services asynchronously from a running node
- Avoiding deadlocks

---

## Setup

Both examples use types from other packages. Add them to `package.xml`:

```xml
<depend>example_interfaces</depend>
<depend>turtlesim</depend>
```

and to `CMakeLists.txt`:

```cmake
find_package(example_interfaces REQUIRED)
find_package(turtlesim REQUIRED)
```

The service type `example_interfaces/srv/AddTwoInts` has a request with `int64 a` and `int64 b`, and a response with `int64 sum`. In C++, each service type `X` provides `X::Request` and `X::Response`.

---

## Writing a Service Server

Create `~/ros2_ws/src/my_cpp_pkg/src/add_two_ints_server.cpp`:

```cpp
#include <functional>
#include <memory>

#include "example_interfaces/srv/add_two_ints.hpp"
#include "rclcpp/rclcpp.hpp"

using example_interfaces::srv::AddTwoInts;
using std::placeholders::_1;
using std::placeholders::_2;

class AddTwoIntsServer : public rclcpp::Node
{
public:
  AddTwoIntsServer()
  : Node("add_two_ints_server")
  {
    service_ = this->create_service<AddTwoInts>(
      "add_two_ints", std::bind(&AddTwoIntsServer::add_callback, this, _1, _2));
    RCLCPP_INFO(this->get_logger(), "Service /add_two_ints is ready");
  }

private:
  void add_callback(
    const std::shared_ptr<AddTwoInts::Request> request,
    std::shared_ptr<AddTwoInts::Response> response)
  {
    response->sum = request->a + request->b;
    RCLCPP_INFO_STREAM(
      this->get_logger(), request->a << " + " << request->b << " = " << response->sum);
  }

  rclcpp::Service<AddTwoInts>::SharedPtr service_;
};

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<AddTwoIntsServer>());
  rclcpp::shutdown();
  return 0;
}
```

### How it works

```cpp
service_ = this->create_service<AddTwoInts>(
  "add_two_ints", std::bind(&AddTwoIntsServer::add_callback, this, _1, _2));
```

The callback takes **two** arguments, the request and the response, so `std::bind` needs both placeholders `_1` and `_2`.

```cpp
void add_callback(
  const std::shared_ptr<AddTwoInts::Request> request,
  std::shared_ptr<AddTwoInts::Response> response)
```

Unlike Python, the callback does **not return** the response. It fills in the response object it receives through a pointer, and rclcpp sends it when the callback ends. Fields are accessed with `->` because both arguments are pointers.

`RCLCPP_INFO_STREAM` is used here because the fields are `int64_t`, whose `printf` format specifier differs between platforms. Streams handle any type.

### Build and test

```cmake
add_executable(add_two_ints_server src/add_two_ints_server.cpp)
ament_target_dependencies(add_two_ints_server rclcpp example_interfaces)
```

Add it to `install(TARGETS ...)`, rebuild, run it, and test it with the CLI:

```bash
$ ros2 run my_cpp_pkg add_two_ints_server
```

```bash
$ ros2 service call /add_two_ints example_interfaces/srv/AddTwoInts "{a: 2, b: 3}"
...
response:
example_interfaces.srv.AddTwoInts_Response(sum=5)
```

The Python client from Module 04 works with this server too: `ros2 run my_first_pkg add_two_ints_client 2 3`.

---

## Writing a One-Shot Client

Create `src/add_two_ints_client.cpp`:

```cpp
#include <chrono>
#include <cstdlib>
#include <memory>

#include "example_interfaces/srv/add_two_ints.hpp"
#include "rclcpp/rclcpp.hpp"

using example_interfaces::srv::AddTwoInts;
using namespace std::chrono_literals;

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);

  if (argc < 3) {
    RCLCPP_ERROR(rclcpp::get_logger("add_two_ints_client"), "Usage: add_two_ints_client X Y");
    rclcpp::shutdown();
    return 1;
  }

  auto node = rclcpp::Node::make_shared("add_two_ints_client");
  auto client = node->create_client<AddTwoInts>("add_two_ints");

  auto request = std::make_shared<AddTwoInts::Request>();
  request->a = std::atoll(argv[1]);
  request->b = std::atoll(argv[2]);

  while (!client->wait_for_service(1s)) {
    if (!rclcpp::ok()) {
      RCLCPP_ERROR(node->get_logger(), "Interrupted while waiting for the service");
      return 1;
    }
    RCLCPP_INFO(node->get_logger(), "Service not available, waiting...");
  }

  auto future = client->async_send_request(request);
  if (rclcpp::spin_until_future_complete(node, future) == rclcpp::FutureReturnCode::SUCCESS) {
    RCLCPP_INFO_STREAM(node->get_logger(), "Result: " << future.get()->sum);
  } else {
    RCLCPP_ERROR(node->get_logger(), "Failed to call service add_two_ints");
  }

  rclcpp::shutdown();
  return 0;
}
```

### How it works

This client is a short program that makes one call and exits, so it doesn't need a node class: `rclcpp::Node::make_shared(...)` creates a plain node directly.

| Line | Python equivalent |
|------|-------------------|
| `node->create_client<AddTwoInts>("add_two_ints")` | `self.create_client(AddTwoInts, 'add_two_ints')` |
| `client->wait_for_service(1s)` | `wait_for_service(timeout_sec=1.0)` |
| `std::make_shared<AddTwoInts::Request>()` | `AddTwoInts.Request()` |
| `client->async_send_request(request)` | `call_async(request)` |
| `rclcpp::spin_until_future_complete(node, future)` | the same, in rclpy |
| `future.get()->sum` | `future.result().sum` |

`spin_until_future_complete` returns a code, so you can check whether the call actually succeeded instead of assuming it did. The `rclcpp::ok()` check inside the waiting loop lets `Ctrl` + `C` end the program cleanly.

### Build and run

```cmake
add_executable(add_two_ints_client src/add_two_ints_client.cpp)
ament_target_dependencies(add_two_ints_client rclcpp example_interfaces)
```

```bash
$ ros2 run my_cpp_pkg add_two_ints_client 2 3
[INFO] [1730800200.000000000] [add_two_ints_client]: Result: 5
```

---

## Calling a Service from a Running Node: `turtle_spawner`

Inside a node that is already spinning, you must not block. rclcpp lets you pass a **callback** directly to `async_send_request`; it runs when the response arrives.

Create `src/turtle_spawner.cpp`:

```cpp
#include <chrono>
#include <cmath>
#include <memory>
#include <random>

#include "rclcpp/rclcpp.hpp"
#include "turtlesim/srv/spawn.hpp"

using namespace std::chrono_literals;
using turtlesim::srv::Spawn;

class TurtleSpawner : public rclcpp::Node
{
public:
  TurtleSpawner()
  : Node("turtle_spawner"), random_engine_(std::random_device{}())
  {
    this->declare_parameter("max_turtles", 5);
    client_ = this->create_client<Spawn>("spawn");
    timer_ = this->create_wall_timer(3s, std::bind(&TurtleSpawner::timer_callback, this));
  }

private:
  void timer_callback()
  {
    if (spawned_ >= this->get_parameter("max_turtles").as_int()) {
      timer_->cancel();
      RCLCPP_INFO(this->get_logger(), "Maximum number of turtles reached");
      return;
    }

    if (!client_->service_is_ready()) {
      RCLCPP_WARN(this->get_logger(), "/spawn is not available yet");
      return;
    }

    std::uniform_real_distribution<double> position(1.0, 10.0);
    std::uniform_real_distribution<double> angle(-M_PI, M_PI);

    auto request = std::make_shared<Spawn::Request>();
    request->x = position(random_engine_);
    request->y = position(random_engine_);
    request->theta = angle(random_engine_);

    client_->async_send_request(
      request,
      [this](rclcpp::Client<Spawn>::SharedFuture future) {
        RCLCPP_INFO(this->get_logger(), "Spawned %s", future.get()->name.c_str());
      });
    spawned_++;
  }

  rclcpp::Client<Spawn>::SharedPtr client_;
  rclcpp::TimerBase::SharedPtr timer_;
  std::mt19937 random_engine_;
  int64_t spawned_ = 0;
};

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<TurtleSpawner>());
  rclcpp::shutdown();
  return 0;
}
```

### What is new

- **`async_send_request(request, callback)`** sends the request and returns immediately. The lambda runs later, when the response arrives, receiving a `SharedFuture` whose `get()` returns the response. This is the C++ equivalent of `future.add_done_callback(...)` in Python.
- **`service_is_ready()`** checks availability without waiting.
- **Integer parameters** are read with `as_int()`, which returns an `int64_t`.
- **Random numbers** come from the `<random>` header: an engine (`std::mt19937`) seeded once, and distributions that produce values in a range.

```cmake
add_executable(turtle_spawner src/turtle_spawner.cpp)
ament_target_dependencies(turtle_spawner rclcpp turtlesim)
```

```bash
$ ros2 run turtlesim turtlesim_node
```

```bash
$ ros2 run my_cpp_pkg turtle_spawner
[INFO] [1730800303.000000000] [turtle_spawner]: Spawned turtle2
...
```

---

## Avoiding Deadlocks

Everything explained in [Python: Avoiding Deadlocks](../04-python/services.md#avoiding-deadlocks) applies to C++ as well. With the default single-threaded executor:

> Never wait for a response inside a callback, for example by calling `future.get()` or `future.wait()` right after `async_send_request`, or by calling `spin_until_future_complete` from inside a callback.

| Where you call | Use |
|----------------|-----|
| In `main()`, without a spinning executor | `async_send_request` + `spin_until_future_complete` |
| Inside a callback of a spinning node | `async_send_request(request, callback)` |

---

## Common Mistakes

| Symptom | Cause |
|---------|-------|
| Template error around `create_service` | Wrong callback signature, or only `_1` given to `std::bind` (two placeholders are needed) |
| The client receives default values | The server callback didn't fill in `response->...` |
| The node freezes after a call | `future.get()` or `spin_until_future_complete` inside a callback |
| `'M_PI' was not declared` | Include `<cmath>`. (On some compilers, `M_PI` also needs `_USE_MATH_DEFINES`) |
| `undefined reference to ...` at link time | The package is missing from `ament_target_dependencies` for that executable |

---

## Try It Yourself

1. Add a request counter to the C++ server and include it in the log message.
2. Make the client send five requests in a row and print all results.
3. Write a C++ `pen_switcher` that calls `/turtle1/set_pen` with a random color every 2 seconds.
4. Run the C++ server together with the Python client, then the Python server with the C++ client. Do you notice any difference?

---

⬅️ [Previous: Writing a Subscriber](subscriber.md) | [⬆️ C++ (rclcpp)](README.md) | [Next: Writing an Action Server and Client](actions.md) ➡️
