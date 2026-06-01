# Bài 26: ros2_control

`ros2_control` là framework điều khiển robot trong ROS2. Nó tách biệt phần điều khiển (controller) khỏi phần cứng (hardware) thông qua một lớp trung gian gọi là Hardware Interface. Điều này cho phép bạn chạy cùng một controller trên cả mô phỏng Gazebo và robot thật mà không cần đổi code.

## 1. Kiến trúc ros2_control

Ba thành phần chính:

1. **Controller Manager**: Quản lý vòng đời của các controller. Load, start, stop, và switch giữa các controller.
2. **Hardware Interface**: Lớp trung gian giữa Controller Manager và robot thật/mô phỏng. Đọc trạng thái joint, ghi lệnh điều khiển.
3. **Robot**: Phần cứng thật (motor, encoder) hoặc mô phỏng Gazebo.

Luồng dữ liệu:

```
Controller Manager
      |
      v
Hardware Interface  <--->  Robot (real or Gazebo)
      |
      v
Controllers (diff_drive, joint_state_broadcaster, v.v.)
```

## 2. Cài đặt

```bash
sudo apt install ros-humble-ros2-control ros-humble-ros2-controllers ros-humble-gazebo-ros2-control
```

## 3. URDF config với `<ros2_control>`

Tag `<ros2_control>` trong URDF định nghĩa các joint, interface type, và plugin hardware.

```xml
<!-- URDF snippet -->
<ros2_control name="GazeboSystem" type="system">
  <hardware>
    <plugin>gazebo_ros2_control/GazeboSystem</plugin>
  </hardware>

  <joint name="left_wheel_joint">
    <command_interface name="velocity">
      <param name="min">-10</param>
      <param name="max">10</param>
    </command_interface>
    <state_interface name="position"/>
    <state_interface name="velocity"/>
  </joint>

  <joint name="right_wheel_joint">
    <command_interface name="velocity">
      <param name="min">-10</param>
      <param name="max">10</param>
    </command_interface>
    <state_interface name="position"/>
    <state_interface name="velocity"/>
  </joint>
</ros2_control>
```

Giải thích:

- `<hardware><plugin>`: Plugin kết nối với backend. `gazebo_ros2_control/GazeboSystem` cho Gazebo. Khi chạy robot thật, bạn đổi thành plugin custom của mình.
- `<command_interface>`: Cách điều khiển joint. Có thể là `position`, `velocity`, hoặc `effort`.
- `<state_interface>`: Trạng thái đọc về từ joint. Thường là `position` và `velocity`.

### Plugin cho robot thật

Khi không dùng Gazebo, bạn viết plugin C++ kế thừa `hardware_interface::SystemInterface`:

```xml
<hardware>
  <plugin>my_robot_hardware/MyRobotHardware</plugin>
  <param name="serial_port">/dev/ttyUSB0</param>
  <param name="baudrate">115200</param>
</hardware>
```

Bài 27 sẽ hướng dẫn chi tiết cách viết plugin này.

## 4. Controller YAML config

File YAML định nghĩa các controller và tham số của chúng.

```yaml
# config/controllers.yaml
controller_manager:
  ros__parameters:
    update_rate: 50  # Hz

    joint_state_broadcaster:
      type: joint_state_broadcaster/JointStateBroadcaster

    diff_drive_controller:
      type: diff_drive_controller/DiffDriveController

diff_drive_controller:
  ros__parameters:
    left_wheel_names: ["left_wheel_joint"]
    right_wheel_names: ["right_wheel_joint"]

    wheel_separation: 0.45
    wheel_radius: 0.1

    wheel_separation_multiplier: 1.0
    left_wheel_radius_multiplier: 1.0
    right_wheel_radius_multiplier: 1.0

    publish_rate: 50.0
    odom_frame_id: "odom"
    base_frame_id: "base_link"
    pose_covariance_diagonal: [0.001, 0.001, 0.0, 0.0, 0.0, 0.01]
    twist_covariance_diagonal: [0.001, 0.0, 0.0, 0.0, 0.0, 0.01]

    open_loop: false
    enable_odom_tf: true

    cmd_vel_timeout: 0.5
    publish_limited_velocity: false

    linear:
      x:
        has_velocity_limits: true
        max_velocity: 1.0
        min_velocity: -1.0
        has_acceleration_limits: true
        max_acceleration: 0.5
        min_acceleration: -0.5

    angular:
      z:
        has_velocity_limits: true
        max_velocity: 1.0
        min_velocity: -1.0
        has_acceleration_limits: true
        max_acceleration: 0.5
        min_acceleration: -0.5
```

### Các controller phổ biến

| Controller | Type | Chức năng |
|------------|------|-----------|
| `joint_state_broadcaster` | `joint_state_broadcaster/JointStateBroadcaster` | Publish `/joint_states` từ hardware |
| `diff_drive_controller` | `diff_drive_controller/DiffDriveController` | Điều khiển differential drive, publish odom |
| `joint_trajectory_controller` | `joint_trajectory_controller/JointTrajectoryController` | Điều khiển theo quỹ đạo (cho arm) |
| `effort_controllers/JointGroupEffortController` | `effort_controllers/JointGroupEffortController` | Ghi lệnh effort (torque) |
| `position_controllers/JointGroupPositionController` | `position_controllers/JointGroupPositionController` | Ghi lệnh position |
| `velocity_controllers/JointGroupVelocityController` | `velocity_controllers/JointGroupVelocityController` | Ghi lệnh velocity |

## 5. Launch file

```python
# launch/robot_control.launch.py
from launch import LaunchDescription
from launch.actions import IncludeLaunchDescription, RegisterEventHandler
from launch.event_handlers import OnProcessExit
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.substitutions import Command, FindExecutable, PathJoinSubstitution
from launch_ros.actions import Node
from launch_ros.substitutions import FindPackageShare

from ament_index_python.packages import get_package_share_directory
import os

def generate_launch_description():
    pkg_share = get_package_share_directory('my_robot_bringup')

    # URDF
    robot_description = Command([
        PathJoinSubstitution([FindExecutable(name='xacro')]),
        ' ',
        PathJoinSubstitution([
            FindPackageShare('my_robot_description'),
            'urdf', 'robot.urdf.xacro'
        ])
    ])

    # Controller config
    controller_params = os.path.join(pkg_share, 'config', 'controllers.yaml')

    # Robot State Publisher
    robot_state_pub = Node(
        package='robot_state_publisher',
        executable='robot_state_publisher',
        parameters=[{'robot_description': robot_description}]
    )

    # Controller Manager
    controller_manager = Node(
        package='controller_manager',
        executable='ros2_control_node',
        parameters=[{'robot_description': robot_description}, controller_params],
        output='screen'
    )

    # Spawn joint_state_broadcaster
    jsb_spawner = Node(
        package='controller_manager',
        executable='spawner',
        arguments=['joint_state_broadcaster', '--controller-manager', '/controller_manager'],
        output='screen'
    )

    # Spawn diff_drive_controller (chỉ sau khi jsb đã load)
    diff_drive_spawner = Node(
        package='controller_manager',
        executable='spawner',
        arguments=['diff_drive_controller', '--controller-manager', '/controller_manager'],
        output='screen'
    )

    # Đảm bảo diff_drive load sau joint_state_broadcaster
    delay_diff_drive = RegisterEventHandler(
        event_handler=OnProcessExit(
            target_action=jsb_spawner,
            on_exit=[diff_drive_spawner]
        )
    )

    return LaunchDescription([
        robot_state_pub,
        controller_manager,
        jsb_spawner,
        delay_diff_drive
    ])
```

Lưu ý: `ros2_control_node` cần URDF để biết cấu trúc robot. Bạn truyền `robot_description` vào parameters.

## 6. CLI ros2 control

### Liệt kê controller

```bash
ros2 control list_controllers
```

Output mẫu:

```
joint_state_broadcaster[joint_state_broadcaster/JointStateBroadcaster] active
diff_drive_controller[diff_drive_controller/DiffDriveController] active
```

### Liệt kê hardware interfaces

```bash
ros2 control list_hardware_interfaces
```

### Load và start controller

```bash
ros2 run controller_manager spawner joint_state_broadcaster
ros2 run controller_manager spawner diff_drive_controller
```

### Stop controller

```bash
ros2 run controller_manager unspawner diff_drive_controller
```

### Switch controller

```bash
ros2 control switch_controllers --start controller1 --stop controller2
```

### Kiểm tra trạng thái controller

```bash
ros2 control list_controllers --state active
ros2 control list_controllers --state inactive
```

## 7. Chạy với Gazebo

Khi dùng Gazebo, bạn không cần chạy `ros2_control_node` riêng. Plugin `gazebo_ros2_control` tự động khởi tạo Controller Manager.

```xml
<gazebo>
  <plugin name="gazebo_ros2_control" filename="libgazebo_ros2_control.so">
    <parameters>$(find my_robot_bringup)/config/controllers.yaml</parameters>
  </plugin>
</gazebo>
```

Launch file cho Gazebo + ros2_control:

```python
# launch/gazebo_control.launch.py (tóm tắt)
# 1. Chạy Gazebo với world
# 2. Spawn robot
# 3. robot_state_publisher
# 4. Controller spawners (jsb, diff_drive)
# Không cần ros2_control_node vì Gazebo plugin đã tạo sẵn.
```

## 8. Tóm tắt

- `ros2_control` tách controller khỏi hardware qua Hardware Interface.
- URDF dùng `<ros2_control>` để định nghĩa joint và interface.
- YAML config định nghĩa controller và tham số.
- Controller Manager quản lý vòng đời controller.
- `spawner` CLI load và start controller. `unspawner` dừng.
- Gazebo có plugin riêng (`libgazebo_ros2_control.so`) tích hợp sẵn Controller Manager.

---

**Bài tiếp theo:** [Bài 27: Hardware Interface](../27.hardware-interface/hardware-interface.md)
