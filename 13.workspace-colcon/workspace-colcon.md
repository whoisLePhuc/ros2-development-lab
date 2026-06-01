# Bài 13: Workspace & Colcon

Every ROS2 project lives inside a **workspace**. `colcon` is the build tool that turns source code into runnable packages. This lesson explains workspace layout, essential `colcon` commands, overlay stacking, and how to fix common build errors.

---

## 1. Workspace Structure

A standard ROS2 workspace has four top-level directories:

```
ros2_ws/
├── src/          # Clone or create packages here
├── build/        # Intermediate build files (CMake, Python caches)
├── install/      # Final binaries, libraries, and setup scripts
└── log/          # Build logs for every package
```

Create a workspace:

```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws
```

After building, always source the overlay:

```bash
source install/setup.bash
```

---

## 2. Essential Colcon Commands

### Build everything

```bash
cd ~/ros2_ws
colcon build
```

### Build a single package

```bash
colcon build --packages-select my_package
```

This saves time when you only changed one package.

### Symlink install (Python packages)

```bash
colcon build --symlink-install
```

With `--symlink-install`, Python files in `install/` are symlinks back to `src/`. Edit code and run immediately without rebuilding.

### Pass CMake arguments

```bash
colcon build --cmake-args -DCMAKE_BUILD_TYPE=Release
```

Use `Release` for optimized C++ binaries. Use `Debug` when you need debug symbols.

### Build with parallel jobs

```bash
colcon build --parallel-workers 4
```

Limit parallel jobs if your machine runs out of RAM.

### Clean and rebuild

```bash
rm -rf build/ install/ log/
colcon build
```

---

## 3. Workspace Overlay Concept

ROS2 uses **underlay/overlay stacking**.

- **Underlay**: the system ROS2 installation (`/opt/ros/humble`)
- **Overlay**: your workspace (`~/ros2_ws/install`)

When you source multiple setup files, the last one wins for packages it contains, but it keeps access to everything underneath.

```bash
# Terminal setup order
source /opt/ros/humble/setup.bash      # underlay
source ~/ros2_ws/install/setup.bash    # overlay
```

You can stack overlays:

```bash
source /opt/ros/humble/setup.bash
source ~/company_ws/install/setup.bash
source ~/project_ws/install/setup.bash
```

`project_ws` sees its own packages, then `company_ws` packages, then system packages.

**Rule**: never source an overlay in your `.bashrc`. Always source it per terminal so you know exactly which workspace is active.

---

## 4. Common Build Errors and Fixes

### Missing dependencies

```bash
rosdep install -i --from-path src --rosdistro humble -y
```

Run this from the workspace root. It reads every `package.xml` and installs system dependencies via `apt` or `pip`.

### CMake cache issues

If you renamed a package or moved files and the build breaks:

```bash
rm -rf build/my_package
rm -rf install/my_package
colcon build --packages-select my_package
```

### Package not found

```bash
# Verify the package is known
ros2 pkg list | grep my_package

# If missing, source the workspace again
source install/setup.bash
```

### Python entry point not found

Check `setup.py` or `setup.cfg`:

```python
entry_points={
    'console_scripts': [
        'my_node = my_package.my_node:main',
    ],
},
```

After editing `setup.py`, rebuild:

```bash
colcon build --packages-select my_package --symlink-install
```

### Mixed Python and C++ build failures

If a Python package fails because a C++ interface package changed:

```bash
colcon build --packages-select my_interfaces my_python_pkg
```

Build the dependency first, then the consumer.

---

## 5. Optimization Tips

- Use `--symlink-install` during development. Skip it for release builds.
- Use `--packages-select` to avoid rebuilding unchanged packages.
- Use `--cmake-args -DCMAKE_BUILD_TYPE=Release` for production.
- Keep `build/` and `install/` on a fast SSD if possible.
- Add `COLCON_EXTENSION_BLOCKLIST=colcon_core.event_handler.desktop_notification` to your environment to silence desktop notifications during builds.

---

## 6. Key Takeaways

- `src/` is for code. `build/`, `install/`, `log/` are generated.
- Source `install/setup.bash` after every build.
- Overlays stack. The last sourced workspace takes precedence.
- `rosdep` is your friend for missing system dependencies.

---

## Bài tiếp theo

[Bài 14: ROS2 Package](../14.ros2-package/ros2-package.md)
