# Debugging Workflow

Knowing the tools is half the job. The other half is using them in a sensible order. Randomly changing parameters and restarting things can take hours; a systematic process usually finds the problem in minutes. This lesson presents that process, the logging and GUI tools that support it, and how to use a debugger on Python and C++ nodes. It ends with a worked example.

## Topics Covered

- A systematic debugging process
- Divide and conquer in a ROS 2 system
- Logging: levels, formats, and log files
- rqt tools
- Debugging Python nodes with `pdb`
- Debugging C++ nodes with `gdb`
- A worked example: "the robot doesn't move"
- Habits that prevent bugs

---

## The Process

```text
 1. Reproduce  ──▶  2. Observe  ──▶  3. Hypothesize  ──▶  4. Test  ──▶  5. Fix & verify
       ▲                                                     │
       └─────────────────── hypothesis wrong ────────────────┘
```

| Step | What to do |
|------|-----------|
| **1. Reproduce** | Find the shortest sequence of steps that triggers the problem. If it happens rarely, record a [bag](rosbag.md) |
| **2. Observe** | Collect facts with the tools, *before* changing anything: which nodes run, what data flows, what the logs say |
| **3. Hypothesize** | Form one concrete explanation: "the controller's subscriber uses a different QoS than the driver" |
| **4. Test** | Check that one hypothesis with one experiment. Change one thing at a time |
| **5. Fix & verify** | Fix the cause, then confirm the original problem is gone and nothing else broke |

The most common mistake is skipping step 2: guessing a fix, changing three things at once, and never learning what the actual problem was.

---

## Divide and Conquer

A ROS 2 system is a chain of nodes connected by topics, services, and transforms. Find **where** in the chain the data goes wrong by checking it at each link, from the source to the output:

```text
 sensor/driver ──▶ /scan ──▶ your node ──▶ /cmd_vel ──▶ bridge ──▶ robot
       ✓              ✓          ?
```

At each link, ask:

| Question | Tool |
|----------|------|
| Is the node running? | `ros2 node list`, `ros2 lifecycle get` |
| Is it connected to the right topics? | `ros2 node info`, `rqt_graph` |
| Does data arrive, at the expected rate? | `ros2 topic hz`, `ros2 topic echo --once` |
| Are the values plausible (units, frame, signs)? | `ros2 topic echo`, PlotJuggler, RViz2 |
| Are the transforms correct? | `tf2_echo`, `view_frames` |

**Replace components with the CLI** to isolate them. If the robot doesn't move, publish a velocity yourself with `ros2 topic pub`: if the robot moves now, the problem is upstream, in whatever should have produced the command. If it still doesn't move, the problem is downstream. Each test halves the search space.

---

## Logging

### Levels

Use log levels consistently in your own nodes:

| Level | Use for |
|-------|---------|
| `DEBUG` | Detailed internal information, normally hidden |
| `INFO` | Normal, important events: "started", "goal reached" |
| `WARN` | Something unexpected, but the node continues: "no scan received for 1 s" |
| `ERROR` | Something failed: "could not connect to the motor" |
| `FATAL` | The node cannot continue |

Show debug output for one node at startup:

```bash
$ ros2 run my_first_pkg scan_avoider --ros-args --log-level scan_avoider:=debug
```

### Output format

Make log lines more informative by setting the format, for example adding the source file and line:

```bash
$ export RCUTILS_CONSOLE_OUTPUT_FORMAT="[{severity}] [{name}]: {message} ({file_name}:{line_number})"
$ export RCUTILS_COLORIZED_OUTPUT=1
```

### Log files

Every node also writes its log to `~/.ros/log/`. Launch files print the exact folder on their first line. When a launch scrolled away, or a robot ran without a screen, look there:

```bash
$ ls -t ~/.ros/log | head -n 3
$ grep -r "ERROR" ~/.ros/log/latest/
```

---

## rqt Tools

**rqt** is a collection of small GUI tools. Start any of them directly, or all from the `rqt` window (**Plugins** menu).

| Tool | Command | Use |
|------|---------|-----|
| **rqt_graph** | `rqt_graph` | Nodes and topics as a live graph |
| **rqt_console** | `ros2 run rqt_console rqt_console` | All log messages of all nodes in one table: filter by node and level, sort, pause |
| **rqt_plot** | `ros2 run rqt_plot rqt_plot` | Plot numeric fields live, e.g. `/odom/twist/twist/linear/x` |
| **rqt_topic** | `ros2 run rqt_topic rqt_topic` | All topics with their rates and latest values |
| **rqt_reconfigure** | `ros2 run rqt_reconfigure rqt_reconfigure` | Change parameters with sliders and checkboxes |
| **rqt_tf_tree** | `ros2 run rqt_tf_tree rqt_tf_tree` | The TF tree, live (like `view_frames`, without a PDF) |

`rqt_console` is especially useful with many nodes: instead of hunting through several terminals, you see every warning and error in one place, with the node that produced it.

---

## Debugging Python Nodes: `pdb`

A **debugger** stops the program at a chosen line, so you can inspect variables and step through the code. For Python, insert `breakpoint()` where you want to stop:

```python
def control_loop(self):
    if self.scan is None:
        return
    front = self.sector_min(self.scan, -30.0, 30.0)
    breakpoint()  # execution stops here
    ...
```

Rebuild if needed, and run the node with `ros2 run` (not from a launch file, which has no interactive terminal):

```bash
$ ros2 run my_first_pkg scan_avoider --ros-args -p use_sim_time:=true
> /home/utku/ros2_ws/.../scan_avoider.py(42)control_loop()
-> ...
(Pdb) p front
0.5823
(Pdb) p len(self.scan.ranges)
360
(Pdb) n        # next line
(Pdb) c        # continue to the next breakpoint
```

| Command | Effect |
|---------|--------|
| `p <expr>` | Print a value |
| `n` | Execute the next line |
| `s` | Step into a function call |
| `c` | Continue until the next breakpoint |
| `l` | Show the code around the current line |
| `q` | Quit |

> 💡 While the program is stopped, the node doesn't spin, so timers and callbacks pause and other nodes may time out. Debuggers are best for logic errors; for timing problems, use logging.

Editors like VS Code offer graphical debugging for Python nodes, with breakpoints set by clicking next to a line.

---

## Debugging C++ Nodes

### C++: gdb

For crashes such as `Segmentation fault`, a debugger shows **where** the program died.

1. Build with debug information:

   ```bash
   $ colcon build --packages-select my_cpp_pkg --cmake-args -DCMAKE_BUILD_TYPE=Debug
   ```

2. Run the node inside `gdb` with the `--prefix` option of [ros2 run](../02-ros2-cli/ros2-run.md#running-under-another-tool---prefix):

   ```bash
   $ ros2 run --prefix 'gdb -ex run --args' my_cpp_pkg wall_bouncer
   ...
   Thread 1 "wall_bouncer" received signal SIGSEGV, Segmentation fault.
   0x000055555556a1b2 in WallBouncer::control_loop (this=0x...) at .../wall_bouncer.cpp:45
   45          const bool near_wall = pose_->x < margin || ...
   ```

3. Print the **backtrace**, the chain of function calls that led to the crash:

   ```text
   (gdb) bt
   #0  WallBouncer::control_loop (this=...) at .../wall_bouncer.cpp:45
   #1  ... rclcpp::GenericTimer<...>::execute_callback ...
   ...
   (gdb) p pose_
   $1 = std::shared_ptr<turtlesim::msg::Pose> (empty) = {get() = 0x0}
   ```

   Line 45 dereferences `pose_`, which is empty: the `if (!pose_)` check is missing. The cause is found in seconds.

| gdb command | Effect |
|-------------|--------|
| `run` | Start the program (done automatically by `-ex run`) |
| `bt` | Backtrace: where did it crash, and how did it get there |
| `p <expr>` | Print a variable |
| `frame <n>` | Switch to frame `n` of the backtrace |
| `break file.cpp:42` | Set a breakpoint |
| `quit` | Exit |

In a launch file, use the `prefix` argument of `Node`, with a separate terminal so `gdb` can be used interactively:

```python
Node(package='my_cpp_pkg', executable='wall_bouncer', prefix=['xterm -e gdb -ex run --args'])
```

---

## A Worked Example: "The Robot Doesn't Move"

You start the simulation and `scan_avoider`, but the robot just sits there. Follow the chain from the controller to the wheels.

**1. Is the node running and publishing?**

```bash
$ ros2 node list | grep scan_avoider
/scan_avoider
$ ros2 topic hz /cmd_vel
(no output)
```

Nothing is published. The problem is in or before `scan_avoider`.

**2. Does the node get its input?**

```bash
$ ros2 topic hz /scan
average rate: 10.000
$ ros2 node info /scan_avoider
  Subscribers:
    /scan: sensor_msgs/msg/LaserScan
  ...
```

Scans are published, and the node subscribes to the right topic and type. So data should arrive... unless the callbacks don't run.

**3. Hypothesis: the control loop never runs.** The timer depends on the node's clock. Check:

```bash
$ ros2 param get /scan_avoider use_sim_time
Boolean value is: True
$ ros2 topic hz /clock
(no output)
```

There it is: the node uses simulation time, but no `/clock` is published. Simulation time stands still, so the timer never fires, and `control_loop` never runs.

**4. Test:** check the bridge configuration. `/clock` is missing from `bridge.yaml` (or the bridge isn't running).

**5. Fix and verify:** add `/clock` to the bridge, restart, and `ros2 topic hz /cmd_vel` shows 10 Hz. The robot drives.

Notice that no step involved guessing: each command answered one question and narrowed down the search.

---

## Habits That Prevent Bugs

- **Build and test in small steps.** Add one node or feature, check it with the CLI, then continue.
- **Handle missing data.** Every subscriber should know what to do before its first message arrives.
- **Log meaningful events** at `INFO`, details at `DEBUG`, and use throttling in fast loops.
- **Declare all dependencies** in `package.xml`, so the package works on other machines.
- **Keep configuration in files** (YAML, launch arguments), not hard-coded in the code.
- **Record bags** of interesting runs; they are cheap and invaluable later.
- **Stop the robot** in your node's shutdown code (publish a zero velocity).
- **Use version control.** When something breaks, `git diff` shows what you changed since it last worked.

---

## Try It Yourself

1. Introduce the `/clock` bug from the worked example on purpose (remove `/clock` from `bridge.yaml`) and find it again using only the CLI.
2. Run three nodes at once and use `rqt_console` to show only warnings and errors.
3. Put a `breakpoint()` in `wall_bouncer.py` and inspect `self.pose` when the turtle is near a wall.
4. Remove the `if (!pose_)` check from the C++ `wall_bouncer`, start it before turtlesim, and find the crash with `gdb`.
5. Plot the turtle's `linear_velocity` from `/turtle1/pose` with `rqt_plot` while you drive.

---

⬅️ [Previous: rosbag2](rosbag.md) | [⬆️ Debugging & Troubleshooting](README.md)
