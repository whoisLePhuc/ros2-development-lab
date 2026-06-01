# Bài 8: Parameter — Cấu hình động

## 1. Parameter là gì?

**Parameter** trong ROS2 là cặp **key-value** dùng để cấu hình node. Thay vì hard-code giá trị trong code, bạn dùng parameter để thay đổi hành vi node mà không cần biên dịch lại.

Ví dụ: tần suất publish, tốc độ tối đa, tên topic, ngưỡng cảnh báo, v.v.

## 2. Khai báo và đọc Parameter

### Ví dụ cơ bản

```python
import rclpy
from rclpy.node import Node


class ParameterDemo(Node):
    def __init__(self):
        super().__init__('parameter_demo')

        # Khai báo parameter với giá trị mặc định
        self.declare_parameter('publish_rate', 1.0)
        self.declare_parameter('robot_name', 'turtlebot')
        self.declare_parameter('max_speed', 0.5)

        # Đọc giá trị parameter
        publish_rate = self.get_parameter('publish_rate').value
        robot_name = self.get_parameter('robot_name').value
        max_speed = self.get_parameter('max_speed').value

        self.get_logger().info(f'Publish rate: {publish_rate} Hz')
        self.get_logger().info(f'Robot name: {robot_name}')
        self.get_logger().info(f'Max speed: {max_speed} m/s')


def main(args=None):
    rclpy.init(args=args)
    node = ParameterDemo()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

### Các kiểu Parameter

| Kiểu | Python type | Ví dụ |
|------|-------------|-------|
| `PARAMETER_INTEGER` | `int` | `10` |
| `PARAMETER_DOUBLE` | `float` | `3.14` |
| `PARAMETER_BOOL` | `bool` | `True` |
| `PARAMETER_STRING` | `str` | `'hello'` |
| `PARAMETER_BYTE_ARRAY` | `list[int]` | `[0x01, 0x02]` |
| `PARAMETER_INTEGER_ARRAY` | `list[int]` | `[1, 2, 3]` |
| `PARAMETER_DOUBLE_ARRAY` | `list[float]` | `[1.1, 2.2]` |
| `PARAMETER_BOOL_ARRAY` | `list[bool]` | `[True, False]` |
| `PARAMETER_STRING_ARRAY` | `list[str]` | `['a', 'b']` |

## 3. Parameter động — Thay đổi trong khi chạy

Node có thể cho phép thay đổi parameter khi đang chạy bằng callback.

```python
import rclpy
from rclpy.node import Node
from rcl_interfaces.msg import SetParametersResult


class DynamicParameterNode(Node):
    def __init__(self):
        super().__init__('dynamic_parameter_node')

        self.declare_parameter('speed', 0.5)
        self.declare_parameter('enabled', True)

        # Đăng ký callback khi parameter thay đổi
        self.add_on_set_parameters_callback(self.parameter_callback)

        self.get_logger().info('Dynamic Parameter Node đã khởi động')
        self.get_logger().info('Thử chạy: ros2 param set /dynamic_parameter_node speed 1.0')

    def parameter_callback(self, params):
        result = SetParametersResult()
        result.successful = True

        for param in params:
            if param.name == 'speed':
                if param.value < 0.0:
                    result.successful = False
                    result.reason = 'speed không được âm'
                else:
                    self.get_logger().info(f'Speed thay đổi thành: {param.value}')
            elif param.name == 'enabled':
                self.get_logger().info(f'Enabled thay đổi thành: {param.value}')

        return result


def main(args=None):
    rclpy.init(args=args)
    node = DynamicParameterNode()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

### Giải thích

- `add_on_set_parameters_callback()`: Đăng ký hàm được gọi khi có yêu cầu thay đổi parameter
- `SetParametersResult`: Trả về `successful=True/False` để chấp nhận hoặc từ chối thay đổi
- Callback nhận danh sách parameter thay đổi, cho phép kiểm tra và validate

## 4. Ví dụ hoàn chỉnh: VelocityController

Node điều khiển vận tốc với parameter có thể thay đổi trực tiếp.

```python
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist
from rcl_interfaces.msg import SetParametersResult


class VelocityController(Node):
    def __init__(self):
        super().__init__('velocity_controller')

        # Khai báo parameter
        self.declare_parameter('linear_speed', 0.5)
        self.declare_parameter('angular_speed', 0.2)
        self.declare_parameter('publish_rate', 10.0)

        # Lấy giá trị ban đầu
        self.linear_speed = self.get_parameter('linear_speed').value
        self.angular_speed = self.get_parameter('angular_speed').value
        publish_rate = self.get_parameter('publish_rate').value

        # Tạo publisher và timer
        self.publisher_ = self.create_publisher(Twist, 'cmd_vel', 10)
        self.timer = self.create_timer(1.0 / publish_rate, self.publish_velocity)

        # Đăng ký callback thay đổi parameter
        self.add_on_set_parameters_callback(self.on_parameter_change)

        self.get_logger().info('Velocity Controller đã khởi động')
        self.get_logger().info(f'linear_speed={self.linear_speed}, angular_speed={self.angular_speed}')

    def on_parameter_change(self, params):
        result = SetParametersResult()
        result.successful = True

        for param in params:
            if param.name == 'linear_speed':
                if param.value < 0.0 or param.value > 2.0:
                    result.successful = False
                    result.reason = 'linear_speed phải trong khoảng [0.0, 2.0]'
                else:
                    self.linear_speed = param.value
                    self.get_logger().info(f'Linear speed cập nhật: {self.linear_speed}')
            elif param.name == 'angular_speed':
                self.angular_speed = param.value
                self.get_logger().info(f'Angular speed cập nhật: {self.angular_speed}')
            elif param.name == 'publish_rate':
                if param.value <= 0.0:
                    result.successful = False
                    result.reason = 'publish_rate phải > 0'
                else:
                    self.timer.timer_period_ns = int(1e9 / param.value)
                    self.get_logger().info(f'Publish rate cập nhật: {param.value} Hz')

        return result

    def publish_velocity(self):
        msg = Twist()
        msg.linear.x = self.linear_speed
        msg.angular.z = self.angular_speed
        self.publisher_.publish(msg)


def main(args=None):
    rclpy.init(args=args)
    node = VelocityController()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

## 5. CLI — Công cụ dòng lệnh cho Parameter

```bash
# Liệt kê tất cả parameter của một node
ros2 param list /velocity_controller

# Đọc giá trị một parameter
ros2 param get /velocity_controller linear_speed

# Thay đổi giá trị parameter
ros2 param set /velocity_controller linear_speed 1.2

# Xuất tất cả parameter ra file YAML
ros2 param dump /velocity_controller > velocity_controller_params.yaml

# Nạp parameter từ file YAML khi khởi động node
ros2 run my_package velocity_controller --ros-args --params-file velocity_controller_params.yaml
```

## 6. File YAML cấu hình Parameter

Tạo file `config/params.yaml`:

```yaml
velocity_controller:
  ros__parameters:
    linear_speed: 0.8
    angular_speed: 0.3
    publish_rate: 20.0
```

Chạy node với file parameter:

```bash
ros2 run my_package velocity_controller --ros-args --params-file config/params.yaml
```

File YAML có thể chứa nhiều node:

```yaml
velocity_controller:
  ros__parameters:
    linear_speed: 0.8
    angular_speed: 0.3

lidar_scanner:
  ros__parameters:
    scan_range: 10.0
    angle_resolution: 0.5
```

## 7. Tóm tắt

- **Parameter** là cặp key-value để cấu hình node động
- `declare_parameter()` khai báo parameter với giá trị mặc định
- `get_parameter().value` đọc giá trị parameter
- `add_on_set_parameters_callback()` cho phép validate và xử lý thay đổi parameter khi node đang chạy
- CLI `ros2 param` giúp xem, thay đổi và xuất parameter nhanh chóng
- File **YAML** dùng để lưu và nạp cấu hình parameter khi khởi động

---

**Bài tiếp theo:** [Bài 9: ROS2 Interface](../10.ros2-interface/ros2-interface.md)
