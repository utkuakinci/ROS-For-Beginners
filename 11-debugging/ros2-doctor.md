# ros2 doctor

`ros2 doctor` checks your ROS 2 setup for common problems: the platform and distribution, network configuration, the middleware, and the running system. It is the first thing to run when "nothing works" and you don't know why, and its report is what people will ask for when you request help online.

## Topics Covered

- Running a health check
- Reading warnings
- The full report
- Checking network communication with `ros2 multicast`
- The ROS 2 daemon
- Asking for help effectively

---

## Running a Health Check

```bash
$ ros2 doctor
All 5 checks passed
```

The checks cover:

| Check | Looks at |
|-------|----------|
| **Platform** | Operating system and whether it matches the ROS 2 distribution |
| **Network** | Network interfaces and whether multicast is likely to work |
| **Middleware** | The RMW implementation in use |
| **Package** | Whether installed ROS 2 packages are up to date with the distribution |
| **Topic** | The running system: publishers without subscribers and vice versa |

The alias `ros2 wtf` ("where's the fault") does the same thing.

---

## Reading Warnings

Start turtlesim alone and run the check again:

```bash
$ ros2 run turtlesim turtlesim_node
```

```bash
$ ros2 doctor
/opt/ros/jazzy/lib/python3.12/site-packages/ros2doctor/api/topic.py: ...
UserWarning: Subscriber without publisher detected on /turtle1/cmd_vel.
1/5 check(s) failed
Failed modules:  topic
```

This warning is harmless here, since nobody is driving the turtle yet. But the same warning on a robot can point straight to the bug: if your controller subscribes to `/cmd_vel` and nothing publishes it, there's a name mismatch somewhere. Start the teleop node and the warning disappears.

Typical warnings and what they mean:

| Warning | Meaning |
|---------|---------|
| `Publisher without subscriber detected on /x` | Data is produced but nobody uses it. Fine for debug topics; suspicious for important ones |
| `Subscriber without publisher detected on /x` | A node waits for data that never comes. Often a name mismatch |
| Package version warnings | Installed packages are older than the latest release: run `sudo apt update && sudo apt upgrade` |
| Network or multicast warnings | Discovery between computers may not work |

---

## The Full Report

```bash
$ ros2 doctor --report
   NETWORK CONFIGURATION
inet         : 127.0.0.1
...
   PLATFORM INFORMATION
system           : Linux
platform info    : Linux-6.8.0-...-x86_64-with-glibc2.39
release          : 6.8.0-...
...
   RMW MIDDLEWARE
middleware name    : rmw_fastrtps_cpp
   ROS 2 INFORMATION
distribution name      : jazzy
distribution type      : ros2
distribution status    : active
release platforms      : {'debian': ['bookworm'], ..., 'ubuntu': ['noble']}
   TOPIC LIST
topic               : /turtle1/cmd_vel
publisher count     : 0
subscriber count    : 1
...
```

| Option | Effect |
|--------|--------|
| `-r`, `--report` | Print the full report |
| `-rf`, `--report-failed` | Report only the sections of failed checks |
| `-iw`, `--include-warnings` | Treat warnings as failures |

The report answers the questions every helper asks first: Which distribution? Which OS? Which middleware? Which network interfaces?

---

## Checking Multicast

ROS 2 discovery uses **multicast** by default: nodes announce themselves to everyone on the network. If nodes on two computers can't see each other, check whether multicast messages get through.

On computer A:

```bash
$ ros2 multicast receive
Waiting for UDP multicast datagram...
```

On computer B (or in another terminal on the same computer):

```bash
$ ros2 multicast send
```

Computer A should print:

```text
Received from xx.xx.xx.xx:43751: 'Hello World!'
```

If nothing arrives:

| Cause | Fix |
|-------|-----|
| Firewall blocks the traffic | Allow UDP multicast, e.g. `sudo ufw allow in proto udp to 224.0.0.0/4` and `sudo ufw allow in proto udp from 224.0.0.0/4` |
| Computers on different networks or subnets | Connect them to the same network |
| Network doesn't support multicast (some Wi-Fi, VPNs, WSL2, cloud) | Use a discovery server or static peers (see the [ROS 2 networking docs](https://docs.ros.org/en/jazzy/Concepts/Intermediate/About-Domain-ID.html)) |
| Different `ROS_DOMAIN_ID` | Make them equal (multicast itself works, but ROS 2 ignores the other domain) |

---

## The ROS 2 Daemon

The CLI uses a background process, the **ROS 2 daemon**, which keeps a cache of the discovered graph so commands like `ros2 topic list` answer quickly. Occasionally the cache becomes stale: nodes that stopped long ago still appear, or new ones are missing.

```bash
$ ros2 daemon status
The daemon is running
$ ros2 daemon stop
The daemon has been stopped
$ ros2 daemon start
The daemon has been started
```

If the CLI shows something you don't believe, restart the daemon before investigating further. You can also bypass it for a single command with `--no-daemon`, for example `ros2 node list --no-daemon`.

---

## Asking for Help Effectively

When you can't solve a problem yourself, the ROS community is helpful: [Robotics Stack Exchange](https://robotics.stackexchange.com/) (tag `ros2`) for questions, and [ROS Discourse](https://discourse.ros.org/) for discussions. You get faster and better answers if you include:

1. **What you want to do**, in one sentence.
2. **What happens instead**, with the **complete** error message as text, not a screenshot.
3. **Your setup**: the relevant parts of `ros2 doctor --report` (distribution, OS, middleware).
4. **A minimal example**: the smallest code or launch file that still shows the problem.
5. **What you already tried.**

Making the minimal example often reveals the bug on its own.

---

## Try It Yourself

1. Run `ros2 doctor` with nothing running, then with turtlesim and teleop running. Compare the results.
2. Find your distribution status and middleware name in `ros2 doctor --report`.
3. Run `ros2 multicast receive` and `ros2 multicast send` in two terminals on the same computer.
4. Start the talker with `ROS_DOMAIN_ID=5` and the listener with the default. Which tool shows you the problem, and which doesn't?
5. Stop all nodes and run `ros2 node list`. If old nodes still appear, restart the daemon and compare.

---

⬅️ [Previous: Common Errors](common-errors.md) | [⬆️ Debugging & Troubleshooting](README.md) | [Next: rosbag2](rosbag.md) ➡️
