# Bài 6: Service — Request & Response

## 1. Service là gì?

**Service** là mô hình giao tiếp **một-một, đồng bộ** (synchronous) trong ROS2:

- Một **Service Server** cung cấp một chức năng xử lý request
- Một **Service Client** gửi request và chờ nhận response
- Client phải đợi Server xử lý xong mới tiếp tục (blocking) hoặc dùng async

Service phù hợp cho các tác vụ ngắn, cần phản hồi ngay lập tức, ví dụ: tính toán, lấy trạng thái, reset hệ thống.

## 2. Cấu trúc file .srv

Service được định nghĩa bởi file `.srv` với hai phần ngăn cách bởi `---`:

```srv
# Request
int64 a
int64 b
---
# Response
int64 sum
```

Ví dụ khác với `std_srvs/srv/Empty.srv`:

```srv
---
```

Không có request và response data, chỉ dùng để kích hoạt một hành động.

## 3. Service Server — Xử lý request

### Ví dụ: AddTwoInts Server

```python
import rclpy
from rclpy.node import Node
from example_interfaces.srv import AddTwoInts


class AddTwoIntsServer(Node):
    def __init__(self):
        super().__init__('add_two_ints_server')
        self.srv = self.create_service(AddTwoInts, 'add_two_ints', self.add_callback)
        self.get_logger().info('AddTwoInts Server đã sẵn sàng')

    def add_callback(self, request, response):
        response.sum = request.a + request.b
        self.get_logger().info(f'Nhận request: {request.a} + {request.b} = {response.sum}')
        return response


def main(args=None):
    rclpy.init(args=args)
    node = AddTwoIntsServer()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

### Giải thích

- `create_service(AddTwoInts, 'add_two_ints', callback)`: Tạo server với kiểu service `AddTwoInts`, tên service `add_two_ints`
- `add_callback(self, request, response)`: Nhận request, điền dữ liệu vào response, trả về response

## 4. Service Client — Gửi request

### Ví dụ: AddTwoInts Client

```python
import rclpy
from rclpy.node import Node
from example_interfaces.srv import AddTwoInts
import sys


class AddTwoIntsClient(Node):
    def __init__(self):
        super().__init__('add_two_ints_client')
        self.client = self.create_client(AddTwoInts, 'add_two_ints')
        # Chờ server sẵn sàng
        while not self.client.wait_for_service(timeout_sec=1.0):
            self.get_logger().info('Đang chờ server...')

    def send_request(self, a, b):
        request = AddTwoInts.Request()
        request.a = a
        request.b = b
        self.future = self.client.call_async(request)
        return self.future


def main(args=None):
    rclpy.init(args=args)
    node = AddTwoIntsClient()

    if len(sys.argv) != 3:
        node.get_logger().info('Usage: python3 client.py <a> <b>')
        return

    future = node.send_request(int(sys.argv[1]), int(sys.argv[2]))

    rclpy.spin_until_future_complete(node, future)

    response = future.result()
    node.get_logger().info(f'Kết quả: {response.sum}')

    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

### Giải thích

- `create_client(AddTwoInts, 'add_two_ints')`: Tạo client kết nối đến service
- `wait_for_service(timeout_sec=1.0)`: Chờ server khởi động
- `call_async(request)`: Gửi request không đồng bộ, trả về `future`
- `spin_until_future_complete(node, future)`: Giữ node chạy cho đến khi nhận được response

## 5. Ví dụ: Robot Reset với Empty Service

Dùng `std_srvs/srv/Empty` để gửi lệnh reset robot không cần tham số.

### Server

```python
import rclpy
from rclpy.node import Node
from std_srvs.srv import Empty


class RobotResetServer(Node):
    def __init__(self):
        super().__init__('robot_reset_server')
        self.srv = self.create_service(Empty, 'robot_reset', self.reset_callback)
        self.get_logger().info('Robot Reset Server đã sẵn sàng')

    def reset_callback(self, request, response):
        self.get_logger().info('Robot đang reset...')
        # Thực hiện reset ở đây
        self.get_logger().info('Robot đã reset xong!')
        return response


def main(args=None):
    rclpy.init(args=args)
    node = RobotResetServer()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

### Client

```python
import rclpy
from rclpy.node import Node
from std_srvs.srv import Empty


class RobotResetClient(Node):
    def __init__(self):
        super().__init__('robot_reset_client')
        self.client = self.create_client(Empty, 'robot_reset')
        while not self.client.wait_for_service(timeout_sec=1.0):
            self.get_logger().info('Đang chờ server...')

    def send_reset(self):
        request = Empty.Request()
        self.future = self.client.call_async(request)
        return self.future


def main(args=None):
    rclpy.init(args=args)
    node = RobotResetClient()
    future = node.send_reset()
    rclpy.spin_until_future_complete(node, future)
    node.get_logger().info('Reset hoàn tất')
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

## 6. So sánh Topic và Service

| Đặc điểm | Topic | Service |
|----------|-------|---------|
| Mô hình | Một-to-nhiều | Một-một |
| Đồng bộ | Không (async) | Có (sync hoặc async) |
| Phù hợp | Dữ liệu liên tục (sensor, camera) | Tác vụ ngắn, cần phản hồi |
| Ví dụ | Nhiệt độ, ảnh camera | Tính toán, reset, lấy trạng thái |
| Đảm bảo gửi | Không (best-effort) | Có (request-response) |

## 7. CLI — Công cụ dòng lệnh cho Service

```bash
# Liệt kê tất cả service đang hoạt động
ros2 service list

# Xem kiểu service của một service cụ thể
ros2 service type /add_two_ints

# Gọi service trực tiếp từ CLI
ros2 service call /add_two_ints example_interfaces/srv/AddTwoInts '{a: 5, b: 3}'

# Gọi Empty service
ros2 service call /robot_reset std_srvs/srv/Empty '{}'

# Tìm tất cả service có cùng kiểu
ros2 service find example_interfaces/srv/AddTwoInts
```

## 8. Tóm tắt

- **Service** là giao tiếp một-một, đồng bộ, request-response
- **Service Server** dùng `create_service()` và callback trả về response
- **Service Client** dùng `create_client()`, `wait_for_service()`, và `call_async()`
- `std_srvs/srv/Empty` dùng cho các lệnh đơn giản không cần tham số
- **Topic** phù hợp dữ liệu liên tục, **Service** phù hợp tác vụ ngắn cần phản hồi

---

**Bài tiếp theo:** [Bài 7: Action — Goal, Feedback, Result](../08.ros2-action/ros2-action.md)
