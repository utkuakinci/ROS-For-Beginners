# 🗺️ Repository Development Roadmap & Todo List

## Repository Setup
- [x] Create module and project skeleton files
- [x] Complete `CONTRIBUTING.md`
- [x] Add ROS 2 workspace entries to `.gitignore`
- [x] Add `.gitattributes` for consistent line endings

## Decisions
- [ ] Choose the target ROS 2 distribution (Jazzy only, or Humble + Jazzy)
- [ ] Add an installation / environment setup module (native Ubuntu, WSL2, Docker)
- [ ] Add missing topics: C++ prerequisites, `CMakeLists.txt` / `ament_cmake`, QoS
- [ ] Reorder `09-navigation`: mapping before localization

## Content
- [ ] Installation and environment setup lesson
- [x] `00-prerequisites` lessons
- [x] `01-ros2-basics`: Nodes and Topics lessons
- [ ] `projects/01-turtlesim`
- [x] Remaining `01-ros2-basics` lessons
- [ ] `02-ros2-cli`
- [ ] `03-workspaces-and-packages`
- [ ] `04-python`

## Code Infrastructure
- [ ] `src/` directory for buildable example packages
- [ ] `.devcontainer/` for a ready-to-use ROS 2 environment
- [ ] GitHub Actions workflow running `colcon build` on every push
