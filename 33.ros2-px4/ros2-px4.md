# Bài 33: ROS2 và PX4

## 1. Kiến trúc Offboard Control

Offboard control cho phép companion computer (chạy ROS2) điều khiển drone thay vì autopilot tự động. Kiến trúc:

```
+---------------+         uORB          +---------------+
|  Companion PC |  <------------------>  |   PX4 FCU     |
|  (ROS2 Node)  |    (microRTPS/UDP)   |  (STM32/NuttX)|
+---------------+                       +---------------+
       |                                         |
       v                                         v
  /fmu/in/*                                 Sensors/Actuators
  /fmu/out/*
```

Luồng điều khiển:
1. ROS2 node publish `OffboardControlMode` để báo PX4 chuyển sang chế độ offboard.
2. ROS2 node publish `TrajectorySetpoint` (hoặc `VehicleRatesSetpoint`) liên tục.
3. ROS2 node gửi `VehicleCommand` để arm và chuyển mode.
4. PX4 nhận setpoint và điều khiển drone theo.

## 2. Yêu cầu an toàn quan trọng

**BẮT BUỘC**: Node offboard phải gửi setpoint với tần số **> 2Hz** (tốt nhất là 10-30Hz). Nếu không, PX4 sẽ tự động rời chế độ offboard và kích hoạt failsafe (thường là land hoặc hold).

## 3. Code: OffboardControl Node

```python
import rclpy
from rclpy.node import Node
from rclpy.qos import QoSProfile, ReliabilityPolicy, HistoryPolicy, DurabilityPolicy

from px4_msgs.msg import OffboardControlMode, TrajectorySetpoint, VehicleCommand, VehicleOdometry


class OffboardControl(Node):
    def __init__(self):
        super().__init__('offboard_control')

        # QoS cho PX4 (best effort, volatile)
        qos_profile = QoSProfile(
            reliability=ReliabilityPolicy.BEST_EFFORT,
            durability=DurabilityPolicy.VOLATILE,
            history=HistoryPolicy.KEEP_LAST,
            depth=1
        )

        # Publishers
        self.offboard_mode_pub = self.create_publisher(
            OffboardControlMode, '/fmu/in/offboard_control_mode', qos_profile)
        self.trajectory_pub = self.create_publisher(
            TrajectorySetpoint, '/fmu/in/trajectory_setpoint', qos_profile)
        self.vehicle_command_pub = self.create_publisher(
            VehicleCommand, '/fmu/in/vehicle_command', qos_profile)

        # Subscribers
        self.odometry_sub = self.create_subscription(
            VehicleOdometry, '/fmu/out/vehicle_odometry',
            self.odometry_callback, qos_profile)

        # Timer gửi setpoint 20Hz
        self.timer = self.create_timer(0.05, self.timer_callback)

        # State
        self.offboard_setpoint_counter = 0
        self.current_pose = None
        self.armed = False
        self.offboard_mode = False

        self.get_logger().info('OffboardControl node started')

    def odometry_callback(self, msg):
        self.current_pose = msg

    def timer_callback(self):
        # Bước 1: Publish OffboardControlMode liên tục
        self.publish_offboard_control_mode()

        # Bước 2: Publish TrajectorySetpoint
        self.publish_trajectory_setpoint()

        # Bước 3: Arm và chuyển offboard mode sau một số chu kỳ
        if self.offboard_setpoint_counter < 11:
            self.offboard_setpoint_counter += 1

        if self.offboard_setpoint_counter == 10:
            self.get_logger().info('Switching to Offboard mode and arming...')
            self.engage_offboard_mode()
            self.arm()

    def publish_offboard_control_mode(self):
        msg = OffboardControlMode()
        msg.position = True      # Điều khiển theo vị trí
        msg.velocity = False
        msg.acceleration = False
        msg.attitude = False
        msg.body_rate = False
        msg.timestamp = int(self.get_clock().now().nanoseconds / 1000)
        self.offboard_mode_pub.publish(msg)

    def publish_trajectory_setpoint(self):
        msg = TrajectorySetpoint()
        # Setpoint: hover tại (0, 0, -5) — NED frame, z âm là lên cao
        msg.position = [0.0, 0.0, -5.0]
        msg.velocity = [float('nan'), float('nan'), float('nan')]
        msg.acceleration = [float('nan'), float('nan'), float('nan')]
        msg.yaw = 0.0
        msg.yawspeed = 0.0
        msg.timestamp = int(self.get_clock().now().nanoseconds / 1000)
        self.trajectory_pub.publish(msg)

    def arm(self):
        """Gửi lệnh arm."""
        msg = VehicleCommand()
        msg.param1 = 1.0          # ARM
        msg.param2 = 0.0
        msg.command = 400         # VEHICLE_CMD_COMPONENT_ARM_DISARM
        msg.target_system = 1
        msg.target_component = 1
        msg.source_system = 1
        msg.source_component = 1
        msg.from_external = True
        msg.timestamp = int(self.get_clock().now().nanoseconds / 1000)
        self.vehicle_command_pub.publish(msg)
        self.get_logger().info('Arm command sent')

    def disarm(self):
        """Gửi lệnh disarm."""
        msg = VehicleCommand()
        msg.param1 = 0.0          # DISARM
        msg.command = 400
        msg.target_system = 1
        msg.target_component = 1
        msg.source_system = 1
        msg.source_component = 1
        msg.from_external = True
        msg.timestamp = int(self.get_clock().now().nanoseconds / 1000)
        self.vehicle_command_pub.publish(msg)
        self.get_logger().info('Disarm command sent')

    def engage_offboard_mode(self):
        """Chuyển sang chế độ OFFBOARD."""
        msg = VehicleCommand()
        msg.param1 = 1.0
        msg.param2 = 6.0          # OFFBOARD mode
        msg.command = 176         # VEHICLE_CMD_DO_SET_MODE
        msg.target_system = 1
        msg.target_component = 1
        msg.source_system = 1
        msg.source_component = 1
        msg.from_external = True
        msg.timestamp = int(self.get_clock().now().nanoseconds / 1000)
        self.vehicle_command_pub.publish(msg)
        self.get_logger().info('Offboard mode command sent')

    def land(self):
        """Gửi lệnh land."""
        msg = VehicleCommand()
        msg.command = 21            # VEHICLE_CMD_NAV_LAND
        msg.param1 = 0.0
        msg.param2 = 0.0
        msg.param3 = 0.0
        msg.param4 = 0.0
        msg.param5 = 0.0
        msg.param6 = 0.0
        msg.param7 = 0.0
        msg.target_system = 1
        msg.target_component = 1
        msg.source_system = 1
        msg.source_component = 1
        msg.from_external = True
        msg.timestamp = int(self.get_clock().now().nanoseconds / 1000)
        self.vehicle_command_pub.publish(msg)
        self.get_logger().info('Land command sent')


def main(args=None):
    rclpy.init(args=args)
    node = OffboardControl()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        node.get_logger().info('Keyboard interrupt, shutting down...')
        node.disarm()
    finally:
        node.destroy_node()
        rclpy.shutdown()


if __name__ == '__main__':
    main()
```

## 4. Sequence điều khiển chuẩn

```
1. Publish OffboardControlMode + TrajectorySetpoint (liên tục, >2Hz)
         |
         v
2. Sau ~0.5s (10 chu kỳ @ 20Hz), gửi VehicleCommand:
   - DO_SET_MODE -> OFFBOARD
   - COMPONENT_ARM_DISARM -> ARM
         |
         v
3. Drone cất cánh và đến setpoint
         |
         v
4. Khi hoàn thành mission, gửi:
   - NAV_LAND (hoặc setpoint z=0)
   - COMPONENT_ARM_DISARM -> DISARM
```

## 5. Các lệnh CLI test

```bash
# Xem trạng thái drone
ros2 topic echo /fmu/out/vehicle_status

# Xem odometry
ros2 topic echo /fmu/out/vehicle_odometry

# Kiểm tra các topic PX4
ros2 topic list | grep fmu

# Gửi arm thủ công (thay vì từ code)
ros2 topic pub /fmu/in/vehicle_command px4_msgs/msg/VehicleCommand \
  "{command: 400, param1: 1.0, target_system: 1, target_component: 1, from_external: true}"

# Gửi offboard mode thủ công
ros2 topic pub /fmu/in/vehicle_command px4_msgs/msg/VehicleCommand \
  "{command: 176, param1: 1.0, param2: 6.0, target_system: 1, target_component: 1, from_external: true}"
```

## 6. Tóm tắt

| Thành phần | Chức năng |
|------------|-----------|
| `OffboardControlMode` | Báo PX4 đang nhận setpoint từ bên ngoài |
| `TrajectorySetpoint` | Gửi vị trí/vận tốc/góc mong muốn (NED frame) |
| `VehicleCommand` | Gửi lệnh cấp cao (arm, mode, land) |
| `vehicle_odometry` | Nhận pose và velocity từ EKF2 |
| Tần số > 2Hz | Yêu cầu bắt buộc để giữ offboard mode |

---

**Bài tiếp theo:** [Bài 34: Drone Simulation](../34.drone-simulation/drone-simulation.md)
