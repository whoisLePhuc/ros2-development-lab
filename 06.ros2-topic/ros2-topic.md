# Bài 5: Topic — Publisher & Subscriber

## 1. Topic là gì?

**Topic** là kênh giao tiếp bất đồng bộ (asynchronous) trong ROS2, hoạt động theo mô hình **publish-subscribe**:

- Một hoặc nhiều **Publisher** gửi message lên topic
- Một hoặc nhiều **Subscriber** nhận message từ topic
- Publisher và Subscriber không biết về sự tồn tại của nhau (decoupled)

Điều này giúp hệ thống linh hoạt: bạn có thể thêm publisher hoặc subscriber mới mà không cần sửa code của các thành phần khác.

## 2. Message Types

ROS2 cung cấp nhiều kiểu message chuẩn:

| Package | Message | Mô tả |
|---------|---------|-------|
| `std_msgs` | `String`, `Float32`, `Int32`, `Bool` | Kiểu cơ bản |
| `geometry_msgs` | `Twist`, `Pose`, `Point` | Dữ liệu hình học |
| `sensor_msgs` | `LaserScan`, `Image`, `Imu` | Dữ liệu cảm biến |
| `nav_msgs` | `Odometry`, `Path` | Dữ liệu điều hướng |

Cài đặt package message:

```bash
sudo apt install ros-humble-std-msgs ros-humble-geometry-msgs ros-humble-sensor-msgs
```

## 3. Publisher — Gửi dữ liệu

### Ví dụ: Publisher gửi nhiệt độ

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import Float32
import random


class TemperaturePublisher(Node):
    def __init__(self):
        super().__init__('temperature_publisher')
        # Tạo publisher trên topic 'temperature' với queue size 10
        self.publisher_ = self.create_publisher(Float32, 'temperature', 10)
        self.timer = self.create_timer(1.0, self.publish_temperature)
        self.get_logger().info('Temperature Publisher đã khởi động')

    def publish_temperature(self):
        msg = Float32()
        msg.data = 25.0 + random.uniform(-2.0, 2.0)  # Nhiệt độ 23-27°C
        self.publisher_.publish(msg)
        self.get_logger().info(f'Published: {msg.data:.2f} °C')


def main(args=None):
    rclpy.init(args=args)
    node = TemperaturePublisher()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

### Giải thích

- `create_publisher(Float32, 'temperature', 10)`: Tạo publisher gửi message kiểu `Float32` lên topic `temperature`, queue size là 10
- `publish(msg)`: Gửi message ngay lập tức
- **Queue size**: Số message tối đa được lưu trong hàng đợi nếu subscriber không kịp xử lý

## 4. Subscriber — Nhận dữ liệu

### Ví dụ: Subscriber nhận nhiệt độ

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import Float32


class TemperatureSubscriber(Node):
    def __init__(self):
        super().__init__('temperature_subscriber')
        # Tạo subscriber lắng nghe topic 'temperature'
        self.subscription = self.create_subscription(
            Float32,
            'temperature',
            self.temperature_callback,
            10
        )
        self.get_logger().info('Temperature Subscriber đã khởi động')

    def temperature_callback(self, msg):
        self.get_logger().info(f'Received: {msg.data:.2f} °C')


def main(args=None):
    rclpy.init(args=args)
    node = TemperatureSubscriber()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

### Giải thích

- `create_subscription(Float32, 'temperature', callback, 10)`: Đăng ký nhận message kiểu `Float32` từ topic `temperature`
- `temperature_callback(self, msg)`: Hàm được gọi mỗi khi có message mới
- `msg.data`: Truy cập dữ liệu bên trong message

## 5. Ví dụ: Điều khiển robot với Twist

`geometry_msgs/Twist` dùng để gửi lệnh vận tốc cho robot:

```python
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist


class RobotController(Node):
    def __init__(self):
        super().__init__('robot_controller')
        self.publisher_ = self.create_publisher(Twist, 'cmd_vel', 10)
        self.timer = self.create_timer(0.1, self.send_command)
        self.get_logger().info('Robot Controller đã khởi động')

    def send_command(self):
        msg = Twist()
        msg.linear.x = 0.5   # Tiến về phía trước 0.5 m/s
        msg.angular.z = 0.2  # Xoay 0.2 rad/s
        self.publisher_.publish(msg)


def main(args=None):
    rclpy.init(args=args)
    node = RobotController()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

Cấu trúc `Twist`:
- `linear.x`: Vận tốc tuyến tính theo trục x (tiến/lùi)
- `linear.y`, `linear.z`: Vận tốc tuyến tính theo trục y, z
- `angular.x`, `angular.y`: Vận tốc góc theo trục x, y (roll, pitch)
- `angular.z`: Vận tốc góc theo trục z (yaw, xoay trái/phải)

## 6. Queue Size — Kích thước hàng đợi

Queue size quyết định số message được lưu tạm khi subscriber không kịp xử lý:

- **Queue size nhỏ (1-10)**: Phù hợp với dữ liệu real-time, message cũ sẽ bị bỏ nếu hàng đợi đầy
- **Queue size lớn (100-1000)**: Phù hợp với dữ liệu cần đảm bảo không mất, nhưng có độ trễ cao hơn

```python
# Real-time: chỉ quan tâm message mới nhất
self.create_publisher(Image, 'camera/image', 1)

# Đảm bảo không mất dữ liệu
self.create_publisher(String, 'log_data', 100)
```

## 7. CLI — Công cụ dòng lệnh cho Topic

```bash
# Liệt kê tất cả topic đang hoạt động
ros2 topic list

# Xem chi tiết một topic
ros2 topic info /temperature

# In message ra terminal (echo)
ros2 topic echo /temperature

# Kiểm tra tần suất publish (Hz)
ros2 topic hz /temperature

# Kiểm tra băng thông sử dụng
ros2 topic bw /temperature

# Publish message trực tiếp từ CLI
ros2 topic pub /temperature std_msgs/msg/Float32 '{data: 26.5}' --once

# Publish liên tục với tần suất 1Hz
ros2 topic pub /temperature std_msgs/msg/Float32 '{data: 26.5}' --rate 1
```

## 8. Tóm tắt

- **Topic** là kênh giao tiếp bất đồng bộ, một-to-nhiều, decoupled
- **Publisher** dùng `create_publisher()` và `publish()` để gửi message
- **Subscriber** dùng `create_subscription()` và callback để nhận message
- **Queue size** kiểm soát số message được lưu tạm
- `geometry_msgs/Twist` là message phổ biến để điều khiển robot
- CLI `ros2 topic` giúp debug và kiểm tra topic nhanh chóng

---

**Bài tiếp theo:** [Bài 6: Service — Request & Response](../07.ros2-service/ros2-service.md)
