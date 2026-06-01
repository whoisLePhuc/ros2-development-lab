# Bài 27: Hardware Interface

Hardware Interface là cầu nối giữa `ros2_control` và phần cứng thật. Nó chuyển đổi lệnh từ controller (velocity, position, effort) thành tín hiệu vật lý gửi xuống motor driver. Ngược lại, nó đọc encoder, IMU, hoặc các sensor khác và đưa lên ROS2.

## 1. Hardware Interface là gì?

Khi bạn dùng Gazebo, plugin `gazebo_ros2_control/GazeboSystem` đóng vai trò Hardware Interface. Nó đọc/ghi trực tiếp vào physics engine.

Khi chạy robot thật, bạn phải tự viết Hardware Interface. Nó thường giao tiếp với MCU (STM32, Arduino) qua UART, CAN, hoặc Ethernet.

```
Controller Manager
      |
      v
Hardware Interface (C++ plugin)
      |
      v
Serial / CAN / Ethernet
      |
      v
MCU (STM32 / Arduino)
      |
      v
Motor Driver + Encoder
```

## 2. Viết SystemInterface bằng C++

Bạn cần kế thừa lớp `hardware_interface::SystemInterface` và override 4 phương thức chính.

### Header file

```cpp
// include/my_robot_hardware/my_robot_hardware.hpp
#ifndef MY_ROBOT_HARDWARE_HPP
#define MY_ROBOT_HARDWARE_HPP

#include <hardware_interface/system_interface.hpp>
#include <rclcpp/rclcpp.hpp>
#include <vector>
#include <string>
#include <serial/serial.h>

namespace my_robot_hardware
{

class MyRobotHardware : public hardware_interface::SystemInterface
{
public:
  // Callbacks lifecycle
  hardware_interface::CallbackReturn on_init(const hardware_interface::HardwareInfo & info) override;
  hardware_interface::CallbackReturn on_configure(const rclcpp_lifecycle::State & previous_state) override;
  hardware_interface::CallbackReturn on_activate(const rclcpp_lifecycle::State & previous_state) override;
  hardware_interface::CallbackReturn on_deactivate(const rclcpp_lifecycle::State & previous_state) override;

  // Read/write loop
  hardware_interface::return_type read(const rclcpp::Time & time, const rclcpp::Duration & period) override;
  hardware_interface::return_type write(const rclcpp::Time & time, const rclcpp::Duration & period) override;

private:
  // Serial
  std::shared_ptr<serial::Serial> serial_port_;
  std::string port_name_;
  int baudrate_;

  // Joint data
  std::vector<double> hw_positions_;
  std::vector<double> hw_velocities_;
  std::vector<double> hw_commands_;

  // Encoder ticks to radians conversion
  double ticks_per_rev_;
  double wheel_radius_;

  bool parseSensorPacket(const std::string & packet, std::vector<double> & positions, std::vector<double> & velocities);
  std::string buildCommandPacket(const std::vector<double> & commands);
};

} // namespace my_robot_hardware

#endif
```

### Implementation

```cpp
// src/my_robot_hardware.cpp
#include "my_robot_hardware/my_robot_hardware.hpp"
#include <hardware_interface/types/hardware_interface_type_values.hpp>
#include <pluginlib/class_list_macros.hpp>

namespace my_robot_hardware
{

hardware_interface::CallbackReturn MyRobotHardware::on_init(const hardware_interface::HardwareInfo & info)
{
  if (hardware_interface::SystemInterface::on_init(info) != CallbackReturn::SUCCESS)
  {
    return CallbackReturn::ERROR;
  }

  // Đọc tham số từ URDF
  port_name_ = info.hardware_parameters.at("serial_port");
  baudrate_ = std::stoi(info.hardware_parameters.at("baudrate"));
  ticks_per_rev_ = std::stod(info.hardware_parameters.at("ticks_per_rev"));
  wheel_radius_ = std::stod(info.hardware_parameters.at("wheel_radius"));

  // Khởi tạo vector theo số joint
  hw_positions_.resize(info_.joints.size(), 0.0);
  hw_velocities_.resize(info_.joints.size(), 0.0);
  hw_commands_.resize(info_.joints.size(), 0.0);

  return CallbackReturn::SUCCESS;
}

hardware_interface::CallbackReturn MyRobotHardware::on_configure(const rclcpp_lifecycle::State & /*previous_state*/)
{
  try {
    serial_port_ = std::make_shared<serial::Serial>(port_name_, baudrate_, serial::Timeout::simpleTimeout(100));
    RCLCPP_INFO(rclcpp::get_logger("MyRobotHardware"), "Connected to %s at %d baud", port_name_.c_str(), baudrate_);
  } catch (const std::exception & e) {
    RCLCPP_ERROR(rclcpp::get_logger("MyRobotHardware"), "Serial open failed: %s", e.what());
    return CallbackReturn::ERROR;
  }

  return CallbackReturn::SUCCESS;
}

hardware_interface::CallbackReturn MyRobotHardware::on_activate(const rclcpp_lifecycle::State & /*previous_state*/)
{
  // Reset commands
  for (auto & cmd : hw_commands_) {
    cmd = 0.0;
  }
  return CallbackReturn::SUCCESS;
}

hardware_interface::CallbackReturn MyRobotHardware::on_deactivate(const rclcpp_lifecycle::State & /*previous_state*/)
{
  if (serial_port_ && serial_port_->isOpen()) {
    serial_port_->close();
  }
  return CallbackReturn::SUCCESS;
}

hardware_interface::return_type MyRobotHardware::read(const rclcpp::Time & /*time*/, const rclcpp::Duration & /*period*/)
{
  if (!serial_port_ || !serial_port_->isOpen()) {
    return hardware_interface::return_type::ERROR;
  }

  // Đọc dòng từ serial
  std::string line = serial_port_->readline();
  if (line.empty()) {
    return hardware_interface::return_type::OK;
  }

  std::vector<double> positions;
  std::vector<double> velocities;

  if (parseSensorPacket(line, positions, velocities)) {
    for (size_t i = 0; i < hw_positions_.size(); ++i) {
      if (i < positions.size()) hw_positions_[i] = positions[i];
      if (i < velocities.size()) hw_velocities_[i] = velocities[i];
    }
  }

  return hardware_interface::return_type::OK;
}

hardware_interface::return_type MyRobotHardware::write(const rclcpp::Time & /*time*/, const rclcpp::Duration & /*period*/)
{
  if (!serial_port_ || !serial_port_->isOpen()) {
    return hardware_interface::return_type::ERROR;
  }

  std::string packet = buildCommandPacket(hw_commands_);
  serial_port_->write(packet);

  return hardware_interface::return_type::OK;
}

bool MyRobotHardware::parseSensorPacket(const std::string & packet, std::vector<double> & positions, std::vector<double> & velocities)
{
  // Giả sử format: "SENSOR,<left_ticks>,<right_ticks>,<left_vel>,<right_vel>\n"
  // Chuyển ticks -> radian: rad = ticks * 2 * PI / ticks_per_rev
  // Chuyển radian -> linear: v = omega * r

  // Parse logic ở đây...
  return true;
}

std::string MyRobotHardware::buildCommandPacket(const std::vector<double> & commands)
{
  // Giả sử format: "CMD,<left_vel>,<right_vel>\n"
  // commands là velocity (rad/s) hoặc đã được controller chuẩn hóa

  std::string packet = "CMD,";
  for (size_t i = 0; i < commands.size(); ++i) {
    packet += std::to_string(commands[i]);
    if (i < commands.size() - 1) packet += ",";
  }
  packet += "\n";
  return packet;
}

} // namespace my_robot_hardware

PLUGINLIB_EXPORT_CLASS(my_robot_hardware::MyRobotHardware, hardware_interface::SystemInterface)
```

### CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.8)
project(my_robot_hardware)

find_package(ament_cmake REQUIRED)
find_package(hardware_interface REQUIRED)
find_package(pluginlib REQUIRED)
find_package(rclcpp REQUIRED)
find_package(serial REQUIRED)

add_library(my_robot_hardware SHARED
  src/my_robot_hardware.cpp
)

target_include_directories(my_robot_hardware PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>
)

ament_target_dependencies(my_robot_hardware
  hardware_interface
  pluginlib
  rclcpp
  serial
)

pluginlib_export_plugin_description_file(hardware_interface my_robot_hardware.xml)

install(TARGETS my_robot_hardware
  EXPORT export_my_robot_hardware
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin
)

install(DIRECTORY include/
  DESTINATION include
)

ament_package()
```

### Plugin XML

```xml
<!-- my_robot_hardware.xml -->
<library path="my_robot_hardware">
  <class name="my_robot_hardware/MyRobotHardware"
         type="my_robot_hardware::MyRobotHardware"
         base_class_type="hardware_interface::SystemInterface">
    <description>My robot hardware interface via serial.</description>
  </class>
</library>
```

## 3. Giao tiếp Serial với STM32/Arduino

### Protocol design

Thiết kế giao thức đơn giản, dễ parse, có checksum.

```
// Packet từ PC xuống MCU (MotorCommand)
// Format: "CMD,<left_vel>,<right_vel>,<checksum>\n"
// <left_vel>, <right_vel>: float, đơn vị rad/s
// checksum: XOR của các byte payload

// Packet từ MCU lên PC (SensorData)
// Format: "SENSOR,<left_ticks>,<right_ticks>,<left_vel>,<right_vel>,<checksum>\n"
// <left_ticks>, <right_ticks>: int32, tích lũy từ lúc boot
// <left_vel>, <right_vel>: float, rad/s
```

### Arduino code (tóm tắt)

```cpp
// Arduino / STM32 code
#include <Encoder.h>

Encoder encLeft(2, 3);
Encoder encRight(4, 5);

const float TICKS_PER_REV = 1440.0;
const float WHEEL_RADIUS = 0.05;

long lastLeftTicks = 0;
long lastRightTicks = 0;
unsigned long lastTime = 0;

void setup() {
  Serial.begin(115200);
}

void loop() {
  // Đọc encoder
  long leftTicks = encLeft.read();
  long rightTicks = encRight.read();

  unsigned long now = millis();
  float dt = (now - lastTime) / 1000.0;

  if (dt > 0) {
    float leftVel = ((leftTicks - lastLeftTicks) / TICKS_PER_REV) * 2 * PI / dt;
    float rightVel = ((rightTicks - lastRightTicks) / TICKS_PER_REV) * 2 * PI / dt;

    // Gửi lên PC
    Serial.print("SENSOR,");
    Serial.print(leftTicks);
    Serial.print(",");
    Serial.print(rightTicks);
    Serial.print(",");
    Serial.print(leftVel, 4);
    Serial.print(",");
    Serial.print(rightVel, 4);
    Serial.println(",0"); // checksum placeholder

    lastLeftTicks = leftTicks;
    lastRightTicks = rightTicks;
    lastTime = now;
  }

  // Nhận lệnh từ PC
  if (Serial.available()) {
    String cmd = Serial.readStringUntil('\n');
    if (cmd.startsWith("CMD,")) {
      // Parse left_vel, right_vel
      // Gán PWM cho motor
    }
  }

  delay(20); // 50 Hz
}
```

### Chuyển encoder ticks sang joint position

```cpp
// Trong Hardware Interface read()
double ticks_to_rad = 2.0 * M_PI / ticks_per_rev_;
hw_positions_[i] = static_cast<double>(ticks) * ticks_to_rad;
hw_velocities_[i] = static_cast<double>(ticks_delta) * ticks_to_rad / dt;
```

## 4. URDF config với serial params

```xml
<ros2_control name="MyRobotHardware" type="system">
  <hardware>
    <plugin>my_robot_hardware/MyRobotHardware</plugin>
    <param name="serial_port">/dev/ttyUSB0</param>
    <param name="baudrate">115200</param>
    <param name="ticks_per_rev">1440</param>
    <param name="wheel_radius">0.05</param>
  </hardware>

  <joint name="left_wheel_joint">
    <command_interface name="velocity"/>
    <state_interface name="position"/>
    <state_interface name="velocity"/>
  </joint>

  <joint name="right_wheel_joint">
    <command_interface name="velocity"/>
    <state_interface name="position"/>
    <state_interface name="velocity"/>
  </joint>
</ros2_control>
```

## 5. Các loại Hardware Interface

ros2_control hỗ trợ ba loại interface:

| Loại | Mô tả | Khi nào dùng |
|------|-------|--------------|
| `SystemInterface` | Quản lý toàn bộ robot (nhiều joint, nhiều sensor) | Hầu hết các trường hợp. Dùng khi MCU quản lý tất cả. |
| `ActuatorInterface` | Quản lý một actuator đơn lẻ | Dùng khi mỗi joint có driver riêng (ví dụ: EtherCAT servo) |
| `SensorInterface` | Chỉ đọc sensor, không ghi lệnh | Dùng cho sensor độc lập như force/torque sensor |

Hầu hết robot di động và robot arm đều dùng `SystemInterface` vì một MCU xử lý tất cả.

## 6. Debug Hardware Interface

```bash
# Kiểm tra plugin có được load không
ros2 plugin list | grep my_robot_hardware

# Chạy controller_manager với hardware interface
ros2 run controller_manager ros2_control_node --ros-args -p robot_description:=$(xacro robot.urdf.xacro)

# Kiểm tra serial port
ls -la /dev/ttyUSB*
sudo chmod 666 /dev/ttyUSB0

# Test serial bằng screen
screen /dev/ttyUSB0 115200
```

## 7. Tóm tắt

- Hardware Interface là plugin C++ kế thừa `SystemInterface`.
- Override `on_init`, `read`, `write` để kết nối với phần cứng.
- Serial là giao thức đơn giản nhất cho STM32/Arduino.
- Thiết kế protocol rõ ràng, có delimiter và checksum.
- Encoder ticks chuyển sang radian qua `2 * PI / ticks_per_rev`.
- URDF truyền tham số serial qua `<param>` trong `<hardware>`.

---

**Bài tiếp theo:** [Bài 28: Micro-ROS — ROS2 trên Vi điều khiển](../28.micro-ros/micro-ros.md)
