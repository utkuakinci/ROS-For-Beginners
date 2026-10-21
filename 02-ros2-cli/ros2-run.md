# ros2 run

`ros2 run` starts a single executable from a package. On its own it is simple, but combined with `--ros-args` it can rename a node, put it in a namespace, remap its topics, set its parameters, and change its log level, all without touching the code.

> 📖 Concepts: [Nodes](../01-ros2-basics/nodes.md), [Parameters](../01-ros2-basics/parameters.md)

## Topics Covered

- Basic usage and finding executables
- ROS arguments (`--ros-args`)
- Remapping names, namespaces, and node names
- Setting parameters and log levels
- Running nodes under other tools with `--prefix`
- Troubleshooting

---

## Basic Usage

```bash
ros2 run [--prefix <command>] <package> <executable> [program arguments] [--ros-args <ROS arguments>]
```

```bash
$ ros2 run turtlesim turtlesim_node
```

### Finding packages and executables

If you don't know what to run, `ros2 pkg` helps:

| Command | Purpose |
|---------|---------|
| `ros2 pkg list` | List all installed packages |
| `ros2 pkg executables <package>` | List the executables of a package |
| `ros2 pkg executables` | List the executables of all packages |
| `ros2 pkg prefix <package>` | Show where a package is installed |
| `ros2 pkg xml <package>` | Show a package's `package.xml` |

```bash
$ ros2 pkg executables turtlesim
turtlesim draw_square
turtlesim mimic
turtlesim turtle_teleop_key
turtlesim turtlesim_node

$ ros2 pkg prefix turtlesim
/opt/ros/jazzy
```

`Tab` completion works for both the package and the executable name.

---

## ROS Arguments: `--ros-args`

Everything after `--ros-args` is interpreted by ROS 2 instead of the program itself.

| Argument | Long form | Effect |
|----------|-----------|--------|
| `-r <from>:=<to>` | `--remap` | Remap a name (topic, service, action, node name, namespace) |
| `-p <name>:=<value>` | `--param` | Set a parameter |
| `--params-file <file>` | | Load parameters from a YAML file |
| `--log-level <level>` | | Set the log level (`debug`, `info`, `warn`, `error`, `fatal`) |
| `--disable-stdout-logs` | | Do not print log messages to the terminal |
| `--disable-rosout-logs` | | Do not publish log messages on `/rosout` |

Each option can be repeated:

```bash
$ ros2 run turtlesim turtlesim_node --ros-args -p background_r:=0 -p background_g:=0 -r __ns:=/robot1
```

If you need to pass normal arguments to the program after the ROS arguments, end the ROS section with `--`:

```bash
$ ros2 run my_pkg my_node --ros-args -p rate:=10.0 -- --my-own-flag
```

---

## Remapping

Remapping replaces one name with another when the node starts.

### Node name and namespace

Two special names change the node itself:

```bash
$ ros2 run turtlesim turtlesim_node --ros-args -r __node:=sim         # node is now /sim
$ ros2 run turtlesim turtlesim_node --ros-args -r __ns:=/robot1       # node is now /robot1/turtlesim
```

A namespace also prefixes all of the node's relative topic, service, and action names: `/robot1/turtle1/cmd_vel`, `/robot1/spawn`, and so on.

### Topics, services, and actions

```bash
$ ros2 run turtlesim turtle_teleop_key --ros-args -r /turtle1/cmd_vel:=/cmd_vel
```

The teleop node now publishes on `/cmd_vel` instead of `/turtle1/cmd_vel`. This is how you connect a generic node to the names your system uses, for example to drive a real robot that listens on `/cmd_vel` with the turtlesim teleop.

Remapping works on services and actions in exactly the same way.

> 💡 Remapping is matched against the name **the node uses internally**. If a remap seems to have no effect, run `ros2 node info` on the node *without* the remap to see its original names.

---

## Parameters

```bash
# Individual values
$ ros2 run turtlesim turtlesim_node --ros-args -p background_r:=255 -p background_b:=0

# From a file
$ ros2 run turtlesim turtlesim_node --ros-args --params-file turtlesim.yaml

# Both: values given later override earlier ones
$ ros2 run turtlesim turtlesim_node --ros-args --params-file turtlesim.yaml -p background_r:=0
```

Values follow the same YAML typing rules as `ros2 param set`: `1` is an integer, `1.0` a double, `"[1, 2]"` an array.

---

## Log Levels

```bash
# Debug output for everything in the process
$ ros2 run turtlesim turtlesim_node --ros-args --log-level debug

# Debug output only for one logger (usually the node's name)
$ ros2 run turtlesim turtlesim_node --ros-args --log-level turtlesim:=debug
```

The per-logger form keeps the output readable when a library underneath also logs heavily at `debug` level.

---

## Running Under Another Tool: `--prefix`

`--prefix` puts a command in front of the executable. This lets you run a node inside a debugger, a profiler, or a separate terminal window:

```bash
# Start the node inside the GDB debugger (C++ nodes)
$ ros2 run --prefix 'gdb -ex run --args' my_pkg my_node

# Check a C++ node for memory errors
$ ros2 run --prefix 'valgrind' my_pkg my_node

# Open the node in its own terminal window
$ ros2 run --prefix 'xterm -e' turtlesim turtle_teleop_key
```

You will use this in [Module 11](../11-debugging/README.md).

---

## Troubleshooting

| Error | Likely cause |
|-------|--------------|
| `ros2: command not found` | ROS 2 is not sourced in this terminal: `source /opt/ros/jazzy/setup.bash` |
| `Package '<name>' not found` | Typo, package not installed, or your workspace is not sourced (`source install/setup.bash`) |
| `No executable found` | Typo in the executable name, or a Python package's `entry_points` is missing it (see [setup.py](../03-workspaces-and-packages/setup-py.md)) |
| A remap or parameter is ignored | It was placed **before** `--ros-args`, or the name does not match the node's internal name |

---

## Try It Yourself

1. Find all executables in the `demo_nodes_cpp` package and run one you have not tried.
2. Start turtlesim as `/arena/sim` (namespace `/arena`, node name `sim`) with a white background, using a single command.
3. Remap the teleop node so it drives the turtle from the previous exercise.
4. Start the talker from `demo_nodes_cpp` so that it publishes on `/news` instead of `/chatter`, and the listener from `demo_nodes_py` so that it listens on `/news`.
5. Start turtlesim with `--log-level turtlesim:=debug` and compare the output with a normal start.

---

⬅️ [Previous: ros2 param](ros2-param.md) | [⬆️ CLI Tools](README.md) | [Next: ros2 launch](ros2-launch.md) ➡️
