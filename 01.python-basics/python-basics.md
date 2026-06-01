# Bài 0.2: Python cơ bản cho ROS2

ROS2 hỗ trợ C++ và Python. Với người mới, Python là lựa chọn nhanh và dễ tiếp cận. Bài này chỉ tập trung những gì bạn thực sự cần để viết node ROS2.

---

## 1. Function (Hàm)

Hàm giúp tái sử dụng code. Trong ROS2, bạn sẽ viết hàm callback để xử lý dữ liệu từ topic.

```python
def greet(name):
    """In lời chào."""
    print(f"Hello, {name}!")

greet("ROS2")
# Output: Hello, ROS2!
```

Hàm có giá trị trả về:

```python
def add(a, b):
    return a + b

result = add(3, 5)
print(result)  # 8
```

---

## 2. Class và `__init__`

Node trong ROS2 là một class. Bạn cần hiểu cách class hoạt động.

```python
class Robot:
    def __init__(self, name, speed):
        # __init__ chạy khi tạo object
        self.name = name
        self.speed = speed

    def move(self):
        print(f"{self.name} đang di chuyển với tốc độ {self.speed} m/s")

# Tạo object
my_robot = Robot("TurtleBot", 0.5)
my_robot.move()
# Output: TurtleBot đang di chuyển với tốc độ 0.5 m/s
```

> `self` là tham chiếu đến object hiện tại. Mọi phương thức trong class đều cần `self` làm tham số đầu tiên.

---

## 3. Import module

Python dùng `import` để sử dụng thư viện bên ngoài. ROS2 cung cấp rất nhiều module như `rclpy`, `std_msgs`, `geometry_msgs`.

```python
# Import toàn bộ module
import math
print(math.pi)

# Import cụ thể một hàm/class
from math import sqrt
print(sqrt(16))  # 4.0

# Import với alias
import numpy as np

# Import module tự viết
from my_package import utils
```

---

## 4. `if __name__ == '__main__'`

Đây là pattern chuẩn để chạy code chỉ khi file được thực thi trực tiếp, không chạy khi bị import.

```python
def main():
    print("Chương trình chính đang chạy")

if __name__ == '__main__':
    main()
```

Trong ROS2, pattern này gần như xuất hiện trong mọi file node:

```python
import rclpy
from rclpy.node import Node

class MyNode(Node):
    def __init__(self):
        super().__init__('my_node')
        self.get_logger().info("Node đã khởi tạo")

def main(args=None):
    rclpy.init(args=args)
    node = MyNode()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

---

## 5. List và Dict

Hai kiểu dữ liệu này xuất hiện liên tục khi xử lý message và cấu hình trong ROS2.

### List

```python
# Tạo list
topics = ["/cmd_vel", "/odom", "/scan"]

# Truy cập phần tử
print(topics[0])  # /cmd_vel

# Thêm phần tử
topics.append("/imu")

# Duyệt list
for topic in topics:
    print(f"Đang đăng ký: {topic}")

# List comprehension
velocities = [0.1, 0.5, 1.0, 2.0]
fast_vels = [v for v in velocities if v > 0.5]
print(fast_vels)  # [1.0, 2.0]
```

### Dict

```python
# Tạo dict
robot_config = {
    "name": "TurtleBot3",
    "max_speed": 1.0,
    "wheel_radius": 0.033
}

# Truy cập giá trị
print(robot_config["name"])  # TurtleBot3

# Thêm cặp key-value
robot_config["sensor"] = "Lidar"

# Duyệt dict
for key, value in robot_config.items():
    print(f"{key}: {value}")
```

---

## 6. ROS2 Node Preview

Đây là cấu trúc tối thiểu của một node ROS2 viết bằng Python. Bạn sẽ học chi tiết ở các bài sau, nhưng hãy làm quen với pattern này ngay từ bây giờ.

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String

class TalkerNode(Node):
    def __init__(self):
        super().__init__('talker')
        self.publisher_ = self.create_publisher(String, 'chatter', 10)
        timer_period = 1.0  # giây
        self.timer = self.create_timer(timer_period, self.timer_callback)
        self.count = 0

    def timer_callback(self):
        msg = String()
        msg.data = f"Hello ROS2 #{self.count}"
        self.publisher_.publish(msg)
        self.get_logger().info(f"Published: {msg.data}")
        self.count += 1

def main(args=None):
    rclpy.init(args=args)
    node = TalkerNode()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

**Các điểm chính:**
- Class kế thừa từ `Node`.
- `super().__init__('talker')` đặt tên node.
- `create_publisher()` tạo publisher trên topic.
- `create_timer()` gọi hàm lặp lại theo chu kỳ.
- `rclpy.spin(node)` giữ node chạy liên tục.

---

## Tóm tắt

- **Function:** đóng gói logic tái sử dụng.
- **Class + `__init__`:** tạo object có trạng thái và hành vi.
- **Import:** dùng thư viện ROS2 và Python.
- **`if __name__ == '__main__'`:** điểm vào chương trình.
- **List/Dict:** lưu trữ và xử lý dữ liệu.
- **Node pattern:** class kế thừa `Node`, dùng `spin()` để chạy.

---

**Bài tiếp theo:** [Bài 1: Tổng quan ROS2](../02.ros2-overview/ros2-overview.md)
