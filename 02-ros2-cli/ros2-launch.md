# ros2 launch

`ros2 launch` starts a whole system described in a launch file: many nodes, each with its own names, parameters, and remappings. This lesson covers how to run launch files, pass arguments, find out what a launch file does, and debug it when something goes wrong.

> 📖 Concepts and writing launch files: [Launch Files](../01-ros2-basics/launch-files.md)

## Topics Covered

- Running launch files from a package or a path
- Passing and discovering launch arguments
- Finding the launch files a package provides
- Inspecting and debugging launch files
- Reading launch output and logs
- Stopping a launch

---

## Basic Usage

```bash
ros2 launch [options] <package> <launch_file> [argument:=value ...]
ros2 launch [options] <path/to/launch_file> [argument:=value ...]
```

```bash
# From an installed package
$ ros2 launch turtlesim multisim.launch.py

# From a file on disk
$ ros2 launch ./my_system.launch.py
```

`Tab` completion works for the package, the launch file name, and even the argument names.

---

## Launch Arguments

Arguments are passed as `name:=value`, after the launch file:

```bash
$ ros2 launch my_robot bringup.launch.py use_sim_time:=true robot_name:=rover
```

To find out which arguments a launch file accepts, use `--show-args` (`-s`). It prints them without starting anything:

```bash
$ ros2 launch ./turtlesim_args.launch.py --show-args
Arguments (pass arguments as '<name>:=<value>'):

    'background_r':
        Red channel of the background color
        (default: '69')
```

> 💡 Get into the habit of running `--show-args` on every unfamiliar launch file, especially large ones like Gazebo or Nav2. It is usually faster than reading the source.

---

## Finding a Package's Launch Files

Launch files are installed in the package's `share` directory:

```bash
$ ls $(ros2 pkg prefix --share turtlesim)/launch
multisim.launch.py
```

To read one:

```bash
$ cat $(ros2 pkg prefix --share turtlesim)/launch/multisim.launch.py
```

Reading the launch files of existing packages is one of the best ways to learn how real systems are put together.

---

## Options

| Option | Effect |
|--------|--------|
| `-s`, `--show-args` | Show the launch arguments, then exit |
| `-p`, `--print`, `--print-description` | Print what the launch file would do, then exit |
| `-d`, `--debug` | Print detailed debug output from the launch system |
| `--launch-prefix '<command>'` | Put a command in front of every executable (like `ros2 run --prefix`) |
| `-n`, `--noninteractive` | Run without a terminal attached to standard input |

```bash
# See the structure of a launch file without running it
$ ros2 launch turtlesim multisim.launch.py --print

# Open every node in its own terminal window
$ ros2 launch turtlesim multisim.launch.py --launch-prefix 'xterm -e'
```

---

## Reading Launch Output

All nodes print to the launch terminal. Each line is prefixed with the process name and an index:

```text
[INFO] [launch]: All log files can be found below /home/utku/.ros/log/2026-10-21-10-15-00-123456-laptop-12345
[INFO] [launch]: Default logging verbosity is set to INFO
[INFO] [turtlesim_node-1]: process started with pid [12346]
[INFO] [turtlesim_node-2]: process started with pid [12347]
[turtlesim_node-1] [INFO] [1729500900.123456789] [turtlesim1.turtlesim]: Starting turtlesim with node name /turtlesim1/turtlesim
...
```

- Lines starting with `[INFO] [launch]` come from the launch system itself.
- Lines starting with `[turtlesim_node-1]` come from a node process.
- `process started with pid` confirms that a node started; **`process has died`** means it crashed or exited. Scroll up from that line to find the node's error message.

All output is also saved under `~/.ros/log/`, in the folder printed on the first line. This is useful when the terminal output scrolled away or when a launch ran on a robot without a screen.

---

## Stopping a Launch

Press **`Ctrl` + `C`** once. The launch system asks every process to shut down cleanly and waits for them. If a node does not stop in time, launch escalates and terminates it.

> ⚠️ Avoid pressing `Ctrl` + `C` repeatedly or closing the terminal window. That can leave node processes running in the background. If you suspect leftovers, check with `ros2 node list`.

---

## Troubleshooting

| Symptom | Likely cause |
|---------|--------------|
| `Package '<name>' not found` | Package not installed, or your workspace is not sourced |
| `file '<name>' was not found in the share directory of package` | Typo in the file name, or the launch file was not installed (check `setup.py` / `CMakeLists.txt` in [Module 03](../03-workspaces-and-packages/README.md)) |
| An argument seems to be ignored | The argument name is misspelled. Unknown arguments are silently accepted, so verify with `--show-args` |
| `process has died [pid ..., exit code ...]` | That node crashed. Find its error message above in the output, then run it alone with `ros2 run` to debug |
| A node that reads the keyboard does not react | It needs its own terminal: use `--launch-prefix 'xterm -e'` or `prefix='xterm -e'` on that node |

---

## Try It Yourself

1. Run `turtlesim multisim.launch.py` with `--print` and identify the two nodes in the output.
2. Find the launch files of the `demo_nodes_cpp` package with `ros2 pkg prefix --share` and run one of them.
3. Run your `turtlesim_args.launch.py` from [Launch Files](../01-ros2-basics/launch-files.md) with a misspelled argument (`backround_r:=0`). What happens, and how would `--show-args` have helped?
4. Start `multisim.launch.py`, stop it with `Ctrl` + `C`, and open the log folder it printed. Which files are inside?
5. Kill one of the turtlesim windows while the launch is running. What does the launch terminal report?

---

⬅️ [Previous: ros2 run](ros2-run.md) | [⬆️ CLI Tools](README.md)
