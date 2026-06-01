# Bài 15: Launch File

Running `ros2 run` for every node quickly becomes tedious. A **launch file** starts multiple nodes, sets parameters, remaps topics, and wires namespaces in one command. ROS2 uses Python-based launch files, which are far more flexible than XML.

---

## 1. Launch File Concept

A launch file is a Python script that returns a `LaunchDescription`. The launch system reads this description and starts every entity inside it: nodes, other launch files, processes, even event handlers.

Benefits:

- Start many nodes with one command.
- Share configuration between robots.
- Override parameters and topic names without recompiling.

---

## 2. Basic Python Launch File

Create `launch/talkers.launch.py` inside a package:

```python
from launch import LaunchDescription
from launch_ros.actions import Node

def generate_launch_description():
    return LaunchDescription([
        Node(
            package='py_talker',
            executable='talker',
            name='my_talker',
            output='screen'
        ),
        Node(
            package='cpp_listener',
            executable='listener',
            name='my_listener',
            output='screen'
        ),
    ])
```

Register the launch file in `setup.py` (Python package):

```python
data_files=[
    ('share/ament_index/resource_index/packages',
        ['resource/' + package_name]),
    ('share/' + package_name, ['package.xml']),
    ('share/' + package_name + '/launch', ['launch/talkers.launch.py']),
],
```

Run it:

```bash
ros2 launch py_talker talkers.launch.py
```

---

## 3. Node Parameters

The `Node()` action accepts many arguments:

| Parameter | Meaning | Example |
|-----------|---------|---------|
| `package` | ROS2 package name | `'py_talker'` |
| `executable` | Entry point name | `'talker'` |
| `name` | Node name override | `'my_talker'` |
| `namespace` | Prepends to all topics/services | `'robot1'` |
| `parameters` | YAML file or dict of params | `[{'rate': 2.0}]` |
| `remappings` | Rename topics/services | `[('chatter', 'robot1/chatter')]` |
| `output` | Where stdout goes | `'screen'` or `'log'` |
| `arguments` | Extra CLI args for the node | `['--ros-args', '--log-level', 'debug']` |

### Example with parameters and remappings

```python
from launch import LaunchDescription
from launch_ros.actions import Node

def generate_launch_description():
    return LaunchDescription([
        Node(
            package='py_talker',
            executable='talker',
            name='talker',
            namespace='robot1',
            parameters=[{
                'publish_rate': 2.0,
                'message_prefix': 'Robot1 says'
            }],
            remappings=[
                ('chatter', 'robot1/chatter')
            ],
            output='screen'
        ),
    ])
```

In the node code, read parameters with:

```python
self.declare_parameter('publish_rate', 1.0)
rate = self.get_parameter('publish_rate').value
```

---

## 4. Include Other Launch Files

Break large systems into reusable pieces.

```python
from launch import LaunchDescription
from launch.actions import IncludeLaunchDescription
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.substitutions import ThisLaunchFileDir
import os

def generate_launch_description():
    return LaunchDescription([
        IncludeLaunchDescription(
            PythonLaunchDescriptionSource([
                os.path.join(
                    get_package_share_directory('other_pkg'),
                    'launch',
                    'other.launch.py'
                )
            ])
        ),
    ])
```

Import `get_package_share_directory`:

```python
from ament_index_python.packages import get_package_share_directory
```

---

## 5. Launch Arguments

Make launch files configurable from the command line.

```python
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument
from launch.substitutions import LaunchConfiguration
from launch_ros.actions import Node

def generate_launch_description():
    return LaunchDescription([
        DeclareLaunchArgument(
            'robot_name',
            default_value='turtlebot',
            description='Name of the robot'
        ),
        DeclareLaunchArgument(
            'use_sim',
            default_value='true',
            description='Use simulation or real hardware'
        ),
        Node(
            package='robot_driver',
            executable='driver_node',
            name=LaunchConfiguration('robot_name'),
            parameters=[{
                'use_sim': LaunchConfiguration('use_sim')
            }],
            output='screen'
        ),
    ])
```

Run with arguments:

```bash
ros2 launch my_pkg robot.launch.py robot_name:=my_robot use_sim:=false
```

---

## 6. Complete Robot Bringup Example

A realistic launch file for a mobile robot:

```python
import os
from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument, IncludeLaunchDescription
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.substitutions import LaunchConfiguration
from launch_ros.actions import Node

def generate_launch_description():
    pkg_share = get_package_share_directory('my_robot_bringup')
    default_rviz_config = os.path.join(pkg_share, 'config', 'robot.rviz')

    return LaunchDescription([
        DeclareLaunchArgument('use_rviz', default_value='true'),
        DeclareLaunchArgument('rviz_config', default_value=default_rviz_config),

        # Robot state publisher (URDF -> TF)
        Node(
            package='robot_state_publisher',
            executable='robot_state_publisher',
            parameters=[{
                'robot_description': open(
                    os.path.join(pkg_share, 'urdf', 'robot.urdf')
                ).read()
            }],
            output='screen'
        ),

        # LiDAR driver
        Node(
            package='rplidar_ros',
            executable='rplidar_composition',
            output='screen',
            parameters=[{
                'serial_port': '/dev/ttyUSB0',
                'serial_baudrate': 115200,
                'frame_id': 'laser',
                'angle_compensate': True
            }]
        ),

        # Camera driver
        Node(
            package='usb_cam',
            executable='usb_cam_node_exe',
            parameters=[{
                'video_device': '/dev/video0',
                'image_width': 640,
                'image_height': 480
            }],
            remappings=[('image_raw', 'camera/image_raw')],
            output='screen'
        ),

        # Teleop node
        Node(
            package='teleop_twist_keyboard',
            executable='teleop_twist_keyboard',
            prefix='xterm -e',
            remappings=[('cmd_vel', 'diffbot/cmd_vel')],
            output='screen'
        ),

        # RViz (optional)
        IncludeLaunchDescription(
            PythonLaunchDescriptionSource([
                os.path.join(
                    get_package_share_directory('rviz2'),
                    'launch',
                    'rviz.launch.py'
                )
            ]),
            launch_arguments={'config': LaunchConfiguration('rviz_config')}.items(),
            condition=IfCondition(LaunchConfiguration('use_rviz'))
        ),
    ])
```

> Note: `IfCondition` requires `from launch.conditions import IfCondition`.

---

## 7. Organizing Launch Files

Best practice directory layout:

```
my_robot_bringup/
├── launch/
│   ├── robot.launch.py          # Main bringup
│   ├── sensors.launch.py        # Cameras, LiDAR, IMU
│   ├── navigation.launch.py     # Nav2 stack
│   └── teleop.launch.py         # Manual control
├── config/
│   ├── robot.rviz
│   └── nav2_params.yaml
├── urdf/
│   └── robot.urdf
└── package.xml
```

---

## 8. Key Takeaways

- Launch files are Python. They return a `LaunchDescription`.
- `Node()` is the most common action. Know its parameters well.
- Use `DeclareLaunchArgument` + `LaunchConfiguration` for reusable configs.
- Split large systems into included launch files.
- Always register launch files in `setup.py` or `CMakeLists.txt` so `ros2 launch` can find them.

---

## Bài tiếp theo

[Bài 16: Logging & Debugging](../16.logging-debugging/logging-debugging.md)
