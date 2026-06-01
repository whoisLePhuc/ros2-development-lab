# Bài 4: ROS2 Node

## 1. Node là gì?

Trong ROS2, **Node** là đơn vị thực thi cơ bản. Mỗi node thực hiện một nhiệm vụ cụ thể, ví dụ: điều khiển động cơ, xử lý ảnh camera, hoặc lập kế hoạch đường đi. Các node giao tiếp với nhau thông qua Topic, Service, Action và Parameter.

Một chương trình ROS2 thường có nhiều node chạy song song, mỗi node độc lập và có thể khởi động, dừng lại mà không ảnh hưởng đến các node khác.

## 2. Cấu trúc một Node Python

Một node ROS2 viết bằng Python cần:

1. Import thư viện `rclpy`
2. Tạo class kế thừa từ `rclpy.node.Node`
3. Gọi `super().__init__()` để đặt tên node
4. Khởi tạo, spin và shutdown node trong `main()`

### Ví dụ cơ bản: Minimal Node

```python
import rclpy
from rclpy.node import Node


class MinimalNode(Node):
    def __init__(self):
        super().__init__('minimal_node')
        self.get_logger().info('Node đã khởi động!')


def main(args=None):
    rclpy.init(args=args)
    node = MinimalNode()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

### Giải thích từng phần

| Dòng | Ý nghĩa |
|------|---------|
| `import rclpy` | Import thư viện client Python của ROS2 |
| `from rclpy.node import Node` | Import class Node cơ sở |
| `super().__init__('minimal_node')` | Đặt tên node là `minimal_node` |
| `rclpy.init(args=args)` | Khởi tạo thư viện rclpy |
| `rclpy.spin(node)` | Giữ node chạy và xử lý callback |
| `rclpy.shutdown()` | Giải phóng tài nguyên khi kết thúc |

## 3. Timer — Thực thi định kỳ

Timer cho phép node thực hiện một hàm lặp lại theo chu kỳ thời gian.

```python
import rclpy
from rclpy.node import Node


class TimerNode(Node):
    def __init__(self):
        super().__init__('timer_node')
        self.counter = 0
        # Tạo timer chạy mỗi 1.0 giây
        self.timer = self.create_timer(1.0, self.timer_callback)

    def timer_callback(self):
        self.counter += 1
        self.get_logger().info(f'Counter: {self.counter}')


def main(args=None):
    rclpy.init(args=args)
    node = TimerNode()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

`create_timer(period_sec, callback)` nhận hai tham số:
- `period_sec`: Chu kỳ lặp lại (giây)
- `callback`: Hàm được gọi mỗi chu kỳ

## 4. Logger — Ghi log

ROS2 cung cấp logger tích hợp với nhiều mức độ: `debug`, `info`, `warn`, `error`, `fatal`.

```python
self.get_logger().debug('Thông tin gỡ lỗi chi tiết')
self.get_logger().info('Thông báo thông thường')
self.get_logger().warn('Cảnh báo')
self.get_logger().error('Lỗi xảy ra')
self.get_logger().fatal('Lỗi nghiêm trọng')
```

Xem log từ terminal:

```bash
ros2 run <package_name> <node_name>
```

Hoặc xem tất cả log:

```bash
ros2 topic echo /rosout
```

## 5. Namespace — Nhóm Node

Namespace giúp tổ chức node theo nhóm, tránh xung đột tên khi chạy nhiều robot.

Chạy node với namespace:

```bash
ros2 run my_package my_node --ros-args --remap __ns:=/robot_1
```

Trong code, tên topic sẽ tự động thêm namespace:
- Topic `/sensor/data` sẽ thành `/robot_1/sensor/data`

Bạn cũng có thể đặt namespace trong launch file:

```python
from launch import LaunchDescription
from launch_ros.actions import Node

def generate_launch_description():
    return LaunchDescription([
        Node(
            package='my_package',
            executable='my_node',
            namespace='robot_1',
            name='my_node'
        )
    ])
```

## 6. Ví dụ hoàn chỉnh: Heartbeat Node

Node gửi heartbeat mỗi 2 giây, có thể dừng bằng Ctrl+C một cách sạch sẽ.

```python
import rclpy
from rclpy.node import Node


class HeartbeatNode(Node):
    def __init__(self):
        super().__init__('heartbeat_node')
        self.timer = self.create_timer(2.0, self.send_heartbeat)
        self.get_logger().info('Heartbeat node đã sẵn sàng')

    def send_heartbeat(self):
        self.get_logger().info('Heartbeat: beep')


def main(args=None):
    rclpy.init(args=args)
    node = HeartbeatNode()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        node.get_logger().info('Node đã dừng bởi người dùng')
    finally:
        node.destroy_node()
        rclpy.shutdown()


if __name__ == '__main__':
    main()
```

### Chạy ví dụ

```bash
# Terminal 1: Chạy node
ros2 run my_package heartbeat_node

# Terminal 2: Liệt kê các node đang chạy
ros2 node list

# Xem thông tin node
ros2 node info /heartbeat_node
```

## 7. Tóm tắt

- **Node** là đơn vị thực thi cơ bản trong ROS2
- Kế thừa từ `rclpy.node.Node` và gọi `super().__init__('node_name')`
- `rclpy.init()` khởi tạo, `rclpy.spin()` giữ node chạy, `rclpy.shutdown()` dọn dẹp
- **Timer** dùng `create_timer(period, callback)` để lặp định kỳ
- **Logger** dùng `get_logger().info/warn/error()` để ghi log
- **Namespace** giúp tổ chức và phân biệt các nhóm node

---

**Bài tiếp theo:** [Bài 5: Topic — Publisher & Subscriber](../06.ros2-topic/ros2-topic.md)
