# Bài 31: AMCL — Localization

## 1. AMCL là gì?

AMCL (Adaptive Monte Carlo Localization) là thuật toán xác định vị trí robot trên một bản đồ đã biết bằng bộ lọc hạt (particle filter). Nó kết hợp dữ liệu odometry và laser scan để ước lượng pose của robot trong frame `map`.

## 2. Cách hoạt động

1. **Phân phối hạt (Particles)**: AMCL duy trì một tập hợp các giả thuyết về vị trí robot (các hạt).
2. **Dự đoán (Prediction)**: Dựa trên lệnh di chuyển (odometry), các hạt được dịch chuyển.
3. **Cập nhật (Update)**: So sánh laser scan thực tế với laser scan dự đoán từ mỗi hạt, gán trọng số cho các hạt khớp tốt.
4. **Resampling**: Loại bỏ hạt có trọng số thấp, nhân bản hạt có trọng số cao.
5. **Adaptive**: Số lượng hạt tự động điều chỉnh dựa trên độ không chắc chắn của ước lượng.

## 3. Các tham số quan trọng

```yaml
amcl:
  ros__parameters:
    # Số hạt
    min_particles: 500          # Số hạt tối thiểu
    max_particles: 2000         # Số hạt tối đa

    # Ngưỡng cập nhật (để tránh cập nhật quá thường xuyên)
    update_min_d: 0.25          # Di chuyển tối thiểu (m) để cập nhật
    update_min_a: 0.2           # Xoay tối thiểu (rad) để cập nhật

    # Mô hình laser
    laser_model_type: "likelihood_field"   # "likelihood_field" hoặc "beam"
    laser_max_range: 100.0
    laser_min_range: -1.0

    # Mô hình odometry
    robot_model_type: "nav2_amcl::DifferentialMotionModel"
    odom_frame_id: "odom"
    base_frame_id: "base_footprint"
    global_frame_id: "map"

    # Pose ban đầu (nếu biết trước)
    set_initial_pose: true
    initial_pose:
      x: 0.0
      y: 0.0
      z: 0.0
      yaw: 0.0

    # Hệ số nhiễu odometry (càng cao = càng không tin tưởng odom)
    alpha1: 0.2     # nhiễu hướng từ rotation
    alpha2: 0.2     # nhiễu khoảng cách từ rotation
    alpha3: 0.2     # nhiễu khoảng cách từ translation
    alpha4: 0.2     # nhiễu hướng từ translation
    alpha5: 0.2     # nhiễu translation từ translation (omni only)

    # Các tham số khác
    transform_tolerance: 1.0
    resample_interval: 1
    recovery_alpha_slow: 0.001
    recovery_alpha_fast: 0.1
```

## 4. Chạy AMCL

AMCL thường được chạy cùng với `map_server` trong Nav2 bringup. Nếu chạy riêng:

```bash
# 1. Chạy map_server
ros2 run nav2_map_server map_server --ros-args \
  -p yaml_filename:=/path/to/map.yaml \
  -p use_sim_time:=true

# 2. Chạy AMCL
ros2 run nav2_amcl amcl --ros-args \
  --params-file /path/to/amcl_params.yaml \
  -p use_sim_time:=true

# 3. Lifecycle manager (nếu cần)
ros2 run nav2_util lifecycle_bringup map_server
ros2 run nav2_util lifecycle_bringup amcl
```

## 5. Đặt Initial Pose trong RViz2

Khi robot khởi động, AMCL cần biết vị trí xấp xỉ của robot trên bản đồ:

1. Mở RViz2.
2. Trong toolbar, chọn công cụ **2D Pose Estimate**.
3. Click và kéo trên bản đồ tại vị trí bạn nghĩ robot đang đứng.
4. AMCL sẽ phân tán các hạt xung quanh vị trí đó và bắt đầu hội tụ.

## 6. Kiểm tra từ CLI

### Xem particle cloud

```bash
ros2 topic echo /particlecloud
```

Mỗi message chứa danh sách các pose (hạt) với trọng số tương ứng.

### Xem pose ước lượng

```bash
ros2 topic echo /amcl_pose
```

Chứa pose đã được lọc (weighted average của các hạt) và covariance.

### Kiểm tra TF

```bash
ros2 run tf2_ros tf2_echo map odom
```

Hiển thị transform từ `map` đến `odom` mà AMCL xuất ra để sửa lỗi odometry drift.

### Xem danh sách topic

```bash
ros2 topic list | grep amcl
# /amcl_pose
# /particlecloud
```

## 7. Launch file tối giản

```python
from launch import LaunchDescription
from launch_ros.actions import Node
from launch.actions import ExecuteProcess


def generate_launch_description():
    map_file = '/path/to/map.yaml'

    map_server = Node(
        package='nav2_map_server',
        executable='map_server',
        name='map_server',
        output='screen',
        parameters=[
            {'yaml_filename': map_file},
            {'use_sim_time': True}
        ]
    )

    amcl = Node(
        package='nav2_amcl',
        executable='amcl',
        name='amcl',
        output='screen',
        parameters=[
            {'use_sim_time': True},
            {'min_particles': 500},
            {'max_particles': 2000},
            {'laser_model_type': 'likelihood_field'},
            {'update_min_d': 0.25},
            {'update_min_a': 0.2},
            {'set_initial_pose': True},
            {'initial_pose': {'x': 0.0, 'y': 0.0, 'yaw': 0.0}},
        ],
        remappings=[('/scan', '/scan')]
    )

    lifecycle = ExecuteProcess(
        cmd=['ros2', 'run', 'nav2_util', 'lifecycle_bringup', 'map_server'],
        output='screen'
    )

    lifecycle2 = ExecuteProcess(
        cmd=['ros2', 'run', 'nav2_util', 'lifecycle_bringup', 'amcl'],
        output='screen'
    )

    return LaunchDescription([
        map_server,
        amcl,
        lifecycle,
        lifecycle2,
    ])
```

## 8. Tài liệu tham khảo

- [Nav2 AMCL Documentation](https://navigation.ros.org/configuration/packages/configuring-amcl.html)
- [Probabilistic Robotics — Thrun, Burgard, Fox](http://www.probabilistic-robotics.org/)
- ROS Wiki: [amcl](http://wiki.ros.org/amcl)

## 9. Tóm tắt

| Topic/Param | Ý nghĩa |
|-------------|---------|
| `/amcl_pose` | Pose ước lượng của robot trong frame `map` |
| `/particlecloud` | Tập hợp các hạt (particles) biểu diễn phân phối pose |
| `min/max_particles` | Điều chỉnh độ chính xác vs. tính toán |
| `update_min_d/a` | Ngưỡng kích hoạt cập nhật filter |
| `laser_model_type` | Mô hình đo lường laser (likelihood_field phổ biến nhất) |

---

**Bài tiếp theo:** [Bài 32: PX4 Fundamentals](../32.px4-fundamentals/px4-fundamentals.md)
