# Bài 29: SLAM — Mapping & Localization

## 1. SLAM là gì?

SLAM (Simultaneous Localization and Mapping) là bài toán đồng thời xây dựng bản đồ môi trường và xác định vị trí robot trong bản đồ đó. Đây là một trong những bài toán cốt lõi của robotics di động.

Có hai chế độ chính:
- **Mapping**: Robot di chuyển trong môi trường chưa biết và xây dựng bản đồ.
- **Localization**: Robot đã có bản đồ, chỉ cần xác định vị trí của mình trên bản đồ đó.

## 2. Cài đặt SLAM Toolbox

```bash
sudo apt update
sudo apt install ros-humble-slam-toolbox
```

## 3. Chạy SLAM Toolbox

### 3.1. Chế độ Mapping (Async)

Tạo file cấu hình `slam_toolbox_params.yaml`:

```yaml
slam_toolbox:
  ros__parameters:
    use_sim_time: true
    mode: mapping          # mapping | localization
    debug_logging: false
    throttle_scans: 1
    transform_publish_period: 0.02
    map_update_interval: 5.0
    resolution: 0.05
    max_laser_range: 20.0    # tùy theo LIDAR
    minimum_travel_distance: 0.5
    minimum_travel_heading: 0.5
    scan_buffer_size: 10
    scan_buffer_maximum_scan_distance: 10.0
    link_match_minimum_response_fine: 0.1
    link_scan_maximum_distance: 1.5
    loop_search_maximum_distance: 3.0
    do_loop_closing: true
    loop_match_minimum_chain_size: 10
    loop_match_maximum_variance_coarse: 3.0
    loop_match_minimum_response_coarse: 0.35
    loop_match_minimum_response_fine: 0.45

    # TF frames
    odom_frame: odom
    map_frame: map
    base_frame: base_footprint
    scan_topic: /scan

    # If using GPS
    use_scan_matching: true
    use_scan_barycenter: true
```

Launch file `slam.launch.py`:

```python
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument
from launch.substitutions import LaunchConfiguration
from launch_ros.actions import Node


def generate_launch_description():
    use_sim_time = LaunchConfiguration('use_sim_time')

    declare_use_sim_time = DeclareLaunchArgument(
        'use_sim_time',
        default_value='true',
        description='Use simulation (Gazebo) clock if true'
    )

    slam_toolbox = Node(
        package='slam_toolbox',
        executable='async_slam_toolbox_node',
        name='slam_toolbox',
        output='screen',
        parameters=[
            {'use_sim_time': use_sim_time},
            '/path/to/slam_toolbox_params.yaml'
        ],
        remappings=[
            ('/scan', '/scan'),
        ]
    )

    return LaunchDescription([
        declare_use_sim_time,
        slam_toolbox,
    ])
```

Chạy:

```bash
ros2 launch your_package slam.launch.py
```

### 3.2. Hiển thị trên RViz2

Mở RViz2 và thêm các display:
- **Map**: Topic `/map` — hiển thị occupancy grid đang được xây dựng.
- **LaserScan**: Topic `/scan` — hiển thị dữ liệu LIDAR.
- **TF**: Hiển thị các frame `map`, `odom`, `base_footprint`.

Di chuyển robot trong Gazebo (hoặc thật), bạn sẽ thấy bản đồ được vẽ dần trong RViz2.

## 4. Lưu bản đồ

Sau khi mapping xong, lưu bản đồ bằng `map_saver_cli`:

```bash
ros2 run nav2_map_server map_saver_cli -f ~/my_map
```

Lệnh này tạo ra hai file:
- `my_map.pgm` — ảnh bản đồ (grayscale).
- `my_map.yaml` — metadata của bản đồ.

Nội dung file YAML:

```yaml
image: my_map.pgm
mode: trinary
resolution: 0.05
origin: [-10.0, -10.0, 0.0]
negate: 0
occupied_thresh: 0.65
free_thresh: 0.25
```

## 5. Chế độ Localization

Khi đã có bản đồ, chuyển sang chế độ localization bằng cách thay đổi tham số:

```yaml
slam_toolbox:
  ros__parameters:
    mode: localization
    map_file_name: /path/to/my_map.yaml
    map_start_pose: [0.0, 0.0, 0.0]  # [x, y, yaw]
```

Hoặc load bản đồ bằng `map_server`:

```bash
ros2 run nav2_map_server map_server --ros-args -p yaml_filename:=/path/to/my_map.yaml
```

## 6. Launch file hoàn chỉnh với Gazebo

```python
from launch import LaunchDescription
from launch.actions import IncludeLaunchDescription, DeclareLaunchArgument
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.substitutions import LaunchConfiguration, PathJoinSubstitution
from launch_ros.substitutions import FindPackageShare
from launch_ros.actions import Node


def generate_launch_description():
    pkg_share = FindPackageShare('your_package')
    use_sim_time = LaunchConfiguration('use_sim_time')

    declare_use_sim_time = DeclareLaunchArgument(
        'use_sim_time',
        default_value='true'
    )

    # Gazebo + robot
    gazebo_launch = IncludeLaunchDescription(
        PythonLaunchDescriptionSource([
            PathJoinSubstitution([pkg_share, 'launch', 'gazebo.launch.py'])
        ])
    )

    # SLAM Toolbox
    slam_toolbox = Node(
        package='slam_toolbox',
        executable='async_slam_toolbox_node',
        name='slam_toolbox',
        output='screen',
        parameters=[
            {'use_sim_time': use_sim_time},
            PathJoinSubstitution([pkg_share, 'config', 'slam_toolbox_params.yaml'])
        ]
    )

    # RViz2
    rviz2 = Node(
        package='rviz2',
        executable='rviz2',
        name='rviz2',
        arguments=['-d', PathJoinSubstitution([pkg_share, 'rviz', 'slam.rviz'])],
        parameters=[{'use_sim_time': use_sim_time}]
    )

    return LaunchDescription([
        declare_use_sim_time,
        gazebo_launch,
        slam_toolbox,
        rviz2,
    ])
```

## 7. Kiểm tra TF

```bash
ros2 run tf2_ros tf2_echo map base_footprint
```

Bạn sẽ thấy transform liên tục cập nhật vị trí robot trong frame `map`.

## 8. Tóm tắt

| Thành phần | Chức năng |
|------------|-----------|
| `async_slam_toolbox_node` | Chạy SLAM ở chế độ async (không đồng bộ với odometry) |
| `map_saver_cli` | Lưu bản đồ ra file `.pgm` và `.yaml` |
| `map_server` | Load bản đồ đã lưu để localization |
| RViz2 | Hiển thị bản đồ, laser scan, và TF |

---

**Bài tiếp theo:** [Bài 30: Navigation2](../30.navigation2/navigation2.md)
