# Bài 38: DDS Deep Dive

## 1. DDS là gì?

ROS2 sử dụng **DDS (Data Distribution Service)** làm middleware giao tiếp giữa các node. DDS là một chuẩn OMG (Object Management Group) cho giao tiếp publish-subscribe trong hệ thống real-time.

### Kiến trúc DDS: DCPS

DDS dựa trên mô hình **DCPS (Data-Centric Publish-Subscribe)**:

- **Domain**: Không gian logic chứa các participant. Các participant trong cùng domain có thể giao tiếp.
- **Participant**: Đại diện cho một ứng dụng (node ROS2) tham gia vào domain.
- **Topic**: Tên kênh giao tiếp (tương đương topic ROS2).
- **Publisher / Subscriber**: Đối tượng DDS tương ứng với publisher/subscriber ROS2.
- **DataWriter / DataReader**: Ghi/đọc dữ liệu vào topic.

```
ROS2 Node
    └── RMW (ROS Middleware Interface)
            └── DDS Implementation (FastDDS / CycloneDDS / RTI Connext)
                    └── Network (UDP / Shared Memory / TCP)
```

### RMW (ROS Middleware)

RMW là lớp abstraction giúp ROS2 không phụ thuộc vào một DDS implementation cụ thể. Khi bạn viết `rclcpp::Publisher`, RMW chuyển lệnh này thành lệnh DDS tương ứng.

## 2. Các DDS Implementation

ROS2 Humble hỗ trợ nhiều DDS implementation:

| Implementation | Mặc định? | Đặc điểm |
|---------------|-----------|----------|
| **FastDDS** | Yes (Humble) | eProsima, open-source, nhiều tùy chọn tuning, hỗ trợ SharedMemory |
| **CycloneDDS** | No | Eclipse Foundation, đơn giản, ổn định, tốt cho embedded |
| **RTI Connext** | No | Thương mại, certified, dùng trong aerospace/medical |
| **GurumDDS** | No | Hàn Quốc, thương mại |

### Chuyển đổi DDS Implementation

Kiểm tra DDS đang dùng:

```bash
echo $RMW_IMPLEMENTATION
```

Chuyển sang CycloneDDS:

```bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
ros2 run my_pkg my_node
```

Hoặc trong launch file:

```python
from launch import LaunchDescription
from launch.actions import SetEnvironmentVariable
from launch_ros.actions import Node

def generate_launch_description():
    return LaunchDescription([
        SetEnvironmentVariable('RMW_IMPLEMENTATION', 'rmw_cyclonedds_cpp'),
        Node(package='my_pkg', executable='my_node'),
    ])
```

Cài đặt CycloneDDS:

```bash
sudo apt install ros-humble-rmw-cyclonedds-cpp
```

## 3. Discovery Mechanism

DDS sử dụng **automatic discovery**: các participant tự động tìm nhau trên mạng.

### Cách discovery hoạt động

1. Khi node khởi động, nó gửi **multicast** gói tin "hello" đến địa chỉ `239.255.0.1` (hoặc broadcast nếu multicast không khả dụng).
2. Các node khác trong cùng domain nhận được và trao đổi thông tin topic, type, QoS.
3. Sau discovery, các node giao tiếp trực tiếp (unicast).

### Kiểm tra discovery

```bash
# Xem tất cả node đang discovery
ros2 node list

# Xem topic và type
ros2 topic list -t

# Xem thông tin DDS participant (FastDDS)
ros2 daemon stop
FASTDDS_STATISTICS="HISTORY_LATENCY_TOPIC;PUBLICATION_THROUGHPUT_TOPIC" ros2 run my_pkg my_node
```

### Vấn đề với discovery

- **Mạng lớn**: Quá nhiều node gửi multicast gây nghẽn.
- **WiFi không ổn định**: Multicast bị drop trên WiFi.
- **Docker / VM**: Cần cấu hình network đúng cách.

## 4. Domain ID

Domain ID tách biệt các nhóm node trên cùng một mạng vật lý.

### Mặc định

- ROS2 mặc định dùng **Domain ID = 0**.
- Các node trong domain 0 không thấy node trong domain 1.

### Thay đổi Domain ID

```bash
export ROS_DOMAIN_ID=42
ros2 run my_pkg my_node
```

Hoặc trong code:

```cpp
// Không khuyến khích, dùng env var thay vì hardcode
setenv("ROS_DOMAIN_ID", "42", 1);
rclcpp::init(argc, argv);
```

### Khi nào dùng Domain ID?

- **Multi-robot**: Mỗi robot một domain để tránh topic trùng tên.
- **Môi trường dev/test**: Developer A dùng domain 10, Developer B dùng domain 20.
- **Security**: Tách biệt mạng logic (nhưng không thay thế encryption).

### Giới hạn Domain ID

- FastDDS: 0 - 232 (một số ID bị reserve cho OS).
- CycloneDDS: 0 - 230.
- Khuyến nghị: Dùng 0-100 để tránh xung đột với port hệ thống.

## 5. FastDDS XML Tuning

FastDDS cho phép tinh chỉnh sâu qua file XML.

### Ví dụ: Bật SharedMemory Transport

SharedMemory cho phép zero-copy giữa các node trong cùng máy (không qua network stack).

**fastdds_config.xml**:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<dds>
    <profiles>
        <transport_descriptors>
            <transport_descriptor>
                <transport_id>shm_transport</transport_id>
                <type>SHM</type>
            </transport_descriptor>
        </transport_descriptors>

        <participant profile_name="shm_profile" is_default_profile="true">
            <rtps>
                <userTransports>
                    <transport_id>shm_transport</transport_id>
                </userTransports>
                <useBuiltinTransports>false</useBuiltinTransports>
            </rtps>
        </participant>
    </profiles>
</dds>
```

Chạy với config:

```bash
export FASTRTPS_DEFAULT_PROFILES_FILE=~/fastdds_config.xml
ros2 run my_pkg my_node
```

### Ví dụ: Tăng buffer size cho image topic

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<dds>
    <profiles>
        <publisher profile_name="image_publisher_profile">
            <qos>
                <publishMode>
                    <kind>ASYNCHRONOUS</kind>
                </publishMode>
                <historyQos>
                    <kind>KEEP_LAST</kind>
                    <depth>10</depth>
                </historyQos>
            </qos>
        </publisher>
    </profiles>
</dds>
```

### Ví dụ: Giới hạn network interface

```xml
<participant profile_name="eth0_only">
    <rtps>
        <defaultUnicastLocatorList>
            <locator>
                <udpv4>
                    <address>192.168.1.100</address>
                </udpv4>
            </locator>
        </defaultUnicastLocatorList>
    </rtps>
</participant>
```

## 6. Performance Considerations

### QoS tối ưu cho throughput cao

```cpp
#include "rclcpp/qos.hpp"

// Tạo publisher với QoS tối ưu throughput
auto qos = rclcpp::QoS(10)
  .reliability(RMW_QOS_POLICY_RELIABILITY_BEST_EFFORT)
  .durability(RMW_QOS_POLICY_DURABILITY_VOLATILE)
  .history(RMW_QOS_POLICY_HISTORY_KEEP_LAST);

auto pub = node->create_publisher<sensor_msgs::msg::Image>("camera/image", qos);
```

| QoS | Khi nào dùng | Tác động performance |
|-----|-------------|---------------------|
| Best Effort | Dữ liệu real-time (camera, lidar) | Nhanh hơn, không retry |
| Reliable | Dữ liệu quan trọng (command, config) | Chậm hơn, có ack |
| Keep Last 1 | Chỉ cần data mới nhất | Ít bộ nhớ |
| Keep Last 10 | Cần buffer chống jitter | Nhiều bộ nhớ hơn |

### Tắt discovery nếu không cần

Trong hệ thống static (biết trước IP), bạn có thể dùng **Initial Peers** thay vì multicast discovery:

```xml
<participant profile_name="static_discovery">
    <rtps>
        <builtin>
            <initialPeersList>
                <locator>
                    <udpv4>
                        <address>192.168.1.10</address>
                    </udpv4>
                </locator>
            </initialPeersList>
            <metatrafficUnicastLocatorList>
                <locator>
                    <udpv4>
                        <address>192.168.1.20</address>
                    </udpv4>
                </locator>
            </metatrafficUnicastLocatorList>
        </builtin>
    </rtps>
</participant>
```

### Monitor performance

```bash
# Xem bandwidth theo topic
ros2 topic bw /topic_name

# Xem Hz thực tế
ros2 topic hz /topic_name

# Xem độ trễ (nếu message có timestamp)
ros2 topic delay /topic_name
```

## 7. Tóm tắt

| Khái niệm | Ý nghĩa |
|-----------|---------|
| DDS | Middleware publish-subscribe dùng trong ROS2 |
| DCPS | Data-Centric Publish-Subscribe, mô hình của DDS |
| RMW | Lớp abstraction giữa ROS2 và DDS |
| FastDDS | DDS mặc định trong ROS2 Humble |
| CycloneDDS | DDS nhẹ, ổn định |
| Discovery | Tự động tìm node/topic trên mạng |
| Domain ID | Tách biệt mạng logic |
| SharedMemory | Zero-copy trong cùng máy |
| `FASTRTPS_DEFAULT_PROFILES_FILE` | Biến môi trường chỉ đến file XML config |

---

**Bài tiếp theo:** [Bài 39: Multi-Robot](../39.multi-robot/multi-robot.md)
