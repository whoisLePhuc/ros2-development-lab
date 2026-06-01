# Bài 32: PX4 Fundamentals

## 1. PX4 là gì?

PX4 là flight control firmware mã nguồn mở cho drone (UAV), tàu lặn, và các phương tiện tự hành khác. Nó chạy trên các vi điều khiển STM32 (như Pixhawk) và cung cấp đầy đủ stack điều khiển từ cảm biến đến actuator.

## 2. Kiến trúc PX4

```
+------------------+
|   Applications   |  (navigator, commander, mavlink)
+------------------+
|    Modules       |  (EKF2, sensors, mc_pos_control)
+------------------+
|   uORB Bus       |  (publish/subscribe internal)
+------------------+
|   Drivers        |  (UART, SPI, I2C, CAN, PWM)
+------------------+
|   Board Support  |  (STM32 HAL, NuttX RTOS)
+------------------+
```

Các thành phần chính:
- **EKF2 (Extended Kalman Filter)**: Ước lượng trạng thái (position, velocity, attitude) từ IMU, GPS, barometer, magnetometer.
- **Mixer**: Chuyển đổi lệnh điều khiển (roll, pitch, yaw, thrust) thành tín hiệu PWM cho từng motor.
- **Navigator**: Quản lý các mission waypoint và flight mode.
- **Commander**: Quản lý trạng thái máy bay (disarmed, armed, takeoff, landing, failsafe).

## 3. Clone và Build PX4-Autopilot

### Yêu cầu hệ thống (Ubuntu 22.04)

```bash
# Cài đặt dependencies
sudo apt update
sudo apt install git zip qtcreator cmake build-essential genromfs ninja-build exiftool
sudo apt install python3-empy python3-toml python3-numpy python3-yaml python3-argparse
sudo apt install python3-dev python3-pip
sudo apt install gazebo libgazebo-dev libgazebo11

# Cài PX4 Python tools
pip3 install --user px4-jinja2 px4-kconfig px4-module-template
```

### Clone repository

```bash
cd ~
git clone https://github.com/PX4/PX4-Autopilot.git --recursive
cd PX4-Autopilot
```

### Build firmware

```bash
# Build cho board Pixhawk (ví dụ: Pixhawk 4 — fmu-v5)
make px4_fmu-v5_default

# Build cho SITL simulation
make px4_sitl_default
```

## 4. Chạy SITL Simulation

PX4 hỗ trợ nhiều simulator. Với Gazebo Classic (Gazebo 11):

```bash
cd ~/PX4-Autopilot
make px4_sitl gazebo-classic
```

Hoặc với Gazebo (Ignition / Gazebo Garden):

```bash
make px4_sitl gz_x500
```

Sau khi khởi động, bạn sẽ thấy:
- Terminal chạy PX4 shell (nsh>).
- Cửa sổ Gazebo mở ra với drone x500.
- QGroundControl có thể kết nối qua UDP port 14540.

## 5. Các topic PX4 quan trọng

PX4 sử dụng uORB (internal) và có thể bridge sang ROS2 qua `px4_ros_com`. Các topic nội bộ quan trọng:

### Input (fmu/in/*)

| Topic | Mô tả |
|-------|-------|
| `fmu/in/vehicle_command` | Lệnh điều khiển cấp cao (arm, disarm, mode switch) |
| `fmu/in/trajectory_setpoint` | Setpoint vị trí/vận tốc cho bộ điều khiển |
| `fmu/in/offboard_control_mode` | Kích hoạt chế độ offboard |
| `fmu/in/manual_control_setpoint` | Input từ RC/joystick |

### Output (fmu/out/*)

| Topic | Mô tả |
|-------|-------|
| `fmu/out/vehicle_odometry` | Pose và velocity ước lượng từ EKF2 |
| `fmu/out/vehicle_status` | Trạng thái máy bay (armed, flight mode, etc.) |
| `fmu/out/battery_status` | Thông tin pin |
| `fmu/out/sensor_combined` | Dữ liệu IMU tổng hợp |
| `fmu/out/vehicle_gps_position` | Vị trí GPS |

## 6. Cài đặt px4_msgs và px4_ros_com

### px4_msgs

Chứa định nghĩa message ROS2 tương ứng với uORB messages của PX4.

```bash
cd ~/ros2_ws/src
git clone https://github.com/PX4/px4_msgs.git
cd ~/ros2_ws
rosdep install --from-paths src --ignore-src -y
colcon build --packages-select px4_msgs
source install/setup.bash
```

### px4_ros_com

Bridge giữa uORB và ROS2, chạy trên companion computer.

```bash
cd ~/ros2_ws/src
git clone https://github.com/PX4/px4_ros_com.git
cd ~/ros2_ws
rosdep install --from-paths src --ignore-src -y
colcon build --packages-select px4_ros_com
source install/setup.bash
```

## 7. Kiểm tra kết nối

Sau khi chạy SITL và bridge:

```bash
# Xem các topic PX4 trên ROS2
ros2 topic list | grep fmu

# Xem odometry
ros2 topic echo /fmu/out/vehicle_odometry

# Xem trạng thái
ros2 topic echo /fmu/out/vehicle_status
```

## 8. Tóm tắt

| Lệnh / Thành phần | Chức năng |
|-------------------|-----------|
| `make px4_fmu-v5_default` | Build firmware cho Pixhawk 4 |
| `make px4_sitl gazebo-classic` | Chạy SITL với Gazebo Classic |
| `px4_msgs` | Định nghĩa ROS2 messages cho PX4 |
| `px4_ros_com` | Bridge uORB <-> ROS2 |
| `fmu/out/vehicle_odometry` | Pose và velocity từ EKF2 |
| `fmu/in/trajectory_setpoint` | Gửi setpoint điều khiển |

---

**Bài tiếp theo:** [Bài 33: ROS2 và PX4](../33.ros2-px4/ros2-px4.md)
