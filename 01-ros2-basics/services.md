# Services

Topics are great for streams of data, but sometimes a node needs to **ask** another node to do something and get an **answer** back. That is what **services** are for: a request/response pattern, like calling a function on another node.

## Topics Covered

- Servers and clients
- Service types (request and response)
- Inspecting services (`list`, `type`, `find`)
- Calling services from the command line
- Topics vs services
- When to use a service

---

## Servers and Clients

A service has one **server**, the node that provides it, and any number of **clients**, nodes that call it. Each call is a pair of messages: a **request** from the client and a **response** from the server.

```text
┌──────────────┐  ── request:  {x: 2, y: 2, name: ''} ──▶  ┌──────────────┐
│    Client    │                                           │    Server    │
│ (any node)   │  ◀── response: {name: 'turtle2'} ───────  │ (/turtlesim) │
└──────────────┘                                           └──────────────┘
```

Key differences from topics:

- A service is **two-way**: every request gets exactly one response.
- It is **on demand**: nothing happens until a client makes a call.
- It is **one-to-one** per call: each response goes back only to the client that asked.

---

## Listing Services

Start turtlesim:

```bash
$ ros2 run turtlesim turtlesim_node
```

Then in another terminal:

```bash
$ ros2 service list
/clear
/kill
/reset
/spawn
/turtle1/set_pen
/turtle1/teleport_absolute
/turtle1/teleport_relative
/turtlesim/describe_parameters
/turtlesim/get_parameter_types
/turtlesim/get_parameters
/turtlesim/list_parameters
/turtlesim/set_parameters
/turtlesim/set_parameters_atomically
```

The `/turtlesim/..._parameters` services exist on every node; they are how parameters work under the hood (see [Parameters](parameters.md)). The others are specific to turtlesim.

---

## Service Types

Like topics, every service has a **type**. Show it with `ros2 service type`, or add `-t` to the list:

```bash
$ ros2 service type /spawn
turtlesim/srv/Spawn

$ ros2 service list -t
/clear [std_srvs/srv/Empty]
/kill [turtlesim/srv/Kill]
/reset [std_srvs/srv/Empty]
/spawn [turtlesim/srv/Spawn]
...
```

A service type has two parts, separated by `---`: the **request** above, the **response** below.

```bash
$ ros2 interface show turtlesim/srv/Spawn
float32 x
float32 y
float32 theta
string name # Optional.  A unique name will be created and returned if this is empty
---
string name
```

So `/spawn` takes a position, an angle, and an optional name, and returns the name of the new turtle.

Some services need no data at all. `std_srvs/srv/Empty` has an empty request and an empty response; the call itself is the message:

```bash
$ ros2 interface show std_srvs/srv/Empty
---
```

You can also search for services by type:

```bash
$ ros2 service find std_srvs/srv/Empty
/clear
/reset
```

---

## Calling Services

The syntax is:

```bash
ros2 service call <service_name> <service_type> "<request in YAML>"
```

### Spawn a second turtle

```bash
$ ros2 service call /spawn turtlesim/srv/Spawn "{x: 2.0, y: 2.0, theta: 0.2, name: ''}"
requester: making request: turtlesim.srv.Spawn_Request(x=2.0, y=2.0, theta=0.2, name='')

response:
turtlesim.srv.Spawn_Response(name='turtle2')
```

A new turtle appears in the bottom left. Check `ros2 topic list`: it brought its own topics, such as `/turtle2/cmd_vel`.

### Change the pen

Each turtle draws a line as it moves. Make `turtle1`'s pen red and thick:

```bash
$ ros2 service call /turtle1/set_pen turtlesim/srv/SetPen "{r: 255, g: 0, b: 0, width: 5, off: 0}"
```

Drive the turtle (with `turtle_teleop_key` or `ros2 topic pub`) and the line is now red.

### Teleport

```bash
$ ros2 service call /turtle1/teleport_absolute turtlesim/srv/TeleportAbsolute "{x: 1.0, y: 9.0, theta: 0.0}"
```

### Clear and reset

```bash
$ ros2 service call /clear std_srvs/srv/Empty       # Erase the drawings
$ ros2 service call /reset std_srvs/srv/Empty       # Back to the start: one turtle, centered
```

> 💡 Just like with topics, press `Tab` after the service type to get a request template autocompleted.

---

## Waiting for a Server

If you call a service whose server is not running, the client waits:

```bash
$ ros2 service call /does_not_exist std_srvs/srv/Empty
waiting for service to become available...
```

It keeps waiting until the server appears or you press `Ctrl` + `C`. In your own code you will handle this explicitly by waiting for the service with a timeout.

---

## Topics vs Services

| | Topic | Service |
|---|---|---|
| **Direction** | One-way | Request and response |
| **Trigger** | Publisher sends whenever it wants | Client calls on demand |
| **Participants** | Many publishers, many subscribers | One server, many clients |
| **Typical data** | Continuous streams | Occasional commands and queries |
| **Example** | `/turtle1/pose` | `/spawn` |

---

## When to Use a Service

Use a service when:

- You need a **response**, or confirmation that something happened.
- The request is **occasional**, not a continuous stream.
- The server can answer **quickly** (milliseconds to a second or so).

Typical examples: resetting a simulation, switching a robot's mode, querying a value, saving a map, or triggering a calibration.

Do **not** use a service for work that takes a long time. The client is stuck waiting with no progress updates and no way to cancel. For long tasks, use an [action](actions.md).

---

## Try It Yourself

1. Spawn three extra turtles at different positions with names of your choice.
2. Remove one of them using the `/kill` service. Use `ros2 interface show` to find out what request it needs.
3. Give each turtle a different pen color, then drive them with `ros2 topic pub` on their own `cmd_vel` topics.
4. Use `/turtle1/set_pen` with `off: 1`. What happens when the turtle moves?
5. Draw a square using only `/turtle1/teleport_absolute` calls.

---

⬅️ [Previous: Topics](topics.md) | [⬆️ ROS 2 Core Concepts](README.md) | [Next: Actions](actions.md) ➡️
