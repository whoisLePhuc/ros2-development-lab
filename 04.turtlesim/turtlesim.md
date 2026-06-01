# Bài 3: Làm quen ROS2 với Turtlesim

Turtlesim là công cụ mô phỏng đơn giản nhất trong ROS2. Nó giúp bạn hiểu ROS Graph mà không cần phần cứng robot thật.

---

## 1. Cài đặt Turtlesim

```bash
sudo apt update
sudo apt install ros-humble-turtlesim
```

Kiểm tra cài đặt:

```bash
ros2 pkg executables turtlesim
```

Bạn sẽ thấy danh sách các node có sẵn:
```
turtlesim draw_square
turtlesim mimic
turtlesim turtle_teleop_key
turtlesim turtlesim_node
```

---

## 2. Chạy Turtlesim

Bạn cần mở ít nhất 2 terminal.

### Terminal 1: Khởi động mô phỏng

```bash
ros2 run turtlesim turtlesim_node
```

Một cửa sổ sẽ hiện ra với một con rùa ở giữa. Đây chính là node `turtlesim_node`.

### Terminal 2: Điều khiển bằng bàn phím

```bash
ros2 run turtlesim turtle_teleop_key
```

Dùng các phím mũi tên để di chuyển con rùa. Giữ phím để rùa di chuyển, thả ra để dừng.

> **Lưu ý:** Terminal chạy `turtle_teleop_key` phải được focus (click vào) để nhận phím.

---

## 3. Khám phá ROS Graph bằng CLI

ROS2 cung cấp nhiều lệnh CLI để quan sát hệ thống đang chạy.

### Liệt kê node

```bash
ros2 node list
```

Output:
```
/teleop_turtle
/turtlesim
```

### Liệt kê topic

```bash
ros2 topic list
```

Output:
```
/parameter_events
/rosout
/turtle1/cmd_vel
/turtle1/color_sensor
/turtle1/pose
```

### Xem thông tin topic

```bash
ros2 topic info /turtle1/cmd_vel
```

Output:
```
Type: geometry_msgs/msg/Twist
Publisher count: 1
Subscription count: 1
```

Topic `/turtle1/cmd_vel` dùng message type `Twist` để gửi lệnh vận tốc. `teleop_turtle` publish, `turtlesim` subscribe.

### Theo dõi dữ liệu topic (echo)

```bash
ros2 topic echo /turtle1/cmd_vel
```

Khi bạn bấm phím mũi tên, bạn sẽ thấy dữ liệu hiện ra:
```
linear:
  x: 2.0
  y: 0.0
  z: 0.0
angular:
  x: 0.0
  y: 0.0
  z: 0.0
```

### Theo dõi topic pose

```bash
ros2 topic echo /turtle1/pose
```

Hiển thị tọa độ (x, y) và góc theta của rùa theo thời gian thực.

### Kiểm tra tần số publish (hz)

```bash
ros2 topic hz /turtle1/cmd_vel
```

Output khi bạn giữ phím:
```
average rate: 62.5
	min: 0.016s max: 0.016s std dev: 0.00000s window: 63
```

### Publish trực tiếp lên topic

Bạn có thể gửi lệnh mà không cần `teleop_turtle`:

```bash
ros2 topic pub /turtle1/cmd_vel geometry_msgs/msg/Twist "{linear: {x: 2.0}, angular: {z: 1.0}}" --rate 1
```

Con rùa sẽ quay tròn. Nhấn `Ctrl+C` để dừng.

---

## 4. Gọi Service

Turtlesim cung cấp nhiều service để tương tác.

### Xem danh sách service

```bash
ros2 service list
```

Output:
```
/clear
/kill
/reset
/spawn
/turtle1/set_pen
/turtle1/teleport_absolute
/turtle1/teleport_relative
/turtlesim/describe_parameters
...
```

### Xem type của service

```bash
ros2 service type /spawn
```

Output: `turtlesim/srv/Spawn`

### Gọi service spawn (tạo rùa mới)

```bash
ros2 service call /spawn turtlesim/srv/Spawn "{x: 5.0, y: 5.0, theta: 0.0, name: 'turtle2'}"
```

Một con rùa thứ hai sẽ xuất hiện tại tọa độ (5, 5).

### Gọi service clear (xóa vết)

```bash
ros2 service call /clear std_srvs/srv/Empty
```

Xóa toàn bộ đường vẽ trên màn hình.

### Gọi service set_pen (đổi màu bút)

```bash
ros2 service call /turtle1/set_pen turtlesim/srv/SetPen "{r: 255, g: 0, b: 0, width: 5}"
```

Rùa `turtle1` sẽ vẽ bằng màu đỏ, nét dày 5 pixel.

---

## 5. Ghi và phát lại dữ liệu với `ros2 bag`

`ros2 bag` cho phép ghi lại dữ liệu topic để phân tích hoặc phát lại sau này.

### Cài đặt ros2 bag

```bash
sudo apt install ros-humble-ros2bag ros-humble-rosbag2-storage-default-plugins
```

### Ghi dữ liệu

```bash
# Tạo thư mục lưu bag
mkdir -p ~/ros2_bag
cd ~/ros2_bag

# Ghi topic /turtle1/cmd_vel và /turtle1/pose
ros2 bag record /turtle1/cmd_vel /turtle1/pose
```

Di chuyển rùa một lúc, sau đó nhấn `Ctrl+C` để dừng ghi. Một thư mục bag sẽ được tạo với tên dạng `rosbag2_YYYY_MM_DD-HH_MM_SS`.

### Xem thông tin bag

```bash
ros2 bag info rosbag2_YYYY_MM_DD-HH_MM_SS
```

Output:
```
Files:             rosbag2_YYYY_MM_DD-HH_MM_SS_0.db3
Bag size:          12.3 KiB
Storage id:        sqlite3
Duration:          15.2s
Start:             ...
End:               ...
Messages:          305
Topic information: Topic: /turtle1/cmd_vel | Type: geometry_msgs/msg/Twist | Count: 15
                   Topic: /turtle1/pose | Type: turtlesim/msg/Pose | Count: 290
```

### Phát lại bag

Tắt `teleop_turtle` (để tránh xung đột), sau đó:

```bash
ros2 bag play rosbag2_YYYY_MM_DD-HH_MM_SS
```

Rùa sẽ di chuyển lại đúng như lúc bạn ghi. Điều này rất hữu ích để debug thuật toán mà không cần chạy robot thật.

---

## 6. ROS Graph trong Turtlesim

Hãy tổng hợp những gì bạn đã thấy:

```
[teleop_turtle] --(publish /turtle1/cmd_vel)--> [turtlesim_node]
                                    |
                                    v
                           [ros2 topic echo]
                           [ros2 bag record]
```

- **Node:** `turtlesim_node` (mô phỏng), `teleop_turtle` (điều khiển).
- **Topic:** `/turtle1/cmd_vel` truyền lệnh vận tốc.
- **Service:** `/spawn`, `/clear`, `/set_pen` để tương tác trực tiếp.
- **Bag:** Ghi lại dữ liệu topic để phát lại.

---

## Tóm tắt

- `ros2 run turtlesim turtlesim_node`: chạy mô phỏng.
- `ros2 run turtlesim turtle_teleop_key`: điều khiển bằng phím.
- `ros2 node/topic/service list`: khám phá hệ thống.
- `ros2 topic echo/hz/pub`: quan sát và gửi dữ liệu.
- `ros2 service call`: gọi service tương tác.
- `ros2 bag record/play`: ghi và phát lại dữ liệu.

---

**Bài tiếp theo:** [Bài 4: Viết ROS2 Node](../05.ros2-node/ros2-node.md)
