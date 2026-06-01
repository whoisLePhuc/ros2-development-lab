# Bài 19: RViz2 — 3D Visualization

RViz2 là công cụ visualization 3D chính thức của ROS2. Nó giúp bạn nhìn thấy dữ liệu robot theo thời gian thực: laser scan, point cloud, camera image, TF frames, và nhiều hơn nữa.

## 1. Khởi động RViz2

```bash
# Cách đơn giản nhất
rviz2

# Hoặc mở với config có sẵn
rviz2 -d my_config.rviz
```

## 2. Giao diện chính

RViz2 có 3 khu vực chính:

- **3D View**: Khung nhìn chính, hiển thị mọi thứ trong không gian 3D
- **Displays Panel**: Bên trái, quản lý các display đang bật
- **Add Button**: Nút để thêm display mới

## 3. Fixed Frame

Fixed Frame là frame gốc mà mọi thứ được vẽ tương đối theo nó. Thường chọn:

- `map`: Nếu bạn muốn xem robot di chuyển trong bản đồ
- `odom`: Nếu muốn xem theo điểm khởi đầu của robot
- `base_link`: Nếu muốn camera "đứng yên" và thế giới xoay quanh robot

Đặt Fixed Frame ở mục **Global Options > Fixed Frame** trong Displays panel.

## 4. Các loại Display phổ biến

### 4.1. RobotModel

Hiển thị mô hình robot từ URDF.

- **Topic**: Tự động lấy từ `/robot_description`
- **Alpha**: Độ trong suốt (0.0 - 1.0)
- Dùng để kiểm tra URDF và xem robot trong 3D

### 4.2. LaserScan

Hiển thị dữ liệu từ LiDAR 2D.

- **Topic**: `/scan`
- **Color Transformer**: `Intensity` hoặc `Flat Color`
- **Style**: `Points`, `Squares`, hoặc `Flat Squares`
- **Size (Pixels)**: Kích thước điểm

### 4.3. PointCloud2

Hiển thị point cloud 3D (từ LiDAR 3D hoặc depth camera).

- **Topic**: `/velodyne_points` hoặc `/camera/depth/color/points`
- **Color Transformer**: `RGB8` (nếu có màu), `Intensity`, `AxisColor`
- **Decay Time**: Giữ point cloud bao lâu trên màn hình

### 4.4. Image

Hiển thị ảnh từ camera.

- **Topic**: `/camera/image_raw`
- **Transport Hint**: `raw` hoặc `compressed`
- Image sẽ hiện ở panel riêng, không trong 3D view

### 4.5. TF

Hiển thị tất cả TF frames dưới dạng trục tọa độ.

- **Show Names**: Hiện tên frame
- **Show Arrows**: Hiện mũi tên trục
- **Marker Scale**: Kích thước trục tọa độ

### 4.6. Marker

Hiển thị các marker tùy chỉnh do node publish.

- **Topic**: `/visualization_marker`
- Hỗ trợ nhiều hình dạng: cube, sphere, cylinder, arrow, text, line strip

## 5. Publish Marker từ code

Bạn có thể vẽ hình, text, hoặc đường line trong RViz2 bằng cách publish `visualization_msgs/Marker`.

### 5.1. Python

```python
import rclpy
from rclpy.node import Node
from visualization_msgs.msg import Marker
from geometry_msgs.msg import Point

class MarkerPublisher(Node):
    def __init__(self):
        super().__init__('marker_publisher')
        self.pub = self.create_publisher(Marker, '/visualization_marker', 10)
        self.timer = self.create_timer(1.0, self.publish_marker)

    def publish_marker(self):
        marker = Marker()
        marker.header.frame_id = 'map'
        marker.header.stamp = self.get_clock().now().to_msg()
        marker.ns = 'my_markers'
        marker.id = 0
        marker.type = Marker.SPHERE
        marker.action = Marker.ADD

        # Vị trí
        marker.pose.position.x = 1.0
        marker.pose.position.y = 2.0
        marker.pose.position.z = 0.0
        marker.pose.orientation.w = 1.0

        # Kích thước
        marker.scale.x = 0.5
        marker.scale.y = 0.5
        marker.scale.z = 0.5

        # Màu (RGBA)
        marker.color.r = 1.0
        marker.color.g = 0.0
        marker.color.b = 0.0
        marker.color.a = 1.0

        self.pub.publish(marker)
        self.get_logger().info('Published marker')

def main(args=None):
    rclpy.init(args=args)
    node = MarkerPublisher()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

### 5.2. Các loại marker

| Type | Giá trị | Mô tả |
|------|---------|-------|
| ARROW | 0 | Mũi tên |
| CUBE | 1 | Hình hộp |
| SPHERE | 2 | Hình cầu |
| CYLINDER | 3 | Hình trụ |
| LINE_STRIP | 4 | Đường nối nhiều điểm |
| LINE_LIST | 5 | Nhiều đoạn thẳng |
| TEXT_VIEW_FACING | 9 | Text luôn hướng về camera |

Ví dụ vẽ đường line:

```python
marker.type = Marker.LINE_STRIP
marker.points = [
    Point(x=0.0, y=0.0, z=0.0),
    Point(x=1.0, y=1.0, z=0.0),
    Point(x=2.0, y=0.0, z=0.0),
]
marker.scale.x = 0.1  # Độ dày đường
```

## 6. Lưu và tải config

### 6.1. Lưu config

File > Save Config As > `my_config.rviz`

Config lưu:

- Danh sách display đang bật
- Topic cho mỗi display
- Fixed Frame
- Góc nhìn camera
- Màu sắc, kích thước

### 6.2. Tải config

File > Open Config > chọn file `.rviz`

Hoặc dòng lệnh:

```bash
rviz2 -d my_config.rviz
```

## 7. Khởi động RViz2 từ Launch File

Thêm vào launch file để RViz2 tự mở khi launch:

```python
from launch import LaunchDescription
from launch_ros.actions import Node
from launch.substitutions import PathJoinSubstitution
from launch_ros.substitutions import FindPackageShare

def generate_launch_description():
    rviz_config = PathJoinSubstitution([
        FindPackageShare('my_pkg'),
        'rviz',
        'robot_view.rviz'
    ])

    return LaunchDescription([
        Node(
            package='rviz2',
            executable='rviz2',
            name='rviz2',
            arguments=['-d', rviz_config],
            output='screen'
        ),
        # ... các node khác
    ])
```

## Tóm tắt

- RViz2 là công cụ visualization 3D chính của ROS2
- Fixed Frame quyết định gốc tọa độ của 3D view
- Các display phổ biến: RobotModel, LaserScan, PointCloud2, Image, TF, Marker
- Publish `visualization_msgs/Marker` để vẽ hình tùy chỉnh
- Lưu config `.rviz` để tái sử dụng giao diện
- Khởi động RViz2 từ launch file bằng Node từ package `rviz2`

---

**Bài tiếp theo:** [Bài 20: RQT — ROS GUI Tools](../20.rqt/rqt.md)
