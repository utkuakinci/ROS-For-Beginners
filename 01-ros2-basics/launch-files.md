# Launch Files

By now you have been opening a new terminal for every node and typing long commands with namespaces, remappings, and parameters. A real robot runs dozens of nodes, so doing this by hand quickly becomes impractical. **Launch files** describe a whole system in one file and start it with a single command.

## Topics Covered

- Why launch files exist
- Running launch files with `ros2 launch`
- Python launch file structure
- Namespaces, parameters, and remapping in launch files
- Launch arguments
- Including other launch files

---

## Why Launch Files?

A launch file lets you:

- **Start many nodes at once** with one command, and stop them all with one `Ctrl` + `C`.
- **Configure** each node: name, namespace, parameters, remappings.
- **Parameterize** the system with arguments (`use_sim:=true`, `robot_name:=robot1`).
- **Reuse** other launch files, for example the official Nav2 launch file inside your own.
- **Document** how your system is put together, in a file you can keep in Git.

ROS 2 launch files can be written in **Python**, **XML**, or **YAML**. This guide uses Python, the most common and most flexible format.

---

## Running a Launch File

Launch files are usually installed as part of a package:

```bash
ros2 launch <package_name> <launch_file>
```

Turtlesim ships with one that starts two simulators:

```bash
$ ros2 launch turtlesim multisim.launch.py
[INFO] [launch]: All log files can be found below /home/utku/.ros/log/...
[INFO] [launch]: Default logging verbosity is set to INFO
[INFO] [turtlesim_node-1]: process started with pid [12345]
[INFO] [turtlesim_node-2]: process started with pid [12346]
...
```

Two turtlesim windows open. Check the nodes and topics:

```bash
$ ros2 node list
/turtlesim1/turtlesim
/turtlesim2/turtlesim

$ ros2 topic list
/turtlesim1/turtle1/cmd_vel
...
/turtlesim2/turtle1/cmd_vel
...
```

One command started two nodes in separate namespaces. Press `Ctrl` + `C` in the launch terminal to stop everything.

You can also run a launch file directly by its path, which is handy while experimenting:

```bash
$ ros2 launch ./my_launch_file.launch.py
```

---

## Writing Your First Launch File

Create a file called `turtlesim_mimic.launch.py`:

```python
from launch import LaunchDescription
from launch_ros.actions import Node


def generate_launch_description():
    return LaunchDescription([
        Node(
            package='turtlesim',
            executable='turtlesim_node',
            namespace='turtlesim1',
            name='sim',
        ),
        Node(
            package='turtlesim',
            executable='turtlesim_node',
            namespace='turtlesim2',
            name='sim',
        ),
        Node(
            package='turtlesim',
            executable='mimic',
            name='mimic',
            remappings=[
                ('/input/pose', '/turtlesim1/turtle1/pose'),
                ('/output/cmd_vel', '/turtlesim2/turtle1/cmd_vel'),
            ],
        ),
    ])
```

How it works:

- `ros2 launch` looks for a function named **`generate_launch_description()`** and calls it.
- The function returns a **`LaunchDescription`**: a list of things to launch.
- Each **`Node(...)`** is the equivalent of one `ros2 run` command:

| `Node` argument | Command-line equivalent |
|-----------------|-------------------------|
| `package`, `executable` | `ros2 run <package> <executable>` |
| `name='sim'` | `--ros-args -r __node:=sim` |
| `namespace='turtlesim1'` | `--ros-args -r __ns:=/turtlesim1` |
| `remappings=[(a, b)]` | `--ros-args -r a:=b` |
| `parameters=[...]` | `--ros-args -p ...` / `--params-file ...` |

The `mimic` node subscribes to `/input/pose` and publishes to `/output/cmd_vel`, making a turtle copy whatever pose it receives. The **remappings** connect its generic topic names to the two turtlesims, so turtle 2 copies turtle 1.

Run it:

```bash
$ ros2 launch ./turtlesim_mimic.launch.py
```

Then drive turtle 1 in another terminal:

```bash
$ ros2 topic pub --rate 1 /turtlesim1/turtle1/cmd_vel geometry_msgs/msg/Twist "{linear: {x: 2.0}, angular: {z: -1.8}}"
```

Both turtles move together. Run `rqt_graph` to see how the mimic node connects them.

> 💡 **Remapping** changes a topic, service, or action name a node uses, without touching its code. It is how you connect generic, reusable nodes to your specific system.

---

## Setting Parameters

Pass parameters as a list of dictionaries, YAML file paths, or both:

```python
Node(
    package='turtlesim',
    executable='turtlesim_node',
    name='sim',
    parameters=[
        {'background_r': 0, 'background_g': 128, 'background_b': 0},
        # '/path/to/turtlesim.yaml',
    ],
),
```

In a real package you will keep YAML files in a `config/` folder and locate them at launch time. That is covered once you create your own packages in [Module 03](../03-workspaces-and-packages/README.md).

---

## Launch Arguments

Hard-coded values in a launch file have the same problem as hard-coded values in code. **Launch arguments** make launch files configurable from the command line.

```python
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument
from launch.substitutions import LaunchConfiguration
from launch_ros.actions import Node


def generate_launch_description():
    background_r = LaunchConfiguration('background_r')

    return LaunchDescription([
        DeclareLaunchArgument(
            'background_r',
            default_value='69',
            description='Red channel of the background color',
        ),
        Node(
            package='turtlesim',
            executable='turtlesim_node',
            parameters=[{'background_r': background_r}],
        ),
    ])
```

- **`DeclareLaunchArgument`** declares an argument with a default value and a description.
- **`LaunchConfiguration('background_r')`** is a placeholder (a *substitution*) that is replaced with the argument's value when the launch file runs.

Pass arguments with `name:=value`:

```bash
$ ros2 launch ./turtlesim_args.launch.py background_r:=255
```

List the arguments a launch file accepts:

```bash
$ ros2 launch ./turtlesim_args.launch.py --show-args
Arguments (pass arguments as '<name>:=<value>'):

    'background_r':
        Red channel of the background color
        (default: '69')
```

---

## Including Other Launch Files

Large systems are built by composing launch files. `IncludeLaunchDescription` runs another launch file as part of yours, and can pass arguments to it:

```python
import os

from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch.actions import IncludeLaunchDescription
from launch.launch_description_sources import PythonLaunchDescriptionSource


def generate_launch_description():
    turtlesim_launch = os.path.join(
        get_package_share_directory('turtlesim'), 'launch', 'multisim.launch.py')

    return LaunchDescription([
        IncludeLaunchDescription(PythonLaunchDescriptionSource(turtlesim_launch)),
    ])
```

`get_package_share_directory('turtlesim')` finds where a package's files are installed, so the launch file works on any computer regardless of where ROS 2 is installed. You will use this pattern constantly, for example to include the Gazebo or Nav2 launch files.

---

## Useful Things to Know

- Name launch files `<something>.launch.py`. The suffix is a convention that tools rely on.
- Node output goes to the launch terminal, prefixed with the process name (`[turtlesim_node-1]`).
- Logs of every launch are saved under `~/.ros/log/`.
- If one node crashes, the others keep running. Watch the output for `process has died` messages.

---

## Try It Yourself

1. Write a launch file that starts one turtlesim and one `turtle_teleop_key`. What goes wrong with the teleop node, and why? (Hint: it needs keyboard input from a terminal. Search for the `prefix='xterm -e'` option of `Node`.)
2. Extend the mimic launch file with a third turtlesim that also copies turtle 1.
3. Add launch arguments for all three background colors to the launch-arguments example and check them with `--show-args`.
4. Write a launch file that includes `turtlesim`'s `multisim.launch.py` and also starts a `mimic` node connecting the two simulators.
5. Rewrite the [Parameters](parameters.md) exercise so that the colors come from a YAML file passed to the node in your launch file.

---

⬅️ [Previous: Parameters](parameters.md) | [⬆️ ROS 2 Core Concepts](README.md)
