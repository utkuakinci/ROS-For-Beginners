# Contributing to ROS-For-Beginners

Thank you for taking the time to contribute! 🎉

This repository aims to provide a high-quality, beginner-friendly roadmap for learning ROS 2. Contributions from the community—whether fixing typos, improving explanations, adding code examples, or reporting issues—are warmly welcomed.

---

## 🛠️ How Can You Contribute?

### 1. Reporting Bugs & Issues
If you find an error in the documentation, broken code snippets, or incorrect commands:
1. Check the [Issues](../../issues) tab to see if it has already been reported.
2. If not, open a new issue with a clear description and steps to reproduce the problem.

### 2. Improving Documentation
- Fix typos, grammatical errors, or awkward phrasing.
- Clarify complex ROS 2 concepts.
- Add diagrams, images, or terminal outputs to enhance visual understanding.
- Write content for lessons still marked as **🚧 Draft**.

### 3. Adding Code Examples & Projects
- Ensure code snippets adhere to ROS 2 standard style guidelines:
  - **Python:** PEP 8 compliance.
  - **C++:** ROS 2 C++ Code Style Guide.
- Include comments explaining key ROS 2 APIs (`rclpy` / `rclcpp`).
- Verify that code compiles and runs on supported ROS 2 distributions (e.g., Humble, Jazzy).

---

## 🚀 Development & Workflow

Follow these simple steps to contribute your changes:

1. **Fork the Repository**
   Click the "Fork" button at the top right of this repository to create your own copy.

2. **Clone Your Fork**
   ```bash
   git clone https://github.com/YOUR-USERNAME/ROS-For-Beginners.git
   cd ROS-For-Beginners
   ```

3. **Create a Branch**
   Use a short, descriptive name with a prefix such as `docs/`, `fix/`, `feat/`, or `chore/`:
   ```bash
   git checkout -b docs/topics-lesson
   ```

4. **Make Your Changes**
   Follow the [lesson structure](#-lesson-structure) below when writing or editing lessons.

5. **Commit Your Changes**
   Write clear commit messages in the imperative mood:
   ```bash
   git add .
   git commit -m "Add publisher example to topics lesson"
   ```

6. **Push to Your Fork**
   ```bash
   git push origin docs/topics-lesson
   ```

7. **Open a Pull Request**
   Open a Pull Request against the `main` branch and describe what you changed and why. Link any related issue (e.g., `Closes #12`).

---

## 📝 Lesson Structure

Every lesson lives in its module folder (for example `01-ros2-basics/topics.md`) and should follow this structure:

```markdown
# Lesson Title

Short introduction: what this lesson covers and why it matters.

## Topics Covered
- Topic one
- Topic two

## <Content sections>
Explanations, commands, code, and expected output.

## Try It Yourself
A small exercise for the reader.

---

⬅️ [Previous: ...](previous.md) | [⬆️ Module](README.md) | [Next: ...](next.md) ➡️
```

Guidelines:
- Write in **English** and keep the tone friendly and beginner-oriented.
- Show the **expected output** after commands so readers can check their progress.
- Use fenced code blocks with a language tag (`bash`, `python`, `cpp`, `xml`, `yaml`).
- Keep the navigation links at the bottom of each page working.
- When a lesson is complete, remove the `🚧 Status: Draft` banner.

---

## ✅ Pull Request Checklist

Before submitting, please make sure that:
- [ ] Commands and code have been tested on a supported ROS 2 distribution.
- [ ] Links between pages work.
- [ ] Spelling and formatting have been checked.
- [ ] The PR focuses on a single topic or lesson.

---

## 💬 Code of Conduct

Be respectful and constructive. Everyone here is learning—questions are always welcome, and feedback should help others improve.
