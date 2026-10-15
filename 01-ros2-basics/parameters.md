# Parameters

Hard-coding values like a robot's maximum speed or a sensor's frame name into your code means editing and rebuilding every time something changes. **Parameters** solve this: they are named configuration values that belong to a node and can be set at startup or changed while the node is running.

## Topics Covered

- What parameters are
- Listing, reading, and describing parameters
- Changing parameters at runtime
- Setting parameters at startup
- Parameter files (YAML)
- Parameter types

---

## What Are Parameters?

Each node owns its own set of parameters. You can think of them as the node's settings.

```text
┌───────────────────────────────┐
│          /turtlesim           │
│ ───────────────────────────── │
│  background_r = 69            │
│  background_g = 86            │
│  background_b = 255           │
│  use_sim_time = false         │
└───────────────────────────────┘
```

Typical uses:

- Tuning values: maximum speed, controller gains, timeouts.
- Hardware settings: serial port name, camera resolution, publish rate.
- Names: which frame or topic to use.

A node **declares** the parameters it accepts, with a type and a default value. You will declare parameters in your own nodes in [Module 04](../04-python/README.md); here you will explore them from the command line.

---

## Listing Parameters

Start turtlesim, then:

```bash
$ ros2 param list
/turtlesim:
  background_b
  background_g
  background_r
  qos_overrides./parameter_events.publisher.depth
  qos_overrides./parameter_events.publisher.durability
  qos_overrides./parameter_events.publisher.history
  qos_overrides./parameter_events.publisher.reliability
  use_sim_time
  ...
```

Without a node name, `ros2 param list` shows the parameters of every running node. Pass a node name to see only that one: `ros2 param list /turtlesim`.

Every node has `use_sim_time`, which makes the node follow simulation time instead of the computer's clock. It becomes important when you work with Gazebo and recorded data.

---

## Reading a Parameter

```bash
$ ros2 param get /turtlesim background_g
Integer value is: 86
```

`ros2 param describe` shows a parameter's type, description, and constraints, if the node provides them:

```bash
$ ros2 param describe /turtlesim background_r
Parameter name: background_r
  Type: integer
  Description: Red channel of the background color
  Constraints:
    Min value: 0
    Max value: 255
    Step: 1
```

---

## Changing a Parameter at Runtime

```bash
$ ros2 param set /turtlesim background_r 150
Set parameter successful
```

The turtlesim background changes color immediately. The node reacts to the change because it registered a callback for parameter updates. Not every node does that; some read their parameters only once at startup, so changing them at runtime has no effect until a restart.

Parameters are **typed**. Setting a value of the wrong type fails:

```bash
$ ros2 param set /turtlesim background_r red
Setting parameter failed: ...
```

> ⚠️ Changes made with `ros2 param set` live only in the running node. Stop the node and they are gone. To keep them, save them to a file (next section) or set them at startup.

---

## Saving and Loading Parameter Files

`ros2 param dump` prints all of a node's current parameters in YAML. Redirect it to a file to save them:

```bash
$ ros2 param dump /turtlesim > turtlesim.yaml
$ cat turtlesim.yaml
/turtlesim:
  ros__parameters:
    background_b: 255
    background_g: 86
    background_r: 150
    qos_overrides:
      ...
    use_sim_time: false
```

The structure is always:

```yaml
<node_name>:
  ros__parameters:
    <parameter_name>: <value>
```

> 📝 Note the **two** underscores in `ros__parameters`. A typo here is a common reason a parameter file seems to be ignored.

Load a file into a running node:

```bash
$ ros2 param load /turtlesim turtlesim.yaml
```

---

## Setting Parameters at Startup

Usually you want a node to start with the right values. There are two ways.

**Individual values** with `-p` after `--ros-args`:

```bash
$ ros2 run turtlesim turtlesim_node --ros-args -p background_r:=255 -p background_g:=0 -p background_b:=0
```

**A parameter file** with `--params-file`:

```bash
$ ros2 run turtlesim turtlesim_node --ros-args --params-file turtlesim.yaml
```

Parameter files are the standard way to configure real robots. Large stacks like Nav2 are configured almost entirely through YAML files with hundreds of parameters. You can edit a file by hand and keep it in Git together with your code.

A file can configure several nodes at once, and the wildcard `/**` applies values to all nodes:

```yaml
/**:
  ros__parameters:
    use_sim_time: true

/turtlesim:
  ros__parameters:
    background_r: 0
    background_g: 128
    background_b: 0
```

---

## Parameter Types

| Type | Example YAML value |
|------|--------------------|
| `bool` | `true` |
| `integer` | `42` |
| `double` | `0.5` |
| `string` | `"base_link"` |
| `byte_array` | `[0x01, 0x02]` |
| `bool_array` | `[true, false]` |
| `integer_array` | `[1, 2, 3]` |
| `double_array` | `[0.1, 0.2]` |
| `string_array` | `["left", "right"]` |

> ⚠️ `1` is an integer and `1.0` is a double. If a node declares a `double` parameter, setting it to `1` fails with a type error. Write `1.0` instead.

---

## How Parameters Work Under the Hood

Remember the `/turtlesim/get_parameters` and `/turtlesim/set_parameters` services from the [Services](services.md) lesson? `ros2 param` simply calls those services. Every change is also announced on the `/parameter_events` topic, so other nodes can react:

```bash
$ ros2 topic echo /parameter_events
```

Run this, change a parameter in another terminal, and watch the event appear.

---

## Try It Yourself

1. Change the turtlesim background to black using three `ros2 param set` commands.
2. Dump the parameters to `dark.yaml`, stop turtlesim, and start it again with `--params-file dark.yaml`. Is the background still black?
3. Edit `dark.yaml` by hand to make the background yellow (red 255, green 255, blue 0) and start turtlesim with it.
4. Try setting `background_r` to `300`. What happens, and which part of `ros2 param describe` explains it?
5. Start turtlesim in the namespace `/robot1`. What node name must you use in the YAML file for it to apply?

---

⬅️ [Previous: Actions](actions.md) | [⬆️ ROS 2 Core Concepts](README.md) | [Next: Launch Files](launch-files.md) ➡️
