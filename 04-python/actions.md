# Writing an Action Server and Client

Actions handle long-running tasks: the client sends a **goal**, the server reports **feedback** while it works, sends a **result** at the end, and the goal can be **cancelled** at any time (see [Actions](../01-ros2-basics/actions.md)). In this lesson you write an action server that computes a Fibonacci sequence step by step, and an action client that rotates the turtle in turtlesim.

## Topics Covered

- Writing an action server: goal, cancel, and execute callbacks
- Publishing feedback and returning a result
- Handling cancellation
- Executors and callback groups
- Writing an action client
- Following a goal through its life cycle

---

## The Action Type: `Fibonacci`

The server uses `example_interfaces/action/Fibonacci`:

```bash
$ ros2 interface show example_interfaces/action/Fibonacci
# Goal
int32 order
---
# Result
int32[] sequence
---
# Feedback
int32[] sequence
```

The goal says how many numbers to compute; the feedback carries the sequence computed so far; the result contains the final sequence. You already depend on `example_interfaces` from the [previous lesson](services.md#the-service-type-addtwoints).

---

## Writing an Action Server

Create `~/ros2_ws/src/my_first_pkg/my_first_pkg/fibonacci_server.py`:

```python
import time

import rclpy
from example_interfaces.action import Fibonacci
from rclpy.action import ActionServer, CancelResponse, GoalResponse
from rclpy.callback_groups import ReentrantCallbackGroup
from rclpy.executors import MultiThreadedExecutor
from rclpy.node import Node


class FibonacciServer(Node):
    def __init__(self):
        super().__init__('fibonacci_server')
        self.action_server = ActionServer(
            self,
            Fibonacci,
            'fibonacci',
            execute_callback=self.execute_callback,
            goal_callback=self.goal_callback,
            cancel_callback=self.cancel_callback,
            callback_group=ReentrantCallbackGroup(),
        )
        self.get_logger().info('Action server /fibonacci is ready')

    def goal_callback(self, goal_request):
        if not 1 <= goal_request.order <= 45:
            self.get_logger().warn(f'Rejecting goal with order {goal_request.order}')
            return GoalResponse.REJECT
        self.get_logger().info(f'Accepting goal with order {goal_request.order}')
        return GoalResponse.ACCEPT

    def cancel_callback(self, goal_handle):
        self.get_logger().info('Received cancel request')
        return CancelResponse.ACCEPT

    def execute_callback(self, goal_handle):
        feedback = Fibonacci.Feedback()
        feedback.sequence = [0, 1]

        for i in range(1, goal_handle.request.order):
            if goal_handle.is_cancel_requested:
                goal_handle.canceled()
                self.get_logger().info('Goal canceled')
                return Fibonacci.Result(sequence=feedback.sequence)

            feedback.sequence.append(feedback.sequence[i] + feedback.sequence[i - 1])
            goal_handle.publish_feedback(feedback)
            self.get_logger().info(f'Feedback: {list(feedback.sequence)}')
            time.sleep(1.0)  # simulate a slow task

        goal_handle.succeed()
        return Fibonacci.Result(sequence=feedback.sequence)


def main():
    rclpy.init()
    node = FibonacciServer()
    executor = MultiThreadedExecutor()
    try:
        rclpy.spin(node, executor=executor)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.try_shutdown()


if __name__ == '__main__':
    main()
```

### The three server callbacks

An action server is built from three callbacks, each answering one question:

| Callback | Question | Returns |
|----------|----------|---------|
| `goal_callback(goal_request)` | Do I accept this goal? | `GoalResponse.ACCEPT` or `GoalResponse.REJECT` |
| `cancel_callback(goal_handle)` | Do I allow this goal to be cancelled? | `CancelResponse.ACCEPT` or `CancelResponse.REJECT` |
| `execute_callback(goal_handle)` | Do the work | A `Result` message |

`goal_callback` and `cancel_callback` are optional. Without them, every goal and every cancel request is accepted. Validating goals in `goal_callback` is good practice: reject what you cannot do *before* starting, instead of failing halfway.

### Inside `execute_callback`

The `goal_handle` is your connection to one specific goal:

| Call | Effect |
|------|--------|
| `goal_handle.request` | The goal message sent by the client (`goal_handle.request.order`) |
| `goal_handle.publish_feedback(msg)` | Send a feedback message to the client |
| `goal_handle.is_cancel_requested` | `True` once the client asked to cancel and the cancel was accepted |
| `goal_handle.succeed()` | Mark the goal as succeeded |
| `goal_handle.canceled()` | Mark the goal as cancelled |
| `goal_handle.abort()` | Mark the goal as failed |

Mark the goal's final state, then `return` the result message. Cancellation is **cooperative**: the server checks `is_cancel_requested` regularly and stops itself. If your loop never checks, a cancel request has no effect.

> 📝 `execute_callback` is the one place where long-running work and `time.sleep()` are expected. It is designed to run for as long as the task takes, which is why this server needs the executor setup explained next.

---

## Executors and Callback Groups

Recall from [Subscriber](subscriber.md#how-callbacks-and-spinning-work) that, by default, a node's callbacks run one at a time. For an action server that is a problem: while `execute_callback` runs for several seconds, a cancel request could not be processed until the goal had already finished.

Two settings fix this:

- **`MultiThreadedExecutor`** runs callbacks in several threads, so more than one can run at the same time.
- **`ReentrantCallbackGroup`** allows the callbacks in the group to run in parallel with each other.

| Setting | Default | Alternative |
|---------|---------|-------------|
| Executor | `SingleThreadedExecutor`: one callback at a time | `MultiThreadedExecutor`: several at once |
| Callback group | `MutuallyExclusiveCallbackGroup`: callbacks in the group never overlap | `ReentrantCallbackGroup`: callbacks in the group may overlap |

Both are needed: a multi-threaded executor only runs callbacks in parallel if their callback group allows it.

> ⚠️ Running callbacks in parallel means two callbacks can modify the same attribute at the same time. Keep shared state minimal, or protect it with a `threading.Lock`. For the simple nodes in this guide, use the defaults unless you need concurrency, as in this action server.

### Test the server with the CLI

Register `'fibonacci_server = my_first_pkg.fibonacci_server:main'`, rebuild, and run it:

```bash
$ ros2 run my_first_pkg fibonacci_server
```

```bash
$ ros2 action send_goal --feedback /fibonacci example_interfaces/action/Fibonacci "{order: 5}"
Waiting for an action server to become available...
Sending goal:
     order: 5

Goal accepted with ID: 3f6a...

Feedback:
    sequence:
- 0
- 1
- 1

...

Result:
    sequence:
- 0
- 1
- 1
- 2
- 3
- 5

Goal finished with status: SUCCEEDED
```

Now try:

- Press `Ctrl` + `C` during a goal: the server logs `Received cancel request` and `Goal canceled`.
- Send `{order: 100}`: the goal is rejected by `goal_callback`.

---

## Writing an Action Client: `rotate_client`

The client side follows a goal through its whole life: send it, learn whether it was accepted, receive feedback, and receive the result. Let's write a client for turtlesim's `rotate_absolute` action, which takes a target heading in **degrees** from the command line.

Create `my_first_pkg/rotate_client.py`:

```python
import math
import sys

import rclpy
from action_msgs.msg import GoalStatus
from rclpy.action import ActionClient
from rclpy.node import Node
from turtlesim.action import RotateAbsolute


class RotateClient(Node):
    def __init__(self):
        super().__init__('rotate_client')
        self.client = ActionClient(self, RotateAbsolute, 'turtle1/rotate_absolute')
        self.done = False

    def send_goal(self, degrees):
        goal = RotateAbsolute.Goal()
        goal.theta = math.radians(degrees)

        self.get_logger().info('Waiting for action server...')
        self.client.wait_for_server()

        self.get_logger().info(f'Sending goal: rotate to {degrees:.1f}°')
        send_future = self.client.send_goal_async(goal, feedback_callback=self.feedback_callback)
        send_future.add_done_callback(self.goal_response_callback)

    def goal_response_callback(self, future):
        goal_handle = future.result()
        if not goal_handle.accepted:
            self.get_logger().warn('Goal rejected')
            self.done = True
            return

        self.get_logger().info('Goal accepted')
        result_future = goal_handle.get_result_async()
        result_future.add_done_callback(self.result_callback)

    def feedback_callback(self, feedback_msg):
        remaining = math.degrees(feedback_msg.feedback.remaining)
        self.get_logger().info(f'Remaining: {remaining:.1f}°')

    def result_callback(self, future):
        result = future.result()
        if result.status == GoalStatus.STATUS_SUCCEEDED:
            rotated = math.degrees(result.result.delta)
            self.get_logger().info(f'Succeeded, rotated by {rotated:.1f}°')
        else:
            self.get_logger().warn(f'Goal finished with status {result.status}')
        self.done = True


def main():
    rclpy.init()
    node = RotateClient()
    node.send_goal(float(sys.argv[1]))
    try:
        while rclpy.ok() and not node.done:
            rclpy.spin_once(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.try_shutdown()


if __name__ == '__main__':
    main()
```

### The life of a goal, step by step

```text
send_goal_async() ──▶ goal_response_callback()   accepted or rejected?
                              │ accepted
                              ▼
                       get_result_async()
                              │
        feedback_callback() ◀─┤  called repeatedly while the server works
                              ▼
                       result_callback()          final status and result
```

| Step | What happens |
|------|--------------|
| `ActionClient(self, RotateAbsolute, 'turtle1/rotate_absolute')` | Create the client; type and name must match the server |
| `wait_for_server()` | Block until the server is available. Fine here, because we are not inside a callback |
| `send_goal_async(goal, feedback_callback=...)` | Send the goal and register the feedback callback. Returns a future |
| `goal_response_callback` | The future completes with a **goal handle**. Check `goal_handle.accepted` |
| `goal_handle.get_result_async()` | Ask for the result. Returns another future |
| `feedback_callback` | Called for each feedback message; the data is in `feedback_msg.feedback` |
| `result_callback` | `future.result()` holds the final `status` and the `result` message |

As with services, everything is asynchronous: no callback waits for anything (see [Avoiding Deadlocks](services.md#avoiding-deadlocks)). The `done` flag lets `main()` spin only until the goal has finished, then exit cleanly.

The status codes come from `action_msgs/msg/GoalStatus`: `STATUS_SUCCEEDED`, `STATUS_CANCELED`, and `STATUS_ABORTED` are the ones you will see. Add `<depend>action_msgs</depend>` to `package.xml`, since the client imports it.

### Cancelling from the client

To cancel a goal from code, keep the goal handle and call `cancel_goal_async()` on it:

```python
cancel_future = goal_handle.cancel_goal_async()
cancel_future.add_done_callback(self.cancel_done_callback)
```

### Run it

Register `'rotate_client = my_first_pkg.rotate_client:main'`, rebuild, and run it next to turtlesim:

```bash
$ ros2 run turtlesim turtlesim_node
```

```bash
$ ros2 run my_first_pkg rotate_client 90
[INFO] [rotate_client]: Waiting for action server...
[INFO] [rotate_client]: Sending goal: rotate to 90.0°
[INFO] [rotate_client]: Goal accepted
[INFO] [rotate_client]: Remaining: 88.2°
[INFO] [rotate_client]: Remaining: 86.4°
...
[INFO] [rotate_client]: Succeeded, rotated by -90.0°
```

While a goal is running, start another `rotate_client` with a different angle in a second terminal. Turtlesim aborts the first goal, and the first client reports a status other than succeeded.

---

## Common Mistakes

| Symptom | Cause |
|---------|-------|
| Cancel requests are only handled after the goal finishes | Single-threaded executor or default callback group. Use `MultiThreadedExecutor` and `ReentrantCallbackGroup` |
| Cancelling has no effect | `execute_callback` never checks `goal_handle.is_cancel_requested` |
| The goal ends as aborted instead of succeeded | `execute_callback` returned without calling `goal_handle.succeed()` |
| The client waits forever | Server not running, or action name or type mismatch. Check with `ros2 action info` |
| `AttributeError: ... has no attribute 'remaining'` | In the feedback callback, the data is in `feedback_msg.feedback.remaining` |
| The client never exits | Nothing ends the spin loop. Set a flag in the result callback as shown |

---

## Try It Yourself

1. Change the Fibonacci server so it aborts the goal (`goal_handle.abort()`) if the order is odd, and observe the status in the CLI.
2. Write a `fibonacci_client` that sends a goal with an order given on the command line and prints each feedback message.
3. Extend `fibonacci_client` so that it cancels the goal after 3 feedback messages.
4. Write a `look_around` node that uses the `rotate_absolute` action to make the turtle face up, left, down, and right in sequence, sending the next goal from the previous goal's result callback.
5. Remove `MultiThreadedExecutor` and the callback group from the Fibonacci server, then try to cancel a goal with `Ctrl` + `C`. When is the cancel request handled?

---

⬅️ [Previous: Writing a Service Server and Client](services.md) | [⬆️ Python (rclpy)](README.md) | [Next: Custom Interfaces](custom-messages.md) ➡️
