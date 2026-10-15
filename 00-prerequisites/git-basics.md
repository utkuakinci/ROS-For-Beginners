# Git Basics

Git is a version control system: it records every change you make to your files so you can review history, undo mistakes, and collaborate. Almost every ROS 2 package lives in a Git repository, and you will clone other people's packages and track your own constantly.

## Topics Covered

- Installing and configuring Git
- Cloning a repository
- Staging and committing changes
- Viewing history and undoing changes
- Branches and merging
- Working with remotes (`push`, `pull`)
- Writing a `.gitignore` for a ROS 2 workspace

---

## Installing and Configuring Git

```bash
$ sudo apt install git
$ git --version
git version 2.43.0
```

Tell Git who you are. This name and email are attached to every commit you make:

```bash
$ git config --global user.name "Your Name"
$ git config --global user.email "you@example.com"
$ git config --global init.defaultBranch main
```

---

## Key Concepts

| Term | Meaning |
|------|---------|
| **Repository (repo)** | A project folder whose history Git tracks (it contains a hidden `.git` folder) |
| **Commit** | A snapshot of your files at a point in time, with a message describing the change |
| **Branch** | An independent line of development |
| **Remote** | A copy of the repository hosted elsewhere, e.g. on GitHub |

Every file moves through three areas:

```text
 Working directory  ──git add──▶  Staging area  ──git commit──▶  Repository
 (files you edit)                 (next snapshot)                (history)
```

---

## Cloning a Repository

`git clone` downloads a repository together with its full history. This is how you get ROS 2 packages that are not available through `apt`, or this guide itself:

```bash
$ cd ~
$ git clone https://github.com/utkuakinci/ROS-For-Beginners.git
Cloning into 'ROS-For-Beginners'...
$ cd ROS-For-Beginners
```

In a ROS 2 workspace, source packages are cloned into the `src` folder:

```bash
$ cd ~/ros2_ws/src
$ git clone https://github.com/ros/ros_tutorials.git -b jazzy
```

`-b jazzy` checks out the branch that matches your ROS 2 distribution. Many ROS repositories keep one branch per distribution.

---

## Starting Your Own Repository

```bash
$ mkdir ~/my_robot && cd ~/my_robot
$ git init
Initialized empty Git repository in /home/utku/my_robot/.git/
```

---

## The Everyday Workflow

```bash
$ echo "# My Robot" > README.md

$ git status                  # What has changed?
On branch main
Untracked files:
        README.md

$ git add README.md           # Stage a file (use `git add .` for everything)
$ git commit -m "Add README"  # Save the snapshot
[main (root-commit) 1a2b3c4] Add README
 1 file changed, 1 insertion(+)
```

Write commit messages that explain *what* changed in the imperative mood: `Add LiDAR driver launch file`, not `stuff` or `fixed things`.

### Viewing history and differences

```bash
$ git log --oneline           # Compact history
1a2b3c4 Add README

$ git diff                    # Unstaged changes
$ git diff --staged           # Staged changes (what will be committed)
```

### Undoing changes

```bash
$ git restore file.py            # Discard unstaged edits to a file
$ git restore --staged file.py   # Unstage a file (keep the edits)
$ git revert <commit>            # Create a new commit that undoes an old one
```

> ⚠️ `git restore file.py` permanently discards your uncommitted edits to that file.

---

## Branches and Merging

Branches let you work on a feature without disturbing working code on `main`.

```bash
$ git branch                       # List branches (* marks the current one)
* main

$ git switch -c feature/lidar      # Create and switch to a new branch
# ...edit files, add, commit...

$ git switch main                  # Go back to main
$ git merge feature/lidar          # Bring the feature's commits into main
$ git branch -d feature/lidar      # Delete the merged branch
```

If both branches changed the same lines, Git reports a **merge conflict** and marks the file:

```text
<<<<<<< HEAD
speed = 0.5
=======
speed = 1.0
>>>>>>> feature/lidar
```

Edit the file to keep the correct version, remove the markers, then `git add` and `git commit`.

---

## Working with Remotes

A remote is usually a repository on GitHub. A cloned repository already has a remote named `origin`.

```bash
$ git remote -v
origin  https://github.com/utkuakinci/ROS-For-Beginners.git (fetch)
origin  https://github.com/utkuakinci/ROS-For-Beginners.git (push)

$ git pull                          # Download and merge new commits
$ git push                          # Upload your commits
$ git push -u origin feature/lidar  # Push a new branch for the first time
```

To publish a repository you created locally, create an empty repository on GitHub, then:

```bash
$ git remote add origin https://github.com/<you>/my_robot.git
$ git push -u origin main
```

On GitHub, you typically push a branch and open a **Pull Request** so others can review the change before it is merged into `main`. See [CONTRIBUTING.md](../CONTRIBUTING.md) for the workflow used in this repository.

---

## A `.gitignore` for a ROS 2 Workspace

Building a ROS 2 workspace creates `build/`, `install/`, and `log/` folders. They are generated, large, and machine-specific, so they should never be committed. List them in a `.gitignore` file at the root of your repository:

```gitignore
# colcon build artifacts
build/
install/
log/

# Python
__pycache__/
*.pyc

# Editors
.vscode/
*~
```

```bash
$ git status --ignored    # Show which files are being ignored
```

---

## Try It Yourself

1. Create a folder `~/git_practice`, run `git init`, and add a `README.md`. Commit it.
2. Create a branch `feature/notes`, add a file `notes.txt`, and commit it.
3. Switch back to `main` and confirm `notes.txt` is not there. Merge the branch and check again.
4. Add a `.gitignore` that ignores `build/`. Create a `build/` folder with a file inside and verify with `git status` that Git ignores it.
5. Run `git log --oneline` and read the history you created.

---

⬅️ [Previous: Linux Basics](linux-basics.md) | [⬆️ Prerequisites](README.md) | [Next: Python Basics](python-basics.md) ➡️
