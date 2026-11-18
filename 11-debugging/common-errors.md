# Common Errors

Most ROS 2 problems fall into a handful of categories, and the same few causes account for most of them. This lesson is a reference: find your symptom or error message, read the likely cause, and apply the fix. Where a topic was covered in depth in an earlier module, the table links to it.

## Topics Covered

- Environment and sourcing errors
- Build errors
- Run-time errors
- Communication problems: no data on a topic
- Transform (TF) and time errors
- Launch and parameter errors
- Simulation and navigation problems

> 💡 Use your browser's search (`Ctrl` + `F`) with a few words from your error message.

---

## Environment and Sourcing

The single most common source of problems: a terminal that doesn't have the right environment.

| Symptom / message | Cause | Fix |
|-------------------|-------|-----|
| `ros2: command not found` | ROS 2 is not sourced in this terminal | `source /opt/ros/jazzy/setup.bash`, and add it to `~/.bashrc` ([Linux Basics](../00-prerequisites/linux-basics.md#environment-variables-and-bashrc)) |
| `Package 'my_pkg' not found` | Your workspace is not sourced, or not rebuilt after creating the package | `source ~/ros2_ws/install/setup.bash` in a new terminal after `colcon build` |
| Old code runs although you changed it | Another workspace with the same package is sourced, or you didn't rebuild | `ros2 pkg prefix my_pkg` shows which copy is used |
| Strange errors mentioning two distributions | Setup files of different distributions (e.g. Humble and Jazzy) sourced in the same shell | Clean `~/.bashrc`, open a fresh terminal, source one distribution only |
| `Tab` completion doesn't work for ROS commands | Not sourced, or `python3-argcomplete` missing | Source, or `sudo apt install python3-argcomplete` |

Check what is sourced:

```bash
$ printenv | grep -E 'ROS|AMENT_PREFIX_PATH'
ROS_DISTRO=jazzy
ROS_DOMAIN_ID=0
AMENT_PREFIX_PATH=/home/utku/ros2_ws/install/my_first_pkg:...:/opt/ros/jazzy
```

---

## Build Errors

Always scroll up to the **first** error; later ones are often follow-up errors.

| Symptom / message | Cause | Fix |
|-------------------|-------|-----|
| `colcon: command not found` | colcon not installed | `sudo apt install python3-colcon-common-extensions` |
| `Could not find a package configuration file provided by "xyz"` | Dependency not installed or not declared | Add to `package.xml`, run `rosdep install --from-paths src --ignore-src -y` ([package.xml](../03-workspaces-and-packages/package-xml.md#resolving-dependencies-with-rosdep)) |
| `fatal error: xyz/msg/abc.hpp: No such file or directory` | Missing `find_package` / `ament_target_dependencies`, or wrong header name (`lower_snake_case.hpp`) | [C++ Publisher](../05-cpp/publisher.md#building-the-node-cmakeliststxt) |
| `undefined reference to ...` | The library is found but not linked to this executable | Add the package to `ament_target_dependencies` for that target |
| Build freezes, computer becomes unresponsive | Parallel C++ compilation uses all RAM | `colcon build --parallel-workers 1`, `MAKEFLAGS="-j2"` ([colcon](../03-workspaces-and-packages/colcon.md#controlling-build-output)) |
| Errors after renaming/deleting a package or switching `--symlink-install` | Stale files in `build/` and `install/` | `rm -rf build install log`, rebuild |
| Build error in an interface package mentioning `rosidl` | Missing `<member_of_group>`, invalid field names | [Custom Interfaces](../04-python/custom-messages.md#common-mistakes) |

---

## Run-Time Errors

| Symptom / message | Cause | Fix |
|-------------------|-------|-----|
| `No executable found` | Entry point missing in `setup.py` / executable missing in `install(TARGETS)`, or no rebuild | [setup.py](../03-workspaces-and-packages/setup-py.md#common-errors) |
| `ModuleNotFoundError: No module named 'xyz'` | Dependency not installed, not declared, or workspace not sourced | Install with `rosdep`; for your own interfaces, rebuild and re-source |
| `file 'x.launch.py' was not found in the share directory` | Launch file not installed (`data_files` / `install(DIRECTORY)`) | [setup.py](../03-workspaces-and-packages/setup-py.md#data_files-installing-non-python-files) |
| The node exits immediately without a message | `main()` doesn't call `spin`, or an exception before logging was set up | Run the node alone with `ros2 run` and read the full output |
| `Segmentation fault (core dumped)` (C++) | Using an empty `SharedPtr`, out-of-range index, dangling reference | Run under `gdb` ([Debugging Workflow](debugging-workflow.md#c-gdb)) |
| The node freezes | Waiting for a service or action result inside a callback (deadlock) | [Avoiding Deadlocks](../04-python/services.md#avoiding-deadlocks) |
| A callback is slow, messages pile up or are dropped | Long computation or `sleep` in a callback | Keep callbacks short; move work to a timer or separate thread |

---

## No Data on a Topic

The topic exists, but the subscriber receives nothing, and nobody prints an error. Go through these causes in order:

| # | Cause | How to check |
|---|-------|--------------|
| 1 | The publisher isn't publishing | `ros2 topic hz /topic` shows no rate |
| 2 | Topic **names** differ (namespace, relative vs absolute name, typo) | `ros2 node info` on both nodes; `ros2 topic list` |
| 3 | Message **types** differ | `ros2 topic info /topic` lists one type per endpoint; `ros2 topic list -t` |
| 4 | **QoS** incompatibility (reliable vs best effort, durability) | `ros2 topic info /topic --verbose`; ROS 2 logs `New publisher discovered on topic '...', offering incompatible QoS` |
| 5 | Different `ROS_DOMAIN_ID` | `printenv ROS_DOMAIN_ID` in every terminal |
| 6 | Discovery limited to localhost | `ROS_AUTOMATIC_DISCOVERY_RANGE` (Jazzy) or `ROS_LOCALHOST_ONLY` (Humble) set differently |
| 7 | Nodes on different computers can't discover each other | Firewall, multicast blocked, different networks; see [ros2 doctor](ros2-doctor.md#checking-multicast) |
| 8 | Subscriber's callback never runs | The node doesn't spin, or another callback blocks the executor |

---

## TF and Time Errors

| Message contains | Cause | Fix |
|------------------|-------|-----|
| `frame does not exist` | No one publishes the frame (yet), or a typo | `view_frames`; start the missing broadcaster ([TF2 Tools](../06-tf2/tf2-tools.md#common-tf-errors)) |
| `not part of the same tree` | A transform linking two subtrees is missing (e.g. `map → odom` without localization) | Start localization/SLAM, or publish the missing static transform |
| `extrapolation into the future` / `into the past` | Asking for a time without data, or mixed clocks | Use the latest transform; set `use_sim_time` consistently |
| RViz2: `Message removed because it is too old` | Timestamps don't match RViz2's clock | `use_sim_time:=true` on RViz2 in simulation; check system clocks on multiple computers |
| Robot jumps between two positions | Two nodes publish the same transform | `ros2 topic info /tf --verbose`; remove one publisher |
| Robot appears rotated or mirrored | Degrees instead of radians, wrong axis, or wrong quaternion | `tf2_echo` shows RPY in degrees; check REP 103 conventions |

> ⚠️ In simulation or when replaying a bag, **every** node must use `use_sim_time:=true`. A single node on wall-clock time causes TF errors that look unrelated to time.

---

## Launch and Parameter Errors

| Symptom / message | Cause | Fix |
|-------------------|-------|-----|
| A launch argument seems to be ignored | Misspelled; unknown arguments are silently accepted | `ros2 launch pkg file --show-args` |
| `process has died [pid ..., exit code ...]` | That node crashed | Scroll up for its error; run it alone with `ros2 run` |
| A parameter file has no effect | Wrong node name key, or `ros__parameters` misspelled | `ros2 param get` on the running node; compare names ([Parameters](../01-ros2-basics/parameters.md#saving-and-loading-parameter-files)) |
| `Setting parameter failed` / wrong type | `1` vs `1.0`, read-only parameter | `ros2 param describe` |
| `Unable to parse the value of parameter robot_description as yaml` | URDF from `Command(...)` not wrapped in `ParameterValue(..., value_type=str)` | [robot_state_publisher](../07-robot-modeling/robot-state-publisher.md#how-it-works) |
| Nodes left running after `Ctrl` + `C` | Terminal closed or interrupted repeatedly | `ros2 node list`; kill leftovers (`pkill -f <name>`) |

---

## Simulation, Navigation, and Manipulation

| Symptom | Cause | See |
|---------|-------|-----|
| Gazebo black, crashing, or very slow | Graphics driver / VM rendering | [Gazebo troubleshooting](../08-simulation/gazebo.md#troubleshooting) |
| Robot doesn't move in Gazebo | `/cmd_vel` not bridged, or plugin topic differs | [Spawning Robots](../08-simulation/spawning-robots.md#common-problems) |
| Sensor data in Gazebo but not in ROS 2 | Topic missing from the bridge configuration | [Sensors](../08-simulation/sensors.md#common-problems) |
| Nav2 never becomes active | A lifecycle node failed to configure | `ros2 lifecycle get`, first error in the output ([Nav2](../09-navigation/nav2.md#common-problems)) |
| AMCL: `Please set the initial pose` | No initial pose | [Localization](../09-navigation/localization.md#setting-the-initial-pose) |
| MoveIt: `Planning failed` | Unreachable or colliding goal | [MoveIt 2](../10-manipulation/moveit.md#common-problems) |

---

## Try It Yourself

Practice recognizing errors by causing them deliberately, then fixing them:

1. Open a new terminal without sourcing your workspace and run `ros2 run my_first_pkg talker`.
2. Run the talker with `--ros-args -r chatter:=chat` and the listener unchanged. Find the cause with the CLI tools.
3. Make the talker publish with best-effort reliability while the listener keeps the default (reliable). Does the listener receive anything? Read the QoS warning, then swap the settings and try again.
4. Remove an entry point from `setup.py`, rebuild, and run the node.
5. Start `turtle_follower` from [TF2 Listeners](../06-tf2/tf2-listener.md) without the broadcaster for `turtle2`, and read the TF error.

---

[⬆️ Debugging & Troubleshooting](README.md) | [Next: ros2 doctor](ros2-doctor.md) ➡️
