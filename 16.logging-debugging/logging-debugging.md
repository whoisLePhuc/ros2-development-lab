# Bài 16: Logging và Debugging

Khi hệ thống ROS2 chạy, bạn cần biết chuyện gì đang xảy ra bên trong. Logging giúp ghi lại thông tin. Debugging giúp tìm và sửa lỗi. Bài này sẽ đi qua cả hai.

## 1. ROS2 Logger

Mỗi node trong ROS2 có một logger riêng. Bạn có thể ghi log từ code C++ hoặc Python.

### 1.1. Các mức log

ROS2 hỗ trợ 5 mức log, theo thứ tự nghiêm trọng tăng dần:

| Mức | Tên | Dùng khi nào |
|-----|-----|--------------|
| 1 | `DEBUG` | Chi tiết nhất, dùng khi đang trace lỗi |
| 2 | `INFO` | Thông tin thông thường |
| 3 | `WARN` | Cảnh báo, có thể không nghiêm trọng |
| 4 | `ERROR` | Lỗi, chức năng bị ảnh hưởng |
| 5 | `FATAL` | Lỗi nghiêm trọng, node có thể crash |

### 1.2. Ghi log trong Python

```python
import rclpy
from rclpy.node import Node

class MyNode(Node):
    def __init__(self):
        super().__init__('my_node')
        self.get_logger().debug('This is debug')
        self.get_logger().info('Node started')
        self.get_logger().warn('Battery low')
        self.get_logger().error('Sensor timeout')
        self.get_logger().fatal('Critical failure')
```

### 1.3. Ghi log trong C++

```cpp
#include "rclcpp/rclcpp.hpp"

class MyNode : public rclcpp::Node {
public:
    MyNode() : Node("my_node") {
        RCLCPP_DEBUG(this->get_logger(), "This is debug");
        RCLCPP_INFO(this->get_logger(), "Node started");
        RCLCPP_WARN(this->get_logger(), "Battery low");
        RCLCPP_ERROR(this->get_logger(), "Sensor timeout");
        RCLCPP_FATAL(this->get_logger(), "Critical failure");
    }
};
```

### 1.4. Thay đổi mức log tại runtime

Bạn có thể đổi mức log mà không cần biên dịch lại code.

```bash
# Đặt mức log cho một node
ros2 run my_pkg my_node --ros-args --log-level debug

# Hoặc chỉ định logger cụ thể
ros2 run my_pkg my_node --ros-args --log-level my_node:=debug
```

Từ code, bạn cũng có thể set:

```python
# Python
import logging
logger = rclpy.logging.get_logger('my_node')
logger.set_level(rclpy.logging.LoggingSeverity.DEBUG)
```

```cpp
// C++
auto logger = this->get_logger();
rcutils_logging_set_logger_level("my_node", RCUTILS_LOG_SEVERITY_DEBUG);
```

### 1.5. Log có định dạng

Bạn có thể log với biến:

```python
# Python
self.get_logger().info(f'Position: x={x}, y={y}')
```

```cpp
// C++
RCLCPP_INFO(this->get_logger(), "Position: x=%.2f, y=%.2f", x, y);
```

## 2. Topic `/rosout`

Tất cả log từ các node đều được publish lên topic `/rosout`. Điều này có nghĩa là bạn có thể xem log từ mọi node từ một terminal duy nhất.

```bash
# Xem log từ mọi node
ros2 topic echo /rosout

# Output sẽ có dạng:
# stamp: ...
# level: 20  # INFO = 20, WARN = 30, ERROR = 40
# name: "my_node"
# msg: "Node started"
```

Mức log được mã hóa thành số:

- DEBUG = 10
- INFO = 20
- WARN = 30
- ERROR = 40
- FATAL = 50

## 3. Công cụ debug qua CLI

ROS2 cung cấp nhiều công cụ dòng lệnh để kiểm tra hệ thống đang chạy.

### 3.1. Kiểm tra topic

```bash
# Liệt kê tất cả topic đang chạy
ros2 topic list

# Xem chi tiết một topic
ros2 topic info /cmd_vel

# Xem message đang được publish
ros2 topic echo /scan

# Kiểm tra tần suất publish (Hz)
ros2 topic hz /scan

# Kiểm tra độ trễ
ros2 topic delay /scan
```

### 3.2. Kiểm tra node

```bash
# Liệt kê node
ros2 node list

# Xem thông tin node
ros2 node info /turtlesim
```

### 3.3. Kiểm tra service và action

```bash
# Liệt kê service
ros2 service list

# Xem type của service
ros2 service type /spawn

# Liệt kê action
ros2 action list

# Xem thông tin action
ros2 action info /turtle1/rotate_absolute
```

### 3.4. Kiểm tra TF

```bash
# Xem transform giữa hai frame
ros2 run tf2_ros tf2_echo base_link laser

# Xuất toàn bộ TF tree ra file PDF
ros2 run tf2_tools view_frames
# File: frames.pdf
```

## 4. `ros2 doctor`

Công cụ này kiểm tra sức khỏe hệ thống ROS2 của bạn.

```bash
# Kiểm tra cơ bản
ros2 doctor

# Kiểm tra chi tiết
ros2 doctor --report
```

`ros2 doctor` sẽ kiểm tra:

- Phiên bản ROS2 và các package
- Biến môi trường (`ROS_DISTRO`, `ROS_LOCALHOST_ONLY`, v.v.)
- Network configuration
- DDS middleware
- Các vấn đề thường gặp

## 5. RQT Overview

RQT là framework GUI cho ROS2. Bạn mở nó bằng:

```bash
rqt
```

### 5.1. Node Graph

Plugins > Introspection > Node Graph

Hiển thị toàn bộ node và topic dưới dạng đồ thị. Rất hữu ích để hiểu kiến trúc hệ thống.

### 5.2. Topic Monitor

Plugins > Topics > Topic Monitor

Xem real-time message từ mọi topic. Bạn có thể chọn topic muốn theo dõi.

### 5.3. Plot

Plugins > Visualization > Plot

Vẽ đồ thị dữ liệu số từ topic. Ví dụ: vẽ vận tốc theo thời gian.

### 5.4. Console

Plugins > Logging > Console

Xem log từ `/rosout` trong giao diện GUI. Có thể lọc theo mức log và node.

## 6. Domain ID

ROS2 dùng DDS domain để phân tách các hệ thống. Mặc định là domain 0.

```bash
# Chạy node trên domain khác
ROS_DOMAIN_ID=42 ros2 run my_pkg my_node

# Hoặc export trước
export ROS_DOMAIN_ID=42
ros2 run my_pkg my_node
```

Lưu ý:

- Domain ID từ 0 đến 101 (một số DDS có giới hạn)
- Các node khác domain sẽ không nhìn thấy nhau
- Hữu ích khi chạy nhiều robot trên cùng một mạng

## 7. Các tình huống debug thường gặp

### 7.1. Node không nhìn thấy nhau

Kiểm tra:

```bash
# Có cùng ROS_DOMAIN_ID không?
echo $ROS_DOMAIN_ID

# Network có hoạt động không?
ros2 doctor

# Có dùng localhost không?
echo $ROS_LOCALHOST_ONLY  # 1 = chỉ local, 0 = network
```

### 7.2. Message không đến subscriber

Kiểm tra:

```bash
# Topic có tồn tại không?
ros2 topic list | grep my_topic

# Type có khớp không?
ros2 topic info /my_topic

# Có publish không?
ros2 topic echo /my_topic

# QoS có tương thích không?
ros2 topic info /my_topic --verbose
```

### 7.3. TF lỗi

```bash
# Kiểm tra TF tree
ros2 run tf2_tools view_frames

# Kiểm tra một transform cụ thể
ros2 run tf2_ros tf2_echo base_link odom
```

### 7.4. Node crash ngay khi khởi động

- Đọc log: `ros2 run pkg node 2>&1 | tee log.txt`
- Kiểm tra parameter file có tồn tại không
- Kiểm tra port có bị chiếm không

## Tóm tắt

- Dùng `get_logger()` để ghi log với 5 mức: DEBUG, INFO, WARN, ERROR, FATAL
- `/rosout` tập trung log từ mọi node
- CLI tools (`topic`, `node`, `service`, `action`, `tf2_echo`) giúp introspection nhanh
- `ros2 doctor` kiểm tra sức khỏe hệ thống
- RQT cung cấp GUI cho việc debug trực quan
- Domain ID giúp cô lập các hệ thống ROS2

---

**Bài tiếp theo:** [Bài 17: Rosbag2 — Record & Playback](../17.rosbag2/rosbag2.md)
