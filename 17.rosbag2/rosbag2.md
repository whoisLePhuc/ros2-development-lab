# Bài 17: Rosbag2 — Record & Playback

Rosbag2 là công cụ ghi và phát lại dữ liệu ROS2. Nó rất hữu ích cho việc test thuật toán mà không cần robot thật, hoặc lưu dataset để phân tích sau.

## 1. Record

### 1.1. Ghi tất cả topic

```bash
# Ghi mọi topic đang chạy
ros2 bag record -a

# Đặt tên cho bag
ros2 bag record -a -o my_session
```

### 1.2. Ghi topic cụ thể

```bash
# Chỉ ghi các topic bạn cần
ros2 bag record /scan /cmd_vel /odom

# Ghi nhiều topic với tên bag tùy chỉnh
ros2 bag record /scan /tf /tf_static -o lidar_dataset
```

### 1.3. Các tùy chọn hữu ích

```bash
# Giới hạn kích thước mỗi file bag (bytes)
ros2 bag record -a --max-bag-size 100000000  # ~100MB

# Nén dữ liệu
ros2 bag record -a --compression-mode file --compression-format zstd

# Ghi ở chất lượng thấp hơn (nếu dùng với QoS)
ros2 bag record -a --qos-profile-overrides-path qos.yaml
```

File `qos.yaml` ví dụ:

```yaml
/scan:
  history: keep_last
  depth: 10
  reliability: best_effort
  durability: volatile
```

## 2. Playback

### 2.1. Phát lại cơ bản

```bash
# Phát lại bag
ros2 bag play my_session

# Phát lại từ thư mục cụ thể
ros2 bag play ~/ros2_bags/lidar_dataset
```

### 2.2. Tùy chỉnh tốc độ phát

```bash
# Phát nhanh gấp 2 lần
ros2 bag play my_session --rate 2.0

# Phát chậm một nửa
ros2 bag play my_session --rate 0.5
```

### 2.3. Loop và offset

```bash
# Phát lặp vô hạn
ros2 bag play my_session --loop

# Bắt đầu từ giây thứ 10
ros2 bag play my_session --start-offset 10.0
```

### 2.4. Remap topic khi phát

```bash
# Phát topic /scan thành /scan_old
ros2 bag play my_session --remap /scan:=/scan_old
```

## 3. Xem thông tin bag

```bash
# Xem metadata của bag
ros2 bag info my_session

# Output ví dụ:
# Files:             my_session_0.db3
# Bag size:          50.3 MiB
# Storage id:        sqlite3
# Duration:          120.5s
# Start:             ...
# End:               ...
# Messages:          15000
# Topic information: 
#   Topic: /scan | Type: sensor_msgs/LaserScan | Count: 1200
#   Topic: /cmd_vel | Type: geometry_msgs/Twist | Count: 600
```

## 4. Python API

Bạn có thể đọc và ghi bag trực tiếp từ Python.

### 4.1. Đọc bag

```python
from rosbag2_py import SequentialReader, StorageOptions, ConverterOptions
from rclpy.serialization import deserialize_message
from sensor_msgs.msg import LaserScan

# Cấu hình reader
storage_options = StorageOptions(
    uri='my_session',
    storage_id='sqlite3'
)
converter_options = ConverterOptions(
    input_serialization_format='cdr',
    output_serialization_format='cdr'
)

reader = SequentialReader()
reader.open(storage_options, converter_options)

# Đọc từng message
while reader.has_next():
    topic, data, timestamp = reader.read_next()
    
    if topic == '/scan':
        msg = deserialize_message(data, LaserScan)
        print(f"Ranges: {len(msg.ranges)}, Time: {timestamp}")
```

### 4.2. Ghi bag

```python
from rosbag2_py import SequentialWriter, StorageOptions, ConverterOptions
from rclpy.serialization import serialize_message
from geometry_msgs.msg import Twist

# Cấu hình writer
storage_options = StorageOptions(
    uri='output_bag',
    storage_id='sqlite3'
)
converter_options = ConverterOptions(
    input_serialization_format='cdr',
    output_serialization_format='cdr'
)

writer = SequentialWriter()
writer.open(storage_options, converter_options)

# Tạo topic
from rosbag2_py import TopicMetadata
writer.create_topic(TopicMetadata(
    name='/cmd_vel',
    type='geometry_msgs/msg/Twist',
    serialization_format='cdr'
))

# Ghi message
msg = Twist()
msg.linear.x = 0.5
msg.angular.z = 0.1

writer.write(
    '/cmd_vel',
    serialize_message(msg),
    int(time.time() * 1e9)  # timestamp nanoseconds
)
```

## 5. Ví dụ: Ghi và phát lại dữ liệu LiDAR

### Bước 1: Khởi động robot (hoặc simulator)

```bash
# Ví dụ với TurtleBot3
ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py
```

### Bước 2: Ghi dữ liệu LiDAR

```bash
# Terminal 2
ros2 bag record /scan /tf /tf_static -o lidar_session
# Để robot di chuyển một vòng
# Ctrl+C để dừng ghi
```

### Bước 3: Kiểm tra bag

```bash
ros2 bag info lidar_session
```

### Bước 4: Phát lại

```bash
# Tắt simulator, chỉ phát bag
ros2 bag play lidar_session --loop

# Mở RViz2 để xem
rviz2
# Add LaserScan display, chọn topic /scan
```

## 6. Mẹo quản lý kích thước bag

| Vấn đề | Giải pháp |
|--------|-----------|
| Bag quá lớn | Chỉ ghi topic cần thiết |
| PointCloud2 nặng | Dùng `--compression-mode file` |
| Image raw quá lớn | Ghi topic nén hoặc giảm FPS |
| Nhiều file bag | Dùng `--max-bag-size` để chia nhỏ |

Ước tính kích thước:

- LiDAR 10Hz: ~1-5 MB/phút
- Camera 30fps: ~100-500 MB/phút
- PointCloud2 10Hz: ~50-200 MB/phút

## Tóm tắt

- `ros2 bag record -a` ghi mọi topic
- `ros2 bag play` phát lại dữ liệu
- `ros2 bag info` xem metadata
- Python API (`SequentialReader`, `SequentialWriter`) cho phép xử lý bag programmatically
- Nén và giới hạn kích thước giúp quản lý storage

---

**Bài tiếp theo:** [Bài 18: Docker cho ROS2](../18.docker-for-ros2/docker-for-ros2.md)
