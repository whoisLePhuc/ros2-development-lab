# Bài 24: Spawn Robot — Đưa robot vào Gazebo

Sau khi Gazebo đã chạy, bạn cần đưa robot vào môi trường. ROS2 cung cấp node `spawn_entity.py` trong package `gazebo_ros` để làm việc này. Bài này hướng dẫn cách spawn từ command line, từ launch file, và cách spawn nhiều robot cùng lúc.

## 1. spawn_entity.py từ command line

Node `spawn_entity.py` đọc URDF hoặc SDF và tạo model trong Gazebo.

### Spawn từ topic robot_description

Cách phổ biến nhất. `robot_state_publisher` đã publish URDF lên topic `/robot_description`.

```bash
ros2 run gazebo_ros spawn_entity.py \
  -topic robot_description \
  -entity my_robot \
  -x 0 -y 0 -z 0.1
```

Các tham số:

| Tham số | Ý nghĩa | Ví dụ |
|---------|---------|-------|
| `-topic` | Topic chứa robot description | `robot_description` |
| `-file` | Đường dẫn file URDF/SDF trực tiếp | `/path/to/robot.urdf` |
| `-entity` | Tên entity trong Gazebo | `my_robot` |
| `-x` | Vị trí X | `0.0` |
| `-y` | Vị trí Y | `0.0` |
| `-z` | Vị trí Z | `0.1` |
| `-R` | Roll (rad) | `0.0` |
| `-P` | Pitch (rad) | `0.0` |
| `-Y` | Yaw (rad) | `0.0` |
| `-robot_namespace` | Namespace cho robot | `robot1` |

### Spawn từ file trực tiếp

```bash
ros2 run gazebo_ros spawn_entity.py \
  -file ~/ros2_ws/src/my_robot_description/urdf/robot.urdf \
  -entity my_robot \
  -x 1.0 -y 2.0 -z 0.1 \
  -Y 1.57
```

Cách này không cần `robot_state_publisher` chạy trước. Tuy nhiên, bạn vẫn nên chạy `robot_state_publisher` sau đó để publish TF.

### Spawn với namespace

Namespace giúp phân biệt topic khi có nhiều robot.

```bash
ros2 run gazebo_ros spawn_entity.py \
  -topic robot_description \
  -entity turtlebot1 \
  -robot_namespace turtlebot1 \
  -x 0 -y 0 -z 0.1
```

Topic sẽ trở thành `/turtlebot1/cmd_vel`, `/turtlebot1/odom`, v.v.

## 2. Spawn từ launch file

Thay vì gõ lệnh thủ công, bạn nên đưa spawn vào launch file.

```python
# launch/spawn_robot.launch.py
from launch import LaunchDescription
from launch.actions import ExecuteProcess
from launch_ros.actions import Node
from launch_ros.parameter_descriptions import ParameterValue
import os
import xacro

from ament_index_python.packages import get_package_share_directory

def generate_launch_description():
    pkg_desc = get_package_share_directory('my_robot_description')
    pkg_bringup = get_package_share_directory('my_robot_bringup')

    xacro_file = os.path.join(pkg_desc, 'urdf', 'robot.urdf.xacro')
    robot_description = xacro.process_file(xacro_file).toxml()

    world_file = os.path.join(pkg_bringup, 'worlds', 'empty.world')

    return LaunchDescription([
        # Gazebo
        ExecuteProcess(
            cmd=['gazebo', '--verbose', world_file,
                 '-s', 'libgazebo_ros_init.so',
                 '-s', 'libgazebo_ros_factory.so'],
            output='screen'
        ),

        # Robot State Publisher
        Node(
            package='robot_state_publisher',
            executable='robot_state_publisher',
            parameters=[{'robot_description': ParameterValue(robot_description, value_type=str)}]
        ),

        # Spawn Robot
        Node(
            package='gazebo_ros',
            executable='spawn_entity.py',
            arguments=['-topic', 'robot_description',
                       '-entity', 'my_robot',
                       '-x', '0.0', '-y', '0.0', '-z', '0.1'],
            output='screen'
        )
    ])
```

Chạy:

```bash
ros2 launch my_robot_bringup spawn_robot.launch.py
```

## 3. Spawn nhiều robot

Bạn có thể spawn nhiều robot bằng cách gọi `spawn_entity.py` nhiều lần với tên entity và namespace khác nhau.

```python
# launch/multi_robot.launch.py
from launch import LaunchDescription
from launch.actions import ExecuteProcess
from launch_ros.actions import Node
from launch_ros.parameter_descriptions import ParameterValue
import os
import xacro

from ament_index_python.packages import get_package_share_directory

def generate_launch_description():
    pkg_desc = get_package_share_directory('my_robot_description')
    xacro_file = os.path.join(pkg_desc, 'urdf', 'robot.urdf.xacro')
    robot_description = xacro.process_file(xacro_file).toxml()

    world_file = os.path.join(
        get_package_share_directory('my_robot_bringup'),
        'worlds', 'empty.world'
    )

    robots = [
        {'name': 'robot1', 'x': '0.0', 'y': '0.0', 'z': '0.1'},
        {'name': 'robot2', 'x': '2.0', 'y': '0.0', 'z': '0.1'},
        {'name': 'robot3', 'x': '0.0', 'y': '2.0', 'z': '0.1'},
    ]

    ld = LaunchDescription()

    # Gazebo
    ld.add_action(ExecuteProcess(
        cmd=['gazebo', '--verbose', world_file,
             '-s', 'libgazebo_ros_init.so',
             '-s', 'libgazebo_ros_factory.so'],
        output='screen'
    ))

    for robot in robots:
        # Robot State Publisher cho mỗi robot
        ld.add_action(Node(
            package='robot_state_publisher',
            executable='robot_state_publisher',
            namespace=robot['name'],
            parameters=[{
                'robot_description': ParameterValue(robot_description, value_type=str),
                'frame_prefix': robot['name'] + '/'
            }]
        ))

        # Spawn mỗi robot
        ld.add_action(Node(
            package='gazebo_ros',
            executable='spawn_entity.py',
            arguments=[
                '-topic', '/' + robot['name'] + '/robot_description',
                '-entity', robot['name'],
                '-robot_namespace', robot['name'],
                '-x', robot['x'], '-y', robot['y'], '-z', robot['z']
            ],
            output='screen'
        ))

    return ld
```

Lưu ý quan trọng khi spawn nhiều robot:

- Mỗi robot phải có `entity` name duy nhất.
- Dùng `namespace` để tránh xung đột topic.
- `frame_prefix` giúp TF của mỗi robot có prefix riêng.
- URDF nên dùng xacro với tham số `robot_name` để tạo link/joint name duy nhất.

Ví dụ xacro cho multi-robot:

```xml
<!-- urdf/robot.urdf.xacro -->
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro" name="$(arg robot_name)">
  <xacro:arg name="robot_name" default="robot"/>

  <link name="$(arg robot_name)/base_link">
    <!-- ... -->
  </link>

  <!-- Các link khác cũng prefix với robot_name -->
</robot>
```

## 4. Xóa và respawn robot

Xóa robot khỏi Gazebo:

```bash
ros2 service call /delete_entity gazebo_msgs/srv/DeleteEntity "{name: 'my_robot'}"
```

Respawn sau khi xóa:

```bash
ros2 run gazebo_ros spawn_entity.py -topic robot_description -entity my_robot -x 0 -y 0 -z 0.1
```

## 5. Tham khảo nhanh

```bash
# Spawn cơ bản
ros2 run gazebo_ros spawn_entity.py -topic robot_description -entity my_robot

# Spawn tại vị trí cụ thể
ros2 run gazebo_ros spawn_entity.py -topic robot_description -entity my_robot -x 1 -y 2 -z 0.1 -Y 1.57

# Spawn với namespace
ros2 run gazebo_ros spawn_entity.py -topic robot_description -entity bot1 -robot_namespace bot1

# Spawn từ file
ros2 run gazebo_ros spawn_entity.py -file robot.urdf -entity my_robot

# Xóa entity
ros2 service call /delete_entity gazebo_msgs/srv/DeleteEntity "{name: 'my_robot'}"
```

## 6. Tóm tắt

- `spawn_entity.py` là cầu nối giữa URDF/SDF và Gazebo world.
- Spawn từ topic là cách linh hoạt nhất vì xử lý xacro tự động.
- Namespace và `frame_prefix` giúp quản lý nhiều robot.
- Launch file tập trung toàn bộ quy trình spawn vào một file duy nhất.

---

**Bài tiếp theo:** [Bài 25: Sensor Simulation](../25.sensor-simulation/sensor-simulation.md)
