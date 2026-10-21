# ros2 service

`ros2 service` lets you discover the services in a system and call them from the terminal. It is the quickest way to trigger an action on a node (reset, save, switch mode) or to test a service server you have just written.

> 📖 Concepts: [Services](../01-ros2-basics/services.md)

## Topics Covered

- Listing and finding services
- Inspecting service types
- Calling services, once or repeatedly
- Troubleshooting service calls

---

## Command Overview

| Command | Purpose |
|---------|---------|
| `ros2 service list` | List all services |
| `ros2 service type <service>` | Print a service's type |
| `ros2 service find <type>` | List services that use a given type |
| `ros2 service call <service> <type> [data]` | Call a service |

Use `ros2 interface show <type>` to see a service's request and response fields (see [ros2 topic](ros2-topic.md#ros2-interface-inspecting-message-types)).

The examples below assume turtlesim is running.

---

## Listing and Finding Services

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
...
```

| Option | Effect |
|--------|--------|
| `-t`, `--show-types` | Show each service's type |
| `-c`, `--count-services` | Print only the number of services |
| `--include-hidden-services` | Also show hidden services (for example, action internals) |

```bash
$ ros2 service type /spawn
turtlesim/srv/Spawn

$ ros2 service find std_srvs/srv/Empty
/clear
/reset
```

> 💡 Every node has six parameter services (`<node>/get_parameters`, `<node>/set_parameters`, ...). To hide the noise, filter them out: `ros2 service list | grep -v parameters`.

---

## Inspecting a Service Type

```bash
$ ros2 interface show turtlesim/srv/Spawn
float32 x
float32 y
float32 theta
string name # Optional.  A unique name will be created and returned if this is empty
---
string name
```

Everything above `---` is the **request**; everything below is the **response**. Generate a ready-to-edit request with `ros2 interface proto`:

```bash
$ ros2 interface proto turtlesim/srv/Spawn
"x: 0.0
y: 0.0
theta: 0.0
name: ''
"
```

---

## `ros2 service call`

```bash
ros2 service call <service> <type> "<request in YAML>"
```

```bash
$ ros2 service call /spawn turtlesim/srv/Spawn "{x: 2.0, y: 2.0, theta: 0.2, name: 'leo'}"
requester: making request: turtlesim.srv.Spawn_Request(x=2.0, y=2.0, theta=0.2, name='leo')

response:
turtlesim.srv.Spawn_Response(name='leo')
```

If the request has no fields, or you want all defaults, the data can be omitted:

```bash
$ ros2 service call /clear std_srvs/srv/Empty
```

| Option | Effect |
|--------|--------|
| `-r`, `--rate <Hz>` | Call the service repeatedly at this rate until `Ctrl` + `C` |

```bash
# Clear the drawing once per second
$ ros2 service call -r 1 /clear std_srvs/srv/Empty
```

### Useful service types to recognize

| Type | Request | Response | Typical use |
|------|---------|----------|-------------|
| `std_srvs/srv/Empty` | nothing | nothing | Trigger something: reset, clear |
| `std_srvs/srv/Trigger` | nothing | `bool success`, `string message` | Trigger with a status report |
| `std_srvs/srv/SetBool` | `bool data` | `bool success`, `string message` | Turn something on or off |

```bash
# Example with a hypothetical motor driver node
$ ros2 service call /motors/enable std_srvs/srv/SetBool "{data: true}"
```

---

## Troubleshooting

| Symptom | Likely cause |
|---------|--------------|
| `waiting for service to become available...` forever | The server is not running, the name is wrong (check the namespace), or `ROS_DOMAIN_ID` differs |
| `The passed service type is invalid` | Typo in the type, or the package defining it is not installed or sourced |
| An error about unknown or wrong fields | The YAML does not match the request. Check it with `ros2 interface show` |
| The call returns, but nothing happens | Read the response: many services report failure in a `success` field instead of an error |
| The call takes very long | The server is doing slow work. That task should probably be an action |

---

## Try It Yourself

1. Use `ros2 service find` to list every service of type `std_srvs/srv/Empty`, then call each one and describe what it does.
2. Spawn a turtle with an empty name. What name does the response contain?
3. Use `ros2 interface proto` to generate a `SetPen` request, edit it, and use it to make the pen green.
4. Call `/turtle1/teleport_relative` with `-r 2` so the turtle steps forward twice per second. Find the right field values with `ros2 interface show`.
5. Call `/kill` on a turtle that does not exist. What happens?

---

⬅️ [Previous: ros2 topic](ros2-topic.md) | [⬆️ CLI Tools](README.md) | [Next: ros2 action](ros2-action.md) ➡️
