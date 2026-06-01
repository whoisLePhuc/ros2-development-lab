# Bài 23: Gazebo Sim — Mô phỏng vật lý

Gazebo là trình mô phỏng robot 3D mã nguồn mở. Nó tính toán va chạm, lực ma sát, ánh sáng, và cho phép bạn thử nghiệm robot trước khi chạy trên phần cứng thật. ROS2 Humble tích hợp Gazebo thông qua các package `gazebo_ros_pkgs`.

## 1. Gazebo là gì?

Gazebo bao gồm ba thành phần chính:

- **Physics engine**: ODE, Bullet, hoặc DART. Xử lý trọng lực, va chạm, ma sát.
- **Sensors**: LiDAR, camera, IMU, GPS, encoder... tất cả đều mô phỏng được.
- **World**: Môi trường 3D chứa mặt đất, vật cản, ánh sáng.

Bạn có thể kiểm tra thuật toán SLAM, điều khiển PID, hoặc tránh vật cản mà không cần robot thật.

## 2. Cài đặt

```bash
sudo apt update
sudo apt install ros-humble-gazebo-ros-pkgs ros-humble-gazebo-ros2-control
```

Kiểm tra cài đặt:

```bash
gazebo --version
```

## 3. Chạy Gazebo

### Cách 1: Chạy trực tiếp

```bash
gazebo --verbose
```

Flag `--verbose` in ra log chi tiết. Hữu ích khi debug plugin.

### Cách 2: Chạy từ launch file ROS2

```python
# launch/gazebo.launch.py
from launch import LaunchDescription
from launch.actions import ExecuteProcess
from launch_ros.actions import Node
from launch.substitutions import LaunchConfiguration
import os

from ament_index_python.packages import get_package_share_directory

def generate_launch_description():
    world_file = os.path.join(
        get_package_share_directory('my_robot_bringup'),
        'worlds',
        'empty.world'
    )

    return LaunchDescription([
        ExecuteProcess(
            cmd=['gazebo', '--verbose', world_file,
                 '-s', 'libgazebo_ros_init.so',
                 '-s', 'libgazebo_ros_factory.so'],
            output='screen'
        )
    ])
```

Hai plugin `libgazebo_ros_init.so` và `libgazebo_ros_factory.so` kết nối Gazebo với ROS2. Không có chúng, bạn không thể spawn robot từ ROS2.

## 4. URDF plugins cho Gazebo

Gazebo không hiểu URDF thuần túy. Bạn cần thêm các plugin SDF bên trong tag `<gazebo>`.

### 4.1. Differential Drive Controller

```xml
<gazebo>
  <plugin name="diff_drive" filename="libgazebo_ros_diff_drive.so">
    <ros>
      <namespace>/</namespace>
      <remapping>cmd_vel:=cmd_vel</remapping>
      <remapping>odom:=odom</remapping>
    </ros>

    <update_rate>50</update_rate>

    <!-- Wheels -->
    <left_joint>left_wheel_joint</left_joint>
    <right_joint>right_wheel_joint</right_joint>

    <!-- Kinematics -->
    <wheel_separation>0.45</wheel_separation>
    <wheel_diameter>0.2</wheel_diameter>

    <!-- Limits -->
    <max_wheel_torque>20</max_wheel_torque>
    <max_wheel_acceleration>1.0</max_wheel_acceleration>

    <!-- Output -->
    <publish_odom>true</publish_odom>
    <publish_odom_tf>true</publish_odom_tf>
    <publish_wheel_tf>false</publish_wheel_tf>

    <odometry_frame>odom</odometry_frame>
    <robot_base_frame>base_link</robot_base_frame>
  </plugin>
</gazebo>
```

Plugin này đăng ký topic `cmd_vel` (geometry_msgs/Twist) và xuất ra `odom` (nav_msgs/Odometry).

### 4.2. LiDAR (Ray Sensor)

```xml
<link name="lidar_link">
  <visual>
    <geometry>
      <cylinder radius="0.05" length="0.08"/>
    </geometry>
  </visual>
  <collision>
    <geometry>
      <cylinder radius="0.05" length="0.08"/>
    </geometry>
  </collision>
  <inertial>
    <mass>0.1</mass>
    <inertia ixx="0.0001" iyy="0.0001" izz="0.0001"/>
  </inertial>

  <sensor name="lidar_sensor" type="ray">
    <always_on>true</always_on>
    <visualize>true</visualize>
    <update_rate>10</update_rate>
    <ray>
      <scan>
        <horizontal>
          <samples>360</samples>
          <resolution>1</resolution>
          <min_angle>-3.14159</min_angle>
          <max_angle>3.14159</max_angle>
        </horizontal>
      </scan>
      <range>
        <min>0.12</min>
        <max>10.0</max>
        <resolution>0.015</resolution>
      </range>
    </ray>
    <plugin name="lidar_plugin" filename="libgazebo_ros_ray_sensor.so">
      <ros>
        <remapping>~/out:=scan</remapping>
      </ros>
      <output_type>sensor_msgs/LaserScan</output_type>
      <frame_name>lidar_link</frame_name>
    </plugin>
  </sensor>
</link>
```

LiDAR này quét 360 độ, 360 mẫu, tầm 10m. Topic output là `/scan`.

### 4.3. Camera

```xml
<link name="camera_link">
  <collision>
    <geometry>
      <box size="0.05 0.05 0.05"/>
    </geometry>
  </collision>
  <visual>
    <geometry>
      <box size="0.05 0.05 0.05"/>
    </geometry>
  </visual>
  <inertial>
    <mass>0.05</mass>
    <inertia ixx="0.00001" iyy="0.00001" izz="0.00001"/>
  </inertial>

  <sensor name="camera_sensor" type="camera">
    <camera>
      <horizontal_fov>1.047</horizontal_fov>
      <image>
        <width>640</width>
        <height>480</height>
        <format>R8G8B8</format>
      </image>
      <clip>
        <near>0.1</near>
        <far>100</far>
      </clip>
    </camera>
    <always_on>true</always_on>
    <update_rate>30</update_rate>
    <visualize>true</visualize>
    <plugin name="camera_plugin" filename="libgazebo_ros_camera.so">
      <ros>
        <remapping>~/image_raw:=camera/image_raw</remapping>
        <remapping>~/camera_info:=camera/camera_info</remapping>
      </ros>
      <frame_name>camera_link</frame_name>
      <hack_baseline>0.07</hack_baseline>
    </plugin>
  </sensor>
</link>
```

### 4.4. IMU

```xml
<link name="imu_link">
  <sensor name="imu_sensor" type="imu">
    <always_on>true</always_on>
    <update_rate>100</update_rate>
    <visualize>true</visualize>
    <plugin name="imu_plugin" filename="libgazebo_ros_imu_sensor.so">
      <ros>
        <remapping>~/out:=imu/data</remapping>
      </ros>
      <frame_name>imu_link</frame_name>
      <initial_orientation_as_reference>false</initial_orientation_as_reference>
    </plugin>
  </sensor>
</link>
```

## 5. World files (.world)

World file là file SDF (Simulation Description Format) định nghĩa môi trường.

```xml
<!-- worlds/empty.world -->
<?xml version="1.0" ?>
<sdf version="1.6">
  <world name="default">
    <scene>
      <ambient>0.4 0.4 0.4 1</ambient>
      <background>0.7 0.7 0.7 1</background>
      <shadows>true</shadows>
    </scene>

    <physics type="ode">
      <max_step_size>0.001</max_step_size>
      <real_time_factor>1</real_time_factor>
      <real_time_update_rate>1000</real_time_update_rate>
    </physics>

    <include>
      <uri>model://ground_plane</uri>
    </include>

    <include>
      <uri>model://sun</uri>
    </include>
  </world>
</sdf>
```

Bạn có thể thêm vật cản:

```xml
<include>
  <uri>model://house_1</uri>
  <pose>2 0 0 0 0 0</pose>
</include>
```

Gazebo cung cấp sẵn nhiều model trong thư viện online. Bạn cũng có thể tạo model riêng và đặt vào thư mục `~/.gazebo/models/`.

## 6. Launch file hoàn chỉnh: Gazebo + Robot

```python
# launch/robot_gazebo.launch.py
from launch import LaunchDescription
from launch.actions import ExecuteProcess, IncludeLaunchDescription
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.substitutions import LaunchConfiguration
from launch_ros.actions import Node
from launch_ros.parameter_descriptions import ParameterValue
import os
import xacro

from ament_index_python.packages import get_package_share_directory

def generate_launch_description():
    pkg_share = get_package_share_directory('my_robot_description')
    world_path = os.path.join(pkg_share, 'worlds', 'empty.world')
    xacro_file = os.path.join(pkg_share, 'urdf', 'robot.urdf.xacro')

    robot_description = xacro.process_file(xacro_file).toxml()

    return LaunchDescription([
        # Start Gazebo
        ExecuteProcess(
            cmd=['gazebo', '--verbose', world_path,
                 '-s', 'libgazebo_ros_init.so',
                 '-s', 'libgazebo_ros_factory.so'],
            output='screen'
        ),

        # Publish robot description
        Node(
            package='robot_state_publisher',
            executable='robot_state_publisher',
            name='robot_state_publisher',
            output='screen',
            parameters=[{'robot_description': ParameterValue(robot_description, value_type=str)}]
        ),

        # Spawn robot
        Node(
            package='gazebo_ros',
            executable='spawn_entity.py',
            arguments=['-topic', 'robot_description',
                       '-entity', 'my_robot',
                       '-x', '0', '-y', '0', '-z', '0.1'],
            output='screen'
        )
    ])
```

Chạy launch file:

```bash
ros2 launch my_robot_bringup robot_gazebo.launch.py
```

Bạn sẽ thấy Gazebo mở lên với robot đứng trên mặt đất. Kiểm tra các topic:

```bash
ros2 topic list
ros2 topic echo /scan
ros2 topic echo /camera/image_raw
```

## 7. Debug thường gặp

| Lỗi | Nguyên nhân | Cách fix |
|-----|-------------|----------|
| Robot rơi xuyên mặt đất | Inertial mass = 0 hoặc collision sai | Kiểm tra `<inertial>` và `<collision>` |
| Sensor không xuất topic | Thiếu plugin hoặc remapping sai | Kiểm tra filename plugin và topic remapping |
| Gazebo chậm | Physics step quá nhỏ hoặc nhiều sensor | Tăng `<max_step_size>`, giảm sample LiDAR |
| Model không load | Thiếu model trong `GAZEBO_MODEL_PATH` | Export path hoặc dùng `model://` URI |

## 8. Tóm tắt

- Gazebo mô phỏng vật lý và sensor trong môi trường 3D.
- Cài `ros-humble-gazebo-ros-pkgs` để kết nối với ROS2.
- URDF cần plugin SDF bên trong `<gazebo>` để hoạt động.
- World file định nghĩa môi trường bằng SDF.
- Launch file kết hợp Gazebo, robot_state_publisher, và spawn_entity.

---

**Bài tiếp theo:** [Bài 24: Spawn Robot — Đưa robot vào Gazebo](../24.spawn-robot/spawn-robot.md)
