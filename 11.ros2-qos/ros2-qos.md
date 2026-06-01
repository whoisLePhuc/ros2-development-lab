# Bài 10: QoS — Quality of Service

ROS2 replaces ROS1's TCP-only transport with DDS, which brings **Quality of Service (QoS)** policies. QoS lets you tune how data flows between publishers and subscribers. A LiDAR stream and a motor command need very different guarantees. This lesson explains every policy and shows how to configure them in code.

---

## 1. Why QoS Matters

Imagine two scenarios:

- **LiDAR sensor**: publishes 10,000 points at 10 Hz. If one message drops, the next scan arrives in 100 ms. You want speed, not perfect delivery.
- **Motor controller**: receives a stop command. That message must arrive, period. You want reliability, even if it costs a little latency.

QoS lets each topic choose the right trade-off.

---

## 2. Reliability

| Policy | Behavior | Best for |
|--------|----------|----------|
| `RELIABLE` | Retries until delivery is confirmed | Commands, configuration |
| `BEST_EFFORT` | Sends once, no retry | Sensor data, video streams |

```python
from rclpy.qos import QoSReliabilityPolicy

qos_reliable = QoSProfile(
    reliability=QoSReliabilityPolicy.RELIABLE
)
qos_best_effort = QoSProfile(
    reliability=QoSReliabilityPolicy.BEST_EFFORT
)
```

---

## 3. Durability

| Policy | Behavior | Best for |
|--------|----------|----------|
| `VOLATILE` | New subscribers see only future messages | Real-time streams |
| `TRANSIENT_LOCAL` | Publisher keeps last N messages for late joiners | Maps, configuration, static transforms |

```python
from rclpy.qos import QoSDurabilityPolicy

qos_transient = QoSProfile(
    durability=QoSDurabilityPolicy.TRANSIENT_LOCAL
)
```

---

## 4. History

| Policy | Behavior |
|--------|----------|
| `KEEP_LAST` | Buffer up to `depth` messages, drop oldest when full |
| `KEEP_ALL` | Buffer everything until the resource limit is hit |

```python
from rclpy.qos import QoSHistoryPolicy

qos_keep_last = QoSProfile(
    history=QoSHistoryPolicy.KEEP_LAST,
    depth=10
)
```

---

## 5. Depth

Depth only applies when `history=KEEP_LAST`. It sets how many messages the publisher or subscriber buffers.

```python
qos_sensor = QoSProfile(
    history=QoSHistoryPolicy.KEEP_LAST,
    depth=1   # Only care about the newest scan
)
```

---

## 6. Deadline

`deadline` specifies the maximum expected interval between consecutive messages. If a publisher misses the deadline, the subscriber can detect it.

```python
from rclpy.duration import Duration

qos_deadline = QoSProfile(
    deadline=Duration(seconds=0.1)  # Expect 10 Hz
)
```

---

## 7. QoS Presets

ROS2 provides ready-made profiles:

| Preset | Reliability | Durability | History | Depth | Typical use |
|--------|-------------|------------|---------|-------|-------------|
| `sensor_data` | BEST_EFFORT | VOLATILE | KEEP_LAST | 1 | LiDAR, camera, IMU |
| `parameters` | RELIABLE | VOLATILE | KEEP_LAST | 1000 | Parameter server |
| `default` | RELIABLE | VOLATILE | KEEP_LAST | 10 | General topics |
| `services` | RELIABLE | VOLATILE | KEEP_ALL | - | Service topics |
| `parameter_events` | RELIABLE | VOLATILE | KEEP_ALL | - | Parameter changes |

```python
from rclpy.qos import qos_profile_sensor_data

self.pub = self.create_publisher(LaserScan, 'scan', qos_profile_sensor_data)
```

---

## 8. QoS Compatibility Rules

A publisher and subscriber only communicate if their QoS is **compatible**.

| Publisher | Subscriber | Compatible? | Result |
|-----------|------------|-------------|--------|
| RELIABLE | RELIABLE | Yes | RELIABLE |
| RELIABLE | BEST_EFFORT | Yes | BEST_EFFORT |
| BEST_EFFORT | RELIABLE | **No** | No connection |
| BEST_EFFORT | BEST_EFFORT | Yes | BEST_EFFORT |
| TRANSIENT_LOCAL | TRANSIENT_LOCAL | Yes | TRANSIENT_LOCAL |
| TRANSIENT_LOCAL | VOLATILE | Yes | VOLATILE |
| VOLATILE | TRANSIENT_LOCAL | **No** | No connection |

The rule of thumb: the subscriber must ask for **equal or less** guarantee than the publisher offers.

---

## 9. Code Examples

### Publisher with custom QoS

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import LaserScan
from rclpy.qos import QoSProfile, QoSReliabilityPolicy, QoSHistoryPolicy

class LidarPublisher(Node):
    def __init__(self):
        super().__init__('lidar_publisher')
        qos = QoSProfile(
            reliability=QoSReliabilityPolicy.BEST_EFFORT,
            history=QoSHistoryPolicy.KEEP_LAST,
            depth=1
        )
        self.pub = self.create_publisher(LaserScan, 'scan', qos)
        self.timer = self.create_timer(0.1, self.publish_scan)

    def publish_scan(self):
        msg = LaserScan()
        # Fill msg fields...
        self.pub.publish(msg)

def main(args=None):
    rclpy.init(args=args)
    node = LidarPublisher()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

### Subscriber with custom QoS

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import LaserScan
from rclpy.qos import QoSProfile, QoSReliabilityPolicy, QoSHistoryPolicy

class LidarSubscriber(Node):
    def __init__(self):
        super().__init__('lidar_subscriber')
        qos = QoSProfile(
            reliability=QoSReliabilityPolicy.BEST_EFFORT,
            history=QoSHistoryPolicy.KEEP_LAST,
            depth=10
        )
        self.sub = self.create_subscription(
            LaserScan, 'scan', self.scan_callback, qos
        )

    def scan_callback(self, msg):
        self.get_logger().info(f'Received scan with {len(msg.ranges)} points')

def main(args=None):
    rclpy.init(args=args)
    node = LidarSubscriber()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

---

## 10. Inspect QoS from CLI

```bash
# See QoS settings for a topic
ros2 topic info /scan --verbose

# Output shows publisher and subscriber QoS side by side
# Look for "Reliability", "Durability", "History", "Depth"
```

If you see `Incompatible QoS` in the output, check the compatibility table above.

---

## 11. Key Takeaways

- `RELIABLE` for commands, `BEST_EFFORT` for sensors.
- `TRANSIENT_LOCAL` lets late subscribers catch up.
- `KEEP_LAST` with `depth=1` is common for real-time sensors.
- Always check compatibility when a subscriber does not receive data.

---

## Bài tiếp theo

[Bài 12: TF2 — Coordinate Transforms](../12.ros2-tf2/ros2-tf2.md)
