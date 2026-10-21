# ros2 param

`ros2 param` reads and changes node parameters while the system is running, and saves them to or loads them from YAML files. It is the main tool for tuning a robot: change a value, observe the effect, and save the result once you are happy.

> 📖 Concepts: [Parameters](../01-ros2-basics/parameters.md)

## Topics Covered

- Listing, reading, and describing parameters
- Setting and deleting parameters
- Saving and loading parameter files
- A practical tuning workflow
- Troubleshooting

---

## Command Overview

| Command | Purpose |
|---------|---------|
| `ros2 param list [node]` | List parameters of one node or of all nodes |
| `ros2 param get <node> <name>` | Read a parameter's value |
| `ros2 param describe <node> <name>` | Show a parameter's type, description, and constraints |
| `ros2 param set <node> <name> <value>` | Change a parameter at runtime |
| `ros2 param delete <node> <name>` | Remove a parameter (if the node allows it) |
| `ros2 param dump <node>` | Print all of a node's parameters as YAML |
| `ros2 param load <node> <file>` | Load parameters from a YAML file into a running node |

The examples below assume turtlesim is running.

---

## Listing Parameters

```bash
$ ros2 param list /turtlesim
  background_b
  background_g
  background_r
  qos_overrides./parameter_events.publisher.depth
  ...
  use_sim_time
```

Without a node name, the parameters of all nodes are listed, grouped by node.

| Option | Effect |
|--------|--------|
| `--param-prefixes <prefix> ...` | Show only parameters whose names start with a prefix |
| `--param-type` | Show each parameter's type |

```bash
$ ros2 param list /turtlesim --param-prefixes background
  background_b
  background_g
  background_r
```

Prefix filtering is essential on stacks like Nav2, where a single node can have more than a hundred parameters.

---

## Reading and Describing

```bash
$ ros2 param get /turtlesim background_r
Integer value is: 69

$ ros2 param describe /turtlesim background_r
Parameter name: background_r
  Type: integer
  Description: Red channel of the background color
  Constraints:
    Min value: 0
    Max value: 255
    Step: 1
```

Always run `describe` before setting an unfamiliar parameter. It tells you the exact type and the allowed range, and whether the parameter is **read-only** (settable only at startup).

---

## Setting and Deleting

```bash
$ ros2 param set /turtlesim background_r 200
Set parameter successful
```

The value is parsed as YAML, so its type follows from how you write it:

| You type | Interpreted as |
|----------|----------------|
| `5` | integer |
| `5.0` | double |
| `true` / `false` | bool |
| `hello` or `"hello"` | string |
| `"[1, 2, 3]"` | integer array |
| `"[0.5, 1.5]"` | double array |
| `"['a', 'b']"` | string array |

> ⚠️ Writing `1` for a parameter declared as `double` fails with a type error. Write `1.0`.

`ros2 param delete` removes a parameter. Most nodes declare their parameters as required and refuse the deletion, so you will rarely use it.

---

## Saving and Loading

```bash
$ ros2 param dump /turtlesim > turtlesim_params.yaml
```

```yaml
/turtlesim:
  ros__parameters:
    background_b: 255
    background_g: 86
    background_r: 200
    ...
```

Load the file back into a running node:

```bash
$ ros2 param load /turtlesim turtlesim_params.yaml
Set parameter background_b successful
Set parameter background_g successful
Set parameter background_r successful
...
```

| Option (`load`) | Effect |
|-----------------|--------|
| `--no-use-wildcard` | Ignore the `/**` wildcard section of the file |

To apply a file at startup instead, pass it to `ros2 run` with `--params-file` (see [ros2 run](ros2-run.md)) or to a node in a launch file.

> 💡 The dump includes every parameter, also the ones you never changed. Before adding a dumped file to your project, delete the lines you don't need so the file only shows the values that matter.

---

## A Practical Tuning Workflow

This is how parameters are tuned on a real robot:

1. **Find** the relevant parameters: `ros2 param list <node> --param-prefixes <group>`.
2. **Understand** them: `ros2 param describe`, plus the package's documentation.
3. **Experiment** at runtime with `ros2 param set` and observe the robot (or a plot of a topic).
4. **Save** the final values with `ros2 param dump`.
5. **Commit** the cleaned-up YAML file to your package's `config/` folder and load it from your launch file.

Graphical alternative: `ros2 run rqt_reconfigure rqt_reconfigure` shows all parameters in a window with sliders and checkboxes.

---

## Troubleshooting

| Symptom | Likely cause |
|---------|--------------|
| `Node not found` | Wrong node name. It must be the full name, including any namespace |
| `Parameter not set` / not in the list | The node never declared it. Check the spelling |
| `Setting parameter failed: ...` type error | Wrong type, e.g. `1` instead of `1.0` |
| `Setting parameter failed: ...` read-only or out of range | Check `ros2 param describe` for constraints |
| `Set parameter successful`, but nothing changes | The node reads the parameter only at startup. Restart it with the new value |
| A parameter file seems to be ignored | The node name in the file does not match, or `ros__parameters` is misspelled |

---

## Try It Yourself

1. List only the parameters starting with `qos_overrides` on turtlesim and count them.
2. Change the background to orange (255, 165, 0) at runtime, dump the parameters, and remove every line from the file except the three background colors.
3. Restart turtlesim, then load your trimmed file into it with `ros2 param load`.
4. Try `ros2 param set /turtlesim use_sim_time true`. Then try setting it to `1`. Compare the results.
5. Open `rqt_reconfigure` and change the background colors with the sliders.

---

⬅️ [Previous: ros2 action](ros2-action.md) | [⬆️ CLI Tools](README.md) | [Next: ros2 run](ros2-run.md) ➡️
