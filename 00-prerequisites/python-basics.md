# Python Basics

ROS 2 programs written in Python use the `rclpy` library. You do not need to be a Python expert, but `rclpy` code leans heavily on a few features: classes, inheritance, and callbacks. This lesson focuses on exactly those, and ends with a preview of a real ROS 2 node so you can see where each concept is used.

## Topics Covered

- Running Python scripts
- Variables, functions, and modules
- Classes and inheritance
- Callbacks and passing functions as arguments
- Type hints
- Virtual environments and `pip` (and why ROS 2 mostly avoids them)

---

## Running Python

Ubuntu ships with Python 3. Check your version (ROS 2 Jazzy uses Python 3.12, Humble uses 3.10):

```bash
$ python3 --version
Python 3.12.3
```

Write a script and run it:

```python
# hello.py
print("Hello, robot!")
```

```bash
$ python3 hello.py
Hello, robot!
```

---

## Variables, Data Types, and Control Flow

```python
robot_name = "turtle1"        # str
speed = 0.5                   # float
wheel_count = 4               # int
is_moving = False             # bool
waypoints = [(0, 0), (1, 2)]  # list of tuples
limits = {"max_speed": 1.0, "max_turn": 2.0}   # dict

print(f"{robot_name} moves at {speed} m/s")    # f-string formatting

if speed > limits["max_speed"]:
    print("Too fast!")
elif speed == 0:
    print("Stopped")
else:
    print("OK")

for x, y in waypoints:
    print(f"Going to ({x}, {y})")
```

> 💡 Python uses **indentation** (4 spaces) to define blocks. Mixing tabs and spaces causes errors.

---

## Functions

```python
def clamp(value, minimum, maximum):
    """Limit value to the range [minimum, maximum]."""
    return max(minimum, min(value, maximum))


print(clamp(2.5, -1.0, 1.0))   # 1.0
```

Default and keyword arguments appear constantly in ROS 2 APIs:

```python
def move(linear=0.0, angular=0.0):
    print(f"linear={linear}, angular={angular}")


move(angular=1.5)    # linear=0.0, angular=1.5
```

---

## Modules and Imports

A module is simply a `.py` file. Import code from other modules instead of copying it:

```python
import math
from geometry_utils import clamp          # your own file geometry_utils.py
from std_msgs.msg import String           # a ROS 2 message type (later!)

print(math.pi)
```

The `if __name__ == "__main__":` block runs only when the file is executed directly, not when it is imported. Every ROS 2 Python node has one:

```python
def main():
    print("Node starting")


if __name__ == "__main__":
    main()
```

---

## Classes and Objects

A **class** bundles data (*attributes*) and behavior (*methods*). Every ROS 2 node you write will be a class.

```python
class Robot:
    def __init__(self, name, max_speed):
        # __init__ runs when an object is created; self is the object itself
        self.name = name
        self.max_speed = max_speed
        self.speed = 0.0

    def set_speed(self, speed):
        self.speed = min(speed, self.max_speed)
        print(f"{self.name} speed set to {self.speed}")


turtle = Robot("turtle1", max_speed=2.0)   # create an object (an instance)
turtle.set_speed(5.0)                      # turtle1 speed set to 2.0
print(turtle.name)                         # turtle1
```

---

## Inheritance

A class can **inherit** from another class to reuse and extend its behavior. `super().__init__(...)` calls the parent class's constructor.

```python
class Robot:
    def __init__(self, name):
        self.name = name

    def describe(self):
        print(f"I am {self.name}")


class MobileRobot(Robot):
    def __init__(self, name, wheels):
        super().__init__(name)      # let Robot set up self.name
        self.wheels = wheels

    def drive(self):
        print(f"{self.name} drives on {self.wheels} wheels")


rover = MobileRobot("rover", wheels=4)
rover.describe()    # inherited from Robot
rover.drive()       # defined in MobileRobot
```

This is exactly how ROS 2 nodes work: your class inherits from `rclpy`'s `Node` class and gets all of its features (publishers, timers, logging...) for free.

---

## Callbacks: Functions as Arguments

In Python, functions are objects. You can pass a function to another function, which calls it later. A function passed this way is called a **callback**.

```python
def on_obstacle(distance):
    print(f"Obstacle at {distance} m!")


def check_sensor(reading, callback):
    if reading < 0.5:
        callback(reading)       # call the function we were given


check_sensor(0.3, on_obstacle)  # Obstacle at 0.3 m!
```

Note that we pass `on_obstacle` (the function itself), **not** `on_obstacle()` (the result of calling it).

ROS 2 is built around callbacks: you tell ROS 2 "call this function whenever a message arrives" or "call this function every 0.5 seconds", and ROS 2 calls it for you. Bound methods (`self.some_method`) work the same way:

```python
class Counter:
    def __init__(self):
        self.count = 0

    def tick(self):
        self.count += 1
        print(f"tick {self.count}")


def run_every_step(callback, steps):
    for _ in range(steps):
        callback()


counter = Counter()
run_every_step(counter.tick, 3)   # tick 1, tick 2, tick 3
```

Short callbacks are sometimes written as **lambdas**, small anonymous functions:

```python
run_every_step(lambda: print("beep"), 2)
```

---

## Type Hints

Type hints document what a function expects and returns. Python does not enforce them, but they make code easier to read and let editors catch mistakes. You will see them in ROS 2 code:

```python
def distance(x: float, y: float) -> float:
    return (x ** 2 + y ** 2) ** 0.5


def greet(names: list[str]) -> None:
    for name in names:
        print(f"Hello {name}")
```

---

## Putting It Together: A Preview of a ROS 2 Node

Here is a complete minimal ROS 2 node. You will write and run nodes like this in [Module 04](../04-python/README.md); for now, just recognize the Python features from this lesson:

```python
import rclpy
from rclpy.node import Node


class HelloNode(Node):                       # inheritance
    def __init__(self):
        super().__init__("hello_node")       # call the parent constructor
        self.count = 0                       # attribute
        self.create_timer(1.0, self.tick)    # pass a method as a callback

    def tick(self):                          # called by ROS 2 every second
        self.count += 1
        self.get_logger().info(f"Hello #{self.count}")   # f-string


def main():
    rclpy.init()
    node = HelloNode()
    rclpy.spin(node)                         # keep running and calling callbacks
    rclpy.shutdown()


if __name__ == "__main__":
    main()
```

---

## Virtual Environments and `pip`

In general Python development, you install packages with `pip`, usually inside a **virtual environment** so each project has isolated dependencies:

```bash
$ python3 -m venv .venv
$ source .venv/bin/activate
(.venv) $ pip install numpy
```

ROS 2 is different. Its Python libraries (`rclpy`, message packages, etc.) are installed system-wide through `apt`, and dependencies of ROS 2 packages are installed with **`rosdep`**, which also uses `apt` (covered in [Module 03](../03-workspaces-and-packages/README.md)). For example:

```bash
$ sudo apt install python3-numpy
```

> ⚠️ On Ubuntu 24.04, `pip install` outside a virtual environment fails with an `externally-managed-environment` error. This is intentional: prefer `apt` packages (`python3-<name>`) for ROS 2 work. A plain virtual environment hides the system ROS 2 libraries, so avoid activating one when running ROS 2 nodes unless you know how to configure it.

---

## Try It Yourself

1. Write a class `Battery` with an attribute `level` (starting at 100) and a method `drain(amount)` that never lets the level drop below 0.
2. Create a subclass `SmartBattery` that calls a callback function passed to its constructor whenever the level falls below 20.
3. Write a function `on_low_battery(level)` that prints a warning, create a `SmartBattery` with it, and drain the battery in a loop until the warning appears.
4. Add type hints to all of your methods.

---

⬅️ [Previous: Git Basics](git-basics.md) | [⬆️ Prerequisites](README.md)
