# Linux Basics

ROS 2 is developed and run primarily on Ubuntu Linux, and nearly everything you do with it happens in a terminal: building packages, starting nodes, and inspecting a running robot. This lesson covers the terminal skills you will use every single day.

## Topics Covered

- Opening and using the terminal
- Navigating the filesystem (`cd`, `ls`, `pwd`)
- Creating, moving, and deleting files
- Permissions and `sudo`
- Installing software with `apt`
- Environment variables and `~/.bashrc`
- Working with multiple terminals

---

## The Terminal

Open a terminal on Ubuntu with **`Ctrl` + `Alt` + `T`**. You will see a prompt like this:

```text
utku@laptop:~$
```

| Part | Meaning |
|------|---------|
| `utku` | Your username |
| `laptop` | The computer's hostname |
| `~` | The current directory (`~` is short for your home directory, e.g. `/home/utku`) |
| `$` | You are a normal user (a `#` would mean you are `root`) |

In this guide, lines starting with `$` are commands you type (without the `$`), and the lines below them are the output you should expect.

### Three keys that save you hours

- **`Tab`** — autocompletes commands, file names, and even ROS 2 package names. Press it twice to see all options.
- **`↑` / `↓`** — scroll through previous commands.
- **`Ctrl` + `C`** — stops the running program. This is how you stop a ROS 2 node.

---

## Navigating the Filesystem

Linux organizes everything in a single tree that starts at `/` (the *root*). Your personal files live in `/home/<username>`, and ROS 2 is installed under `/opt/ros/<distro>`.

```bash
$ pwd                 # Print Working Directory: where am I?
/home/utku

$ ls                  # List the contents of the current directory
Desktop  Documents  Downloads  Music  Pictures

$ ls -la              # Long format, including hidden files (names starting with .)
drwxr-x--- 15 utku utku 4096 Oct 16 10:12 .
-rw-r--r--  1 utku utku 3771 Oct 16 10:12 .bashrc
drwxr-xr-x  2 utku utku 4096 Oct 16 10:12 Desktop
...
```

Move around with `cd` (Change Directory):

```bash
$ cd Documents        # Go into a folder (relative path)
$ cd ..               # Go up one level
$ cd /opt/ros         # Go to an absolute path
$ cd ~                # Go back home (plain `cd` does the same)
$ cd -                # Go back to the previous directory
```

> 💡 **Relative vs absolute paths:** An absolute path starts with `/` and works from anywhere (`/home/utku/ros2_ws`). A relative path starts from where you are (`ros2_ws/src`).

---

## Working with Files and Directories

```bash
$ mkdir -p ~/ros2_ws/src          # Create a directory (and parents with -p)
$ cd ~/ros2_ws/src
$ touch notes.txt                 # Create an empty file
$ cp notes.txt notes_backup.txt   # Copy
$ mv notes_backup.txt old.txt     # Move or rename
$ rm old.txt                      # Delete a file
$ rm -r some_folder               # Delete a folder and everything inside it
```

> ⚠️ `rm` does not use a trash bin. Deleted files are gone. Double-check before running `rm -r`.

Reading files:

```bash
$ cat notes.txt            # Print the whole file
$ less /etc/os-release     # Scroll through a long file (press q to quit)
$ head -n 5 file.txt       # First 5 lines
$ tail -f log.txt          # Follow a file as it grows (Ctrl+C to stop)
```

Editing files: `nano` is the easiest terminal editor for beginners.

```bash
$ nano notes.txt
```

Save with **`Ctrl` + `O`**, then `Enter`; exit with **`Ctrl` + `X`**. You can of course also use a graphical editor such as VS Code (`code .` opens the current folder).

Finding things:

```bash
$ find ~/ros2_ws -name "*.py"        # Find files by name
$ grep -r "create_publisher" ~/ros2_ws/src   # Search text inside files
```

---

## Permissions and `sudo`

Every file has an owner and permissions that say who may **r**ead, **w**rite, and e**x**ecute it:

```text
-rwxr-xr--  1 utku utku  512 Oct 16 10:12 my_script.py
 │  │  │
 │  │  └── others: read only
 │  └───── group:  read + execute
 └──────── owner:  read + write + execute
```

A script must be executable before you can run it directly:

```bash
$ chmod +x my_script.py
$ ./my_script.py
```

System-wide changes (installing software, editing files under `/etc` or `/opt`) need administrator rights. Prefix the command with `sudo` and enter **your own** password:

```bash
$ sudo apt update
[sudo] password for utku:
```

> ⚠️ Only use `sudo` when a command actually needs it. Never run `colcon build` or `ros2` commands with `sudo` — it creates root-owned files in your workspace that cause confusing errors later.

---

## Installing Software with `apt`

Ubuntu installs software from *repositories* using `apt`. ROS 2 itself is installed this way.

```bash
$ sudo apt update                    # Refresh the list of available packages
$ sudo apt upgrade                   # Upgrade installed packages
$ sudo apt install tree              # Install a package
$ apt search turtlesim               # Search for packages
$ sudo apt remove tree               # Uninstall
```

ROS 2 packages follow the naming pattern `ros-<distro>-<package-name>`:

```bash
$ sudo apt install ros-jazzy-turtlesim
```

> 📝 Examples in this guide use **Jazzy**. If you use Humble, replace `jazzy` with `humble`.

---

## Environment Variables and `~/.bashrc`

Environment variables are named values that programs read to configure themselves.

```bash
$ echo $HOME
/home/utku

$ echo $PATH
/usr/local/bin:/usr/bin:/bin:...

$ export MY_ROBOT=turtle    # Set a variable for this terminal only
$ echo $MY_ROBOT
turtle

$ printenv | grep ROS       # Show all ROS-related variables
```

ROS 2 relies heavily on environment variables. They are set by **sourcing** a setup script:

```bash
$ source /opt/ros/jazzy/setup.bash
$ printenv | grep ROS
ROS_VERSION=2
ROS_PYTHON_VERSION=3
ROS_DISTRO=jazzy
...
```

`source` runs the script *inside the current terminal*, so the variables only exist in that terminal. Open a new terminal and they are gone. To avoid sourcing manually every time, add the line to `~/.bashrc`, a script that runs whenever a new terminal opens:

```bash
$ echo "source /opt/ros/jazzy/setup.bash" >> ~/.bashrc
$ source ~/.bashrc          # Apply it to the current terminal too
```

> 💡 `>>` appends to a file; a single `>` **overwrites** it. Be careful not to wipe your `.bashrc` with `>`.

If you ever see `ros2: command not found`, the cause is almost always a terminal that has not sourced ROS 2.

---

## Working with Multiple Terminals

A ROS 2 system is made of several programs running at the same time, and each one usually gets its own terminal. You will regularly have three or four open.

| Shortcut | Action |
|----------|--------|
| `Ctrl` + `Alt` + `T` | Open a new terminal window |
| `Ctrl` + `Shift` + `T` | Open a new tab in the current terminal |
| `Ctrl` + `Shift` + `C` / `V` | Copy / paste inside the terminal |
| `Ctrl` + `C` | Stop the running program |
| `Ctrl` + `L` | Clear the screen |

> 💡 Tools like **Terminator** (`sudo apt install terminator`) or **tmux** let you split one window into several panes, which is very handy for ROS 2 work.

---

## Try It Yourself

1. Create the folder `~/ros2_practice/notes` with a single command.
2. Inside it, create a file `commands.txt` with `nano` and write down five commands from this lesson.
3. Display the file with `cat`, then count its lines with `wc -l commands.txt`.
4. Set an environment variable `ROBOT_NAME`, print it, then open a new terminal and check that it is gone.
5. Add an `export ROBOT_NAME=...` line to `~/.bashrc`, open a new terminal, and confirm the variable now persists.

---

[⬆️ Prerequisites](README.md) | [Next: Git Basics](git-basics.md) ➡️
