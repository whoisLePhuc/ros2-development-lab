# Bài 1: Tổng quan ROS2

ROS2 là nền tảng phát triển robot phổ biến nhất hiện nay. Bài này giúp bạn hiểu ROS2 là gì, tại sao nó tồn tại, và các khái niệm cốt lõi.

---

## 1. ROS là gì?

**ROS (Robot Operating System)** không phải là một hệ điều hành. Nó là một bộ middleware, tức là phần mềm trung gian giúp các thành phần của robot giao tiếp với nhau.

Hãy tưởng tượng robot của bạn có nhiều bộ phận:
- Camera nhận diện vật thể
- Lidar đo khoảng cách
- Motor điều khiển bánh xe
- AI lập kế hoạch đường đi

Mỗi bộ phận là một chương trình riêng. ROS2 giúp chúng "nói chuyện" với nhau theo cách chuẩn hóa, để bạn không phải tự viết giao thức truyền thông từ đầu.

---

## 2. ROS1 vs ROS2

| Đặc điểm | ROS1 | ROS2 |
|----------|------|------|
| Hệ điều hành | Ubuntu (chủ yếu) | Ubuntu, Windows, macOS |
| Giao thức truyền thông | TCPROS/UDPROS tự viết | DDS (chuẩn công nghiệp) |
| Real-time | Không hỗ trợ | Hỗ trợ real-time |
| Bảo mật | Không có | Có bảo mật (SROS2) |
| Khởi động | Cần roscore | Không cần roscore |
| Ngôn ngữ | C++, Python | C++, Python, và nhiều hơn |

**Tại sao chọn ROS2?**
- ROS1 đã ngừng phát triển tính năng mới từ năm 2020.
- ROS2 dùng DDS, cho phép robot giao tiếp qua mạng LAN/WAN dễ dàng.
- ROS2 phù hợp cho sản xuất thực tế, không chỉ nghiên cứu.

---

## 3. ROS Graph: Các thành phần cốt lõi

ROS Graph là mạng lưới các chương trình đang chạy và kết nối giữa chúng.

### Node

Node là một đơn vị tính toán. Mỗi node thường đảm nhận một nhiệm vụ cụ thể.

Ví dụ:
- `camera_node`: đọc dữ liệu từ camera
- `detection_node`: nhận diện vật thể
- `control_node`: gửi lệnh đến motor

Một chương trình Python hoặc C++ có thể chứa một hoặc nhiều node.

### Topic

Topic là kênh giao tiếp một chiều, dùng để truyền dữ liệu liên tục.

- Publisher gửi dữ liệu lên topic.
- Subscriber đăng ký nhận dữ liệu từ topic.
- Một topic có thể có nhiều publisher và nhiều subscriber.

Ví dụ thực tế:
- Topic `/camera/image_raw`: camera_node publish, detection_node subscribe.
- Topic `/cmd_vel`: control_node publish, motor_driver subscribe.

### Service

Service là giao tiếp hai chiều theo kiểu request/response.

- Client gửi request.
- Server xử lý và trả về response.
- Phù hợp cho tác vụ ngắn, cần phản hồi ngay.

Ví dụ:
- Client gọi service `/get_map` để nhận bản đồ từ server.
- Client gọi service `/set_mode` để đổi chế độ hoạt động robot.

### Action

Action là giao tiếp hai chiều cho tác vụ dài hạn, có thể bị hủy giữa chừng.

- Client gửi goal (mục tiêu).
- Server báo cáo tiến độ (feedback) liên tục.
- Khi xong, server trả kết quả (result).

Ví dụ:
- Robot di chuyển đến tọa độ (x, y): action `/navigate_to_pose`.
- Robot quay đầu 90 độ: action `/rotate`.

### Parameter

Parameter là cấu hình của node, lưu giá trị như số, chuỗi, hoặc list.

Ví dụ:
- `max_speed`: 1.0
- `robot_name`: "TurtleBot3"
- `use_sim_time`: true

---

## 4. DDS Middleware

ROS2 dùng DDS (Data Distribution Service) làm lớp truyền thông. DDS là chuẩn OMG, đã được dùng trong hàng không, quốc phòng, và xe tự hành.

**Lợi ích của DDS:**
- Tự động phát hiện node trên mạng.
- QoS (Quality of Service): điều chỉnh độ tin cậy, độ trễ, băng thông.
- Không cần master node như `roscore` trong ROS1.

Các implementation DDS phổ biến trong ROS2:
- Fast DDS (mặc định trên Humble)
- Cyclone DDS
- RTI Connext

---

## 5. Kiến trúc phân tán (Distributed Architecture)

Một hệ thống ROS2 có thể chạy trên nhiều máy tính cùng lúc.

```
[Máy tính 1 - Robot]
  ├── camera_node
  ├── lidar_node
  └── motor_driver_node

[Máy tính 2 - Ground Station]
  ├── rviz2 (hiển thị)
  └── teleop_node (điều khiển)
```

Các node trên hai máy tính giao tiếp qua mạng như thể chúng chạy trên cùng một máy. Bạn chỉ cần đảm bảo các máy cùng domain ID (môi trường `ROS_DOMAIN_ID`).

---

## 6. Cấu trúc Workspace

Workspace là thư mục chứa toàn bộ code ROS2 của bạn.

```
ros2_ws/                  # Thư mục workspace
├── build/                # File build tạm (tự động tạo)
├── install/              # File cài đặt sau build
├── log/                  # Log build
└── src/                  # Source code của bạn
    ├── my_package/         # Một gói ROS2
    │   ├── package.xml     # Khai báo phụ thuộc
    │   ├── setup.py        # Cấu hình Python
    │   └── my_package/
    │       └── __init__.py
    └── another_package/
```

**Các file quan trọng:**
- `package.xml`: khai báo tên gói, phiên bản, tác giả, và các gói phụ thuộc.
- `setup.py`: cấu hình entry point cho node Python.
- `CMakeLists.txt`: dùng cho gói C++.

---

## Tóm tắt

- ROS2 là middleware, không phải OS.
- ROS2 thay thế ROS1 với DDS, real-time, và bảo mật.
- ROS Graph gồm: Node, Topic, Service, Action, Parameter.
- DDS cho phép giao tiếp phân tán không cần master.
- Workspace tổ chức code theo gói trong thư mục `src/`.

---

**Bài tiếp theo:** [Bài 2: Cài đặt ROS2](../03.ros2-installation/ros2-installation.md)
