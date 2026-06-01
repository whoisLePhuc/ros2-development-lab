# Bài 39: Multi-Robot

## 1. Vấn đề khi chạy nhiều robot

Khi bạn có nhiều robot chạy cùng một mạng ROS2, các topic có thể bị **xung đột tên**:

- Robot 1 publish `/cmd_vel` → cả Robot 2 cũng nhận được.
- Robot 1 publish `/scan` → Robot 2 tưởng là lidar của mình.

ROS2 cung cấp 3 cơ chế để tách biệt:
1. **Namespace**: Thêm tiền tố vào topic/node name.
2. **Domain ID**: Tách biệt hoàn toàn ở lớp DDS.
3. **TF Prefix**: Tách biệt hệ tọa độ.

## 2. Namespace cho topic separation

Namespace là cách đơn giản nhất để tách biệt topic. Thay vì `/cmd_vel`, bạn có `/robot1/cmd_vel`.

### Dùng namespace trong launch file

```python
from launch import LaunchDescription
from launch_ros.actions import Node

def generate_launch_description():
    return LaunchDescription([
        # Robot 1
        Node(
            package='turtlesim',
            executable='turtlesim_node',
            namespace='robot1',
            name='sim',
            output='screen'),
        Node(
            package='turtlesim',
            executable='turtle_teleop_key',
            namespace='robot1',
            name='teleop',
            output='screen'),

        # Robot 2
        Node(
            package='turtlesim',
            executable='turtlesim_node',
            namespace='robot2',
            name='sim',
            output='screen'),
        Node(
            package='turtlesim',
            executable='turtle_teleop_key',
            namespace='robot2',
            name='teleop',
            output='screen'),
    ])
```

### Kết quả

```bash
ros2 topic list
/robot1/cmd_vel
/robot1/pose
/robot2/cmd_vel
/robot2/pose
```

### Trong code C++

```cpp
// Node tự động thêm namespace từ launch
// Không cần thay đổi code, chỉ cần truyền namespace trong launch
auto node = std::make_shared<rclcpp::Node>("my_node");  // Tên gốc
```

Nếu launch với `namespace='robot1'`, node sẽ có tên `/robot1/my_node`.

### Remapping topic

Đôi khi bạn muốn một node trong namespace vẫn dùng topic global:

```python
Node(
    package='my_pkg',
    executable='my_node',
    namespace='robot1',
    remappings=[
        ('scan', '/global_scan'),  # /robot1/scan → /global_scan
        ('cmd_vel', 'cmd_vel'),   # Giữ nguyên /robot1/cmd_vel
    ]
)
```

## 3. Multi-robot launch file đầy đủ

Giả sử bạn có một robot với: lidar, camera, motor driver, navigation.

```python
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument
from launch.substitutions import LaunchConfiguration
from launch_ros.actions import Node

def generate_launch_description():
    # Argument để chọn robot ID
    robot_id_arg = DeclareLaunchArgument(
        'robot_id',
        default_value='robot1',
        description='ID của robot'
    )
    robot_id = LaunchConfiguration('robot_id')

    # Namespace dựa trên robot_id
    namespace = robot_id

    return LaunchDescription([
        robot_id_arg,

        # Lidar node
        Node(
            package='rplidar_ros',
            executable='rplidar_composition',
            namespace=namespace,
            name='lidar',
            output='screen',
            parameters=[{'serial_port': '/dev/ttyUSB0'}]
        ),

        # Camera node
        Node(
            package='usb_cam',
            executable='usb_cam_node_exe',
            namespace=namespace,
            name='camera',
            output='screen',
            parameters=[{'video_device': '/dev/video0'}]
        ),

        # Motor driver
        Node(
            package='diff_drive_controller',
            executable='diff_drive_controller',
            namespace=namespace,
            name='motor',
            output='screen',
            parameters=[{'wheel_separation': 0.3}]
        ),

        # Navigation (Nav2)
        Node(
            package='nav2_bringup',
            executable='navigation_launch.py',
            namespace=namespace,
            output='screen',
            arguments=['use_namespace:=true', ['namespace:=', namespace]]
        ),
    ])
```

Chạy:

```bash
# Robot 1
ros2 launch my_robot_bringup robot.launch.py robot_id:=robot1

# Robot 2
ros2 launch my_robot_bringup robot.launch.py robot_id:=robot2
```

## 4. Domain ID cho network isolation

Nếu bạn muốn tách biệt hoàn toàn ở lớp mạng (ví dụ: 10 robot trong nhà máy, mỗi robot là một hệ thống độc lập):

```bash
# Robot 1
export ROS_DOMAIN_ID=1
ros2 launch my_robot_bringup robot.launch.py

# Robot 2
export ROS_DOMAIN_ID=2
ros2 launch my_robot_bringup robot.launch.py
```

### Kết hợp Namespace + Domain ID

Trong hệ thống lớn, bạn có thể dùng cả hai:
- **Domain ID**: Tách biệt nhóm robot (ví dụ: Domain 1 = Khu vực A, Domain 2 = Khu vực B).
- **Namespace**: Tách biệt robot trong cùng khu vực.

```bash
# Khu vực A, Robot 1
export ROS_DOMAIN_ID=1
ros2 launch my_robot_bringup robot.launch.py robot_id:=robot1

# Khu vực A, Robot 2
export ROS_DOMAIN_ID=1
ros2 launch my_robot_bringup robot.launch.py robot_id:=robot2

# Khu vực B, Robot 3
export ROS_DOMAIN_ID=2
ros2 launch my_robot_bringup robot.launch.py robot_id:=robot3
```

## 5. TF2 với prefix

Khi nhiều robot cùng publish TF, frame name sẽ bị trùng (`base_link`, `laser`, `odom`).

### Thêm prefix vào frame name

```python
from launch import LaunchDescription
from launch_ros.actions import Node

def generate_launch_description():
    robot_id = 'robot1'

    return LaunchDescription([
        Node(
            package='robot_state_publisher',
            executable='robot_state_publisher',
            namespace=robot_id,
            parameters=[{
                'robot_description': '<robot name="' + robot_id + '">...</robot>',
                'frame_prefix': robot_id + '/',  # Thêm prefix vào tất cả frame
            }]
        ),
    ])
```

### Trong URDF

```xml
<robot name="my_robot">
  <link name="base_link"/>
  <link name="laser"/>
  <joint name="laser_joint" type="fixed">
    <parent link="base_link"/>
    <child link="laser"/>
    <origin xyz="0.2 0 0.1"/>
  </joint>
</robot>
```

Với `frame_prefix='robot1/'`, TF tree sẽ là:
- `robot1/base_link`
- `robot1/laser`
- `robot1/odom`

### Xem TF tree

```bash
ros2 run tf2_tools view_frames
# Mở frames.pdf để xem cây TF
```

## 6. Multi-robot coordination example

Giả sử bạn muốn 2 robot di chuyển theo đội hình (formation control).

### Central coordinator node

```python
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist

class FormationController(Node):
    def __init__(self):
        super().__init__('formation_controller')

        # Publisher cho từng robot
        self.pub1 = self.create_publisher(Twist, '/robot1/cmd_vel', 10)
        self.pub2 = self.create_publisher(Twist, '/robot2/cmd_vel', 10)

        # Timer điều khiển đội hình
        self.timer = self.create_timer(0.1, self.control_loop)
        self.t = 0.0

    def control_loop(self):
        self.t += 0.1

        # Robot 1: đi thẳng
        cmd1 = Twist()
        cmd1.linear.x = 0.5
        self.pub1.publish(cmd1)

        # Robot 2: đi thẳng + offset
        cmd2 = Twist()
        cmd2.linear.x = 0.5
        cmd2.linear.y = 0.3 * (self.t % 10 < 5)  # Zigzag
        self.pub2.publish(cmd2)

def main():
    rclpy.init()
    node = FormationController()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

### Decentralized coordination (không có master)

Mỗi robot tự quyết định dựa trên vị trí robot khác:

```python
# Trên Robot 1
class DecentralizedNode(Node):
    def __init__(self):
        super().__init__('robot1_controller')
        self.pub = self.create_publisher(Twist, 'cmd_vel', 10)
        self.sub = self.create_subscription(
            Pose, '/robot2/pose', self.robot2_pose_cb, 10)

    def robot2_pose_cb(self, msg):
        # Tính toán dựa trên vị trí robot 2
        cmd = Twist()
        cmd.linear.x = 0.3
        self.pub.publish(cmd)
```

## 7. Tóm tắt

| Kỹ thuật | Mục đích | Phạm vi |
|----------|---------|---------|
| Namespace | Tách topic/node name | Trong cùng domain |
| Domain ID | Tách hoàn toàn ở lớp DDS | Khác domain = không thấy nhau |
| TF Prefix | Tách frame name trong TF tree | Trong cùng domain |
| Remapping | Đổi tên topic cụ thể | Linh hoạt |

---

**Bài tiếp theo:** [Bài 40: MoveIt2](../40.moveit2/moveit2.md)
