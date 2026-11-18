# rosbag2

Many bugs only show up while the robot moves: a scan that looks wrong for half a second, a wheel that slips, a TF jump. **rosbag2** records topics to a file (a *bag*) and replays them later, exactly as they were published. With a recording, you can investigate a problem as often as you want, without the robot or the simulator, and share it with others.

## Topics Covered

- Recording topics
- Inspecting a bag with `ros2 bag info`
- Playing a bag back
- Replaying with simulation time
- Debugging a node offline with recorded data
- Reading a bag in Python
- Visualization tools

## In Simple Terms

rosbag2 is a **dashcam for robots**: it records everything that happens on the topics you choose, so you can replay the moment of the problem as often as you need, without the robot.

**Example:** the robot turned the wrong way at one corner. You replay the recorded laser data and watch your node make the same decision again, this time with extra log messages.

---

## Recording

Start something that publishes data, for example the simulation from [Module 08](../08-simulation/simulation-project.md):

```bash
$ ros2 launch my_robot_gazebo sim.launch.py
$ ros2 run my_first_pkg scan_avoider --ros-args -p use_sim_time:=true
```

Record a few topics:

```bash
$ ros2 bag record -o explore_run /scan /odom /tf /tf_static /cmd_vel
[INFO] [rosbag2_recorder]: Press SPACE for pausing/resuming
[INFO] [rosbag2_recorder]: Listening for topics...
[INFO] [rosbag2_recorder]: Subscribed to topic '/scan'
...
```

Stop with `Ctrl` + `C`. The bag is a folder:

```text
explore_run/
├── explore_run_0.mcap     ← the recorded messages
└── metadata.yaml          ← topics, types, counts, duration
```

| Option | Effect |
|--------|--------|
| `-o <name>` | Name of the output folder (default: `rosbag2_<date>_<time>`) |
| `-a`, `--all` | Record all topics |
| `-e`, `--regex <expr>` | Record topics matching a regular expression, e.g. `-e "/camera/.*"` |
| `-d`, `--max-bag-duration <s>` | Split into a new file every N seconds |
| `-b`, `--max-bag-size <bytes>` | Split into a new file every N bytes |
| `-s`, `--storage <id>` | Storage format: `mcap` (default in Jazzy) or `sqlite3` |

> ⚠️ `-a` on a robot with cameras or 3D LiDARs produces gigabytes per minute. Record only the topics you need. `/tf_static` must almost always be included: without it, the replayed data can't be placed in the TF tree.

---

## Inspecting a Bag

```bash
$ ros2 bag info explore_run
Files:             explore_run_0.mcap
Bag size:          8.4 MiB
Storage id:        mcap
Duration:          61.2s
Start:             Nov 18 2026 10:15:02.115 (1795508102.115)
End:               Nov 18 2026 10:16:03.315 (1795508163.315)
Messages:          6134
Topic information: Topic: /scan | Type: sensor_msgs/msg/LaserScan | Count: 612 | Serialization Format: cdr
                   Topic: /odom | Type: nav_msgs/msg/Odometry | Count: 1836 | Serialization Format: cdr
                   Topic: /tf | Type: tf2_msgs/msg/TFMessage | Count: 1836 | Serialization Format: cdr
                   Topic: /tf_static | Type: tf2_msgs/msg/TFMessage | Count: 1 | Serialization Format: cdr
                   Topic: /cmd_vel | Type: geometry_msgs/msg/Twist | Count: 612 | Serialization Format: cdr
```

The message counts are a quick sanity check: 612 scans in 61 s is 10 Hz, as configured. A count of 0 means the topic was never published during the recording.

---

## Playing Back

Stop the simulation and all nodes, then:

```bash
$ ros2 bag play explore_run
```

The recorded messages are published again with their original timing. To any node, it looks as if the robot were running. Try it with RViz2 showing `/scan` and TF.

| Option | Effect |
|--------|--------|
| `-r`, `--rate <factor>` | Playback speed: `0.5` = half speed, `2.0` = double |
| `-l`, `--loop` | Restart at the end |
| `--topics <t1> <t2>` | Play only these topics |
| `--remap <from>:=<to>` | Rename topics on playback |
| `-p`, `--start-paused` | Start paused |
| `--clock [Hz]` | Publish `/clock` from the bag's timestamps (see below) |

While playing, the terminal accepts keyboard controls: **Space** pauses and resumes, **→** publishes the next message (when paused), **↑** / **↓** change the playback rate. Stepping message by message while paused is very useful to examine a single moment.

---

## Playback and Time

Recorded messages carry their **original timestamps**. If nodes use the wall clock while replaying, those timestamps are "in the past" and TF lookups fail with extrapolation errors. The solution is the same as in simulation: let the bag provide the clock.

```bash
$ ros2 bag play explore_run --clock
```

```bash
$ rviz2 --ros-args -p use_sim_time:=true
```

`--clock` publishes `/clock` following the bag's time, and every node that processes the replayed data must run with `use_sim_time:=true` (see [Simulation Time](../08-simulation/gazebo.md#simulation-time)).

---

## Debugging a Node Offline

The real power of bags: run your node on recorded input as often as you like. Suppose `scan_avoider` turned the wrong way at one corner. Replay **only the input** and let the node react again:

```bash
$ ros2 bag play explore_run --clock --topics /scan /tf /tf_static
```

```bash
$ ros2 run my_first_pkg scan_avoider --ros-args -p use_sim_time:=true
```

```bash
$ ros2 topic echo /cmd_vel
```

Now you can add log messages, change parameters, or fix the code, and replay the **same** situation each time, which is impossible with a live robot. Note that `/cmd_vel` is **not** replayed: your node produces it, and replaying the recorded commands at the same time would mix old and new output.

> 💡 The node's decisions no longer influence the robot's motion (the scans are fixed), so this tests how the node **reacts** to data, not the closed loop. For closed-loop testing, use simulation.

---

## Reading a Bag in Python

For analysis (plots, statistics, checking for problems), read the bag directly in a script with `rosbag2_py`. This example prints the robot's odometry positions:

```python
import rosbag2_py
from rclpy.serialization import deserialize_message
from rosidl_runtime_py.utilities import get_message

reader = rosbag2_py.SequentialReader()
reader.open(
    rosbag2_py.StorageOptions(uri='explore_run', storage_id='mcap'),
    rosbag2_py.ConverterOptions(input_serialization_format='cdr',
                                output_serialization_format='cdr'),
)

topic_types = {t.name: t.type for t in reader.get_all_topics_and_types()}

while reader.has_next():
    topic, data, timestamp = reader.read_next()
    if topic == '/odom':
        msg = deserialize_message(data, get_message(topic_types[topic]))
        p = msg.pose.pose.position
        print(f'{timestamp / 1e9:.2f} s: x={p.x:.2f}, y={p.y:.2f}')
```

```bash
$ python3 read_odom.py
1795508102.15 s: x=0.00, y=0.00
1795508102.18 s: x=0.01, y=0.00
...
```

From here it's a small step to plotting the trajectory with `matplotlib`, or to finding the moment the robot came closest to a wall.

---

## Visualization Tools

| Tool | Use | Install |
|------|-----|---------|
| **PlotJuggler** | Plot any numeric field over time, live or from a bag | `sudo apt install ros-jazzy-plotjuggler-ros` |
| **Foxglove** | Web-based visualization of bags (3D, plots, images), good for sharing | [foxglove.dev](https://foxglove.dev/) |
| **rqt_bag** | Timeline view of a bag inside rqt | `sudo apt install ros-jazzy-rqt-bag` |
| **RViz2** | 3D view of replayed data | already installed |

PlotJuggler is particularly helpful for debugging controllers: plot the commanded velocity (`/cmd_vel/linear/x`) against the measured one (`/odom/twist/twist/linear/x`) and see delays, overshoot, and saturation at a glance.

---

## Try It Yourself

1. Record 30 seconds of turtlesim driven with the teleop node (`/turtle1/cmd_vel`, `/turtle1/pose`). Restart turtlesim and replay **only** `/turtle1/cmd_vel`. Does the turtle draw the same path? Why might it differ?
2. Replay the same bag at `--rate 2.0` and at `--rate 0.5`.
3. Record the simulation with `-d 10` and look at the files created.
4. Use the Python reader to compute the total distance the robot traveled from `/odom`.
5. Install PlotJuggler, load your bag, and plot `/cmd_vel/angular/z` next to `/odom/twist/twist/angular/z`.

---

⬅️ [Previous: ros2 doctor](ros2-doctor.md) | [⬆️ Debugging & Troubleshooting](README.md) | [Next: Debugging Workflow](debugging-workflow.md) ➡️
