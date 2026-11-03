# Writing a Service Server and Client

Services implement **request/response** communication: a client sends a request, the server does some work and sends back a response (see [Services](../01-ros2-basics/services.md)). In this lesson you write a minimal server and client, then a node that calls turtlesim's `/spawn` service from code, and learn the one rule that prevents the most common service bug: never wait for a response inside a callback.

## Topics Covered

- Writing a service server
- Writing a service client
- Asynchronous calls and futures
- Waiting for a service to become available
- Calling services from inside a running node
- Avoiding deadlocks

---

## The Service Type: `AddTwoInts`

This lesson uses `example_interfaces/srv/AddTwoInts`, a simple type made for examples:

```bash
$ ros2 interface show example_interfaces/srv/AddTwoInts
int64 a
int64 b
---
int64 sum
```

The request has two integers; the response has their sum. Add the dependency to `package.xml`:

```xml
<depend>example_interfaces</depend>
```

---

## Writing a Service Server

Create `~/ros2_ws/src/my_first_pkg/my_first_pkg/add_two_ints_server.py`:

```python
import rclpy
from example_interfaces.srv import AddTwoInts
from rclpy.node import Node


class AddTwoIntsServer(Node):
    def __init__(self):
        super().__init__('add_two_ints_server')
        self.service = self.create_service(AddTwoInts, 'add_two_ints', self.add_callback)
        self.get_logger().info('Service /add_two_ints is ready')

    def add_callback(self, request, response):
        response.sum = request.a + request.b
        self.get_logger().info(f'{request.a} + {request.b} = {response.sum}')
        return response


def main():
    rclpy.init()
    node = AddTwoIntsServer()
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

### How it works

```python
self.service = self.create_service(AddTwoInts, 'add_two_ints', self.add_callback)
```

`create_service(service_type, service_name, callback)` registers the service. From now on, every request to `/add_two_ints` calls `add_callback`.

```python
def add_callback(self, request, response):
    response.sum = request.a + request.b
    return response
```

The callback receives **two** objects: the incoming `request` and an empty `response` prepared for you. Fill in the response fields and **return** it. Forgetting the `return` is a classic mistake: the client then receives nothing useful.

### Test it with the CLI

Register the executable (`'add_two_ints_server = my_first_pkg.add_two_ints_server:main'`), rebuild, and run it:

```bash
$ ros2 run my_first_pkg add_two_ints_server
[INFO] [1730700100.000000000] [add_two_ints_server]: Service /add_two_ints is ready
```

You don't need a client to test a server. Use the CLI from [ros2 service](../02-ros2-cli/ros2-service.md):

```bash
$ ros2 service call /add_two_ints example_interfaces/srv/AddTwoInts "{a: 2, b: 3}"
requester: making request: example_interfaces.srv.AddTwoInts_Request(a=2, b=3)

response:
example_interfaces.srv.AddTwoInts_Response(sum=5)
```

---

## Writing a Service Client

A client creates a request, sends it, and waits for the response. Create `my_first_pkg/add_two_ints_client.py`:

```python
import sys

import rclpy
from example_interfaces.srv import AddTwoInts
from rclpy.node import Node


class AddTwoIntsClient(Node):
    def __init__(self):
        super().__init__('add_two_ints_client')
        self.client = self.create_client(AddTwoInts, 'add_two_ints')
        while not self.client.wait_for_service(timeout_sec=1.0):
            self.get_logger().info('Service not available, waiting...')

    def send_request(self, a, b):
        request = AddTwoInts.Request()
        request.a = a
        request.b = b
        return self.client.call_async(request)


def main():
    rclpy.init()
    node = AddTwoIntsClient()
    future = node.send_request(int(sys.argv[1]), int(sys.argv[2]))
    rclpy.spin_until_future_complete(node, future)

    response = future.result()
    node.get_logger().info(f'Result: {response.sum}')

    node.destroy_node()
    rclpy.try_shutdown()


if __name__ == '__main__':
    main()
```

### How it works

```python
self.client = self.create_client(AddTwoInts, 'add_two_ints')
```

The type and name must match the server's.

```python
while not self.client.wait_for_service(timeout_sec=1.0):
    self.get_logger().info('Service not available, waiting...')
```

`wait_for_service` returns `True` as soon as the server is available, or `False` after the timeout. The loop keeps waiting and prints a message every second, so you can see what the node is doing.

```python
request = AddTwoInts.Request()
```

Each service type has a `Request` and a `Response` class.

```python
return self.client.call_async(request)
```

`call_async` sends the request and returns immediately with a **future**: a placeholder for a response that will arrive later.

```python
rclpy.spin_until_future_complete(node, future)
response = future.result()
```

`spin_until_future_complete` spins the node until the response has arrived, then `future.result()` returns it. This client is a short script that makes one call and exits, so it doesn't need `rclpy.spin`.

### Run it

Register `'add_two_ints_client = my_first_pkg.add_two_ints_client:main'`, rebuild, and with the server running:

```bash
$ ros2 run my_first_pkg add_two_ints_client 2 3
[INFO] [1730700110.000000000] [add_two_ints_client]: Result: 5
```

Stop the server and run the client again: it waits, printing `Service not available, waiting...`, until you start the server.

---

## Calling a Service from a Running Node: `turtle_spawner`

Most of the time, a client is not a one-shot script but part of a node that is already spinning, with timers and subscriptions. Let's write a node that spawns a new turtle at a random position every 3 seconds, up to a maximum number.

Create `my_first_pkg/turtle_spawner.py`:

```python
import math
import random

import rclpy
from rclpy.node import Node
from turtlesim.srv import Spawn


class TurtleSpawner(Node):
    def __init__(self):
        super().__init__('turtle_spawner')
        self.declare_parameter('max_turtles', 5)
        self.spawned = 0

        self.client = self.create_client(Spawn, 'spawn')
        self.timer = self.create_timer(3.0, self.timer_callback)

    def timer_callback(self):
        if self.spawned >= self.get_parameter('max_turtles').value:
            self.timer.cancel()
            self.get_logger().info('Maximum number of turtles reached')
            return

        if not self.client.service_is_ready():
            self.get_logger().warn('/spawn is not available yet')
            return

        request = Spawn.Request()
        request.x = random.uniform(1.0, 10.0)
        request.y = random.uniform(1.0, 10.0)
        request.theta = random.uniform(-math.pi, math.pi)

        future = self.client.call_async(request)
        future.add_done_callback(self.spawn_done_callback)
        self.spawned += 1

    def spawn_done_callback(self, future):
        try:
            response = future.result()
        except Exception as error:
            self.get_logger().error(f'Spawn failed: {error}')
            return
        self.get_logger().info(f'Spawned {response.name}')


def main():
    rclpy.init()
    node = TurtleSpawner()
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

### What is new

**`service_is_ready()`** checks whether the server is available *without waiting*. Inside a callback, you must not block, so instead of a `wait_for_service` loop, the node simply tries again on the next timer tick.

**`future.add_done_callback(...)`** registers a function to run when the response arrives. The timer callback returns immediately, the node keeps spinning, and `spawn_done_callback` is called later with the completed future. This is the correct way to call a service from inside a node.

**`self.timer.cancel()`** stops the timer once enough turtles exist.

Register it, rebuild, and run it next to turtlesim:

```bash
$ ros2 run turtlesim turtlesim_node
```

```bash
$ ros2 run my_first_pkg turtle_spawner
[INFO] [1730700203.000000000] [turtle_spawner]: Spawned turtle2
[INFO] [1730700206.000000000] [turtle_spawner]: Spawned turtle3
...
[INFO] [1730700218.000000000] [turtle_spawner]: Maximum number of turtles reached
```

---

## ⚠️ Avoiding Deadlocks

This is the most important rule in this lesson:

> **Never wait for a service response inside a callback**, whether with the synchronous `client.call()`, a loop that checks `future.done()`, or by spinning again with `spin_until_future_complete()`.

Why? With the default executor, callbacks run one at a time in a single thread (see [Subscriber](subscriber.md#how-callbacks-and-spinning-work)). The response arrives as an event that the executor must process, but the executor is busy running *your* callback, which is waiting for that very response. Nothing moves. The node freezes.

```text
 timer_callback() ──▶ waits for response ──▶ the response must be processed by
        ▲                                     the executor ... which is busy
        └──────────────── running timer_callback() ◀─────────────────┘
```

| Where you call | Use |
|----------------|-----|
| In `main()`, before or instead of `spin` | `call_async` + `spin_until_future_complete` |
| Inside a callback of a spinning node | `call_async` + `future.add_done_callback(...)` |

There are advanced ways to wait inside callbacks (multi-threaded executors and callback groups, touched on in [Actions](actions.md#executors-and-callback-groups)), but `add_done_callback` is simpler and works everywhere.

---

## Common Mistakes

| Symptom | Cause |
|---------|-------|
| The client receives an empty or default response | The server callback doesn't `return response` |
| The client waits forever | Server not running, name or type mismatch, or different `ROS_DOMAIN_ID` |
| The node freezes when calling a service | Waiting for the response inside a callback. Use `add_done_callback` |
| `TypeError: add_callback() takes 2 positional arguments but 3 were given` | The server callback must accept both `request` and `response` |
| `IndexError: list index out of range` in the client | You ran the client without its two number arguments |

---

## Try It Yourself

1. Extend the server so it counts the requests it has handled and includes the count in its log message.
2. Change `add_two_ints_client` so it sends five requests in a row (1 + 1, 2 + 2, ...) and prints all results.
3. Write a node `pen_switcher` that calls `/turtle1/set_pen` every 2 seconds with a random color. Use `ros2 interface show turtlesim/srv/SetPen` to find the request fields.
4. Combine `turtle_spawner` with `/kill`: after spawning the maximum number of turtles, remove them again one by one.
5. Deliberately create a deadlock: in `turtle_spawner`, replace the `call_async` and `add_done_callback` lines with the synchronous `response = self.client.call(request)`. Run it and observe that the node freezes after the first call, then revert the change.

---

⬅️ [Previous: Writing a Subscriber](subscriber.md) | [⬆️ Python (rclpy)](README.md) | [Next: Writing an Action Server and Client](actions.md) ➡️
