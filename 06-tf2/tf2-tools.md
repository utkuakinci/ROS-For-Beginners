# TF2 Tools

When a robot shows up in the wrong place in RViz2, or a node keeps logging `Could not transform ...`, the cause is almost always in the transform tree. This lesson covers the tools that show you what the tree really looks like, and a checklist for the most common TF errors.

## Topics Covered

- Drawing the tree with `view_frames`
- Printing a transform with `tf2_echo`
- Checking rates and delays with `tf2_monitor`
- Reading `/tf` and `/tf_static` directly
- Visualizing frames in RViz2
- Common TF errors and how to fix them

All examples assume the follow demo from [TF2 Listeners](tf2-listener.md#a-launch-file-for-the-demo) is running:

```bash
$ ros2 launch my_first_pkg tf_follow.launch.py
```

---

## `view_frames`: Draw the Tree

```bash
$ ros2 run tf2_tools view_frames
[INFO] [view_frames]: Listening to tf data for 5.0 seconds...
[INFO] [view_frames]: Generating graph in frames.pdf file...
```

This listens for 5 seconds and writes `frames_<date>.pdf` (and a `.gv` source file) to the current folder. Open it with any PDF viewer:

```text
            world
           ╱      ╲
      turtle1     turtle2
     ╱       ╲
turtle1_camera  turtle1_antenna
```

Each arrow in the PDF is labeled with:

| Label | Meaning |
|-------|---------|
| `Broadcaster` | The node publishing this transform |
| `Average rate` | How often it is published (`10000` for static transforms) |
| `Most recent transform` | Timestamp of the last received transform |
| `Buffer length` | How much history is stored |

`view_frames` is the first tool to reach for when something is wrong: missing frames, a frame attached to the wrong parent, or two separate trees that should be connected are immediately visible.

---

## `tf2_echo`: Print One Transform

```bash
$ ros2 run tf2_ros tf2_echo world turtle1
[INFO] [tf2_echo]: Waiting for transform world ->  turtle1: Invalid frame ID "world" passed to canTransform argument target_frame - frame does not exist
At time 1731241000.502401234
- Translation: [5.544, 5.544, 0.000]
- Rotation: in Quaternion [0.000, 0.000, 0.000, 1.000]
- Rotation: in RPY (radian) [0.000, -0.000, 0.000]
- Rotation: in RPY (degree) [0.000, -0.000, 0.000]
- Matrix:
  1.000  0.000  0.000  5.544
  0.000  1.000  0.000  5.544
  0.000  0.000  1.000  0.000
  0.000  0.000  0.000  1.000
```

The syntax is `tf2_echo <target_frame> <source_frame>`: the source frame's pose expressed in the target frame, the same order as `lookup_transform`. The first warning is normal: right after starting, the tool hasn't received any data yet.

Useful details:

- The rotation is shown as a quaternion **and** as roll/pitch/yaw in radians and degrees, which makes it easy to check orientations.
- Swap the arguments to get the inverse transform.
- Add a rate in Hz as a third argument to print less often: `tf2_echo world turtle1 1.0`.

---

## `tf2_monitor`: Rates and Delays

```bash
$ ros2 run tf2_ros tf2_monitor
```

`tf2_monitor` lists every frame with its publishing rate and the delay between the transform's timestamp and the moment it was received. Pass two frames to monitor a single chain:

```bash
$ ros2 run tf2_ros tf2_monitor world turtle2
```

A large or growing delay points to an overloaded computer, a slow network, or a node stamping transforms with the wrong time.

---

## Reading `/tf` and `/tf_static` Directly

Transforms are ordinary topics, so the tools from [ros2 topic](../02-ros2-cli/ros2-topic.md) work on them:

```bash
$ ros2 topic echo /tf --once
transforms:
- header:
    stamp:
      sec: 1731241000
      nanosec: 502401234
    frame_id: world
  child_frame_id: turtle1
  transform:
    translation:
      x: 5.544444561004639
      ...

$ ros2 topic hz /tf
$ ros2 topic info /tf --verbose                   # which nodes publish transforms
```

`/tf_static` uses transient local durability, so request it explicitly to see transforms published before you started listening:

```bash
$ ros2 topic echo /tf_static --qos-durability transient_local
```

---

## Visualizing Frames in RViz2

RViz2 draws frames as small coordinate axes, which is the most intuitive way to check positions and orientations:

```bash
$ rviz2
```

1. Set **Global Options → Fixed Frame** to `world`.
2. Click **Add** → **TF** → **OK**.
3. In the TF display, enable **Show Names**.

Each frame appears as three arrows: **red = x, green = y, blue = z** (easy to remember: RGB = XYZ). Drive `turtle1` and watch its frames move, with `turtle2`'s frame chasing it. You will use RViz2 in depth in [Module 07](../07-robot-modeling/rviz.md).

---

## Common TF Errors

| Error message contains | Likely cause | How to check / fix |
|------------------------|--------------|--------------------|
| `frame does not exist` | No one publishes that frame (yet), or a typo in the name | `view_frames`; check broadcaster is running and the name is identical |
| `not part of the same tree` / `Could not find a connection` | Two separate trees; a link between them is missing | `view_frames` shows two roots. Publish the missing transform (e.g. `map → odom`) |
| `extrapolation into the future` | Asking for a time newer than the latest transform | Use `rclpy.time.Time()` for latest, or wait briefly with a timeout |
| `extrapolation into the past` | Asking for a time older than the buffer, or clocks don't match | Check timestamps; with simulation, check all nodes use the same `use_sim_time` |
| Frame jumps back and forth | Two nodes publish the same transform | `ros2 topic info /tf --verbose`, `view_frames` (broadcaster name changes) |
| Robot appears rotated or mirrored | Wrong angle units (degrees instead of radians) or axis convention | `tf2_echo` shows RPY in degrees for checking |
| `TF_REPEATED_DATA` warnings | A transform is published twice with the same timestamp | Publish only when data changes, with fresh timestamps |

### A debugging checklist

1. Run `view_frames`. Does the tree look like you expect?
2. Use `tf2_echo` on the transform in question. Are the values plausible (meters, radians, signs)?
3. Check rates and delays with `tf2_monitor`.
4. Look at the frames in RViz2. Do the axes point where they should?
5. In simulation or with recorded data, confirm every node has the same `use_sim_time` setting.

---

## Try It Yourself

1. Run `view_frames` with the follow demo and find the broadcaster and rate of every transform.
2. Stop `broadcaster2` (comment it out of the launch file or kill the node) and look at the follower's log and at `view_frames`. Which error appears?
3. Publish a static transform `world → turtle1` from the command line while the demo is running. What happens to `turtle1`'s frame in RViz2, and why?
4. Use `tf2_echo turtle1 turtle1_camera` and `tf2_echo turtle1_camera turtle1`. Explain the difference.
5. Open RViz2 with the TF display and set the Fixed Frame to `turtle1` instead of `world`. What changes, and why can this view be useful?

---

⬅️ [Previous: TF2 Listeners](tf2-listener.md) | [⬆️ Transform Library (TF2)](README.md)
