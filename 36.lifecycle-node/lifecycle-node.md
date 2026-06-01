# Bài 36: Lifecycle Node

## 1. Lifecycle Node là gì?

Trong ROS2, một node thông thường khởi động ngay lập tức và bắt đầu publish/subscribe ngay khi constructor chạy. Điều này có thể gây ra vấn đề khi node cần khởi tạo phần cứng, kết nối mạng, hoặc load cấu hình trước khi hoạt động.

**Lifecycle Node** giải quyết vấn đề này bằng cách định nghĩa một máy trạng thái rõ ràng. Node chỉ thực sự "chạy" khi chuyển sang trạng thái **Active**. Trước đó, bạn có thể chuẩn bị tài nguyên, kiểm tra lỗi, và tắt node một cách an toàn.

## 2. Các trạng thái Lifecycle

Lifecycle Node có 4 trạng thái chính và các chuyển tiếp giữa chúng:

```
Unconfigured → Configuring → Inactive → Activating → Active
     ↑              ↓            ↑           ↓         ↓
     └────────── Finalized ←────┘    Deactivating ←────┘
```

| Trạng thái | Mô tả |
|-----------|-------|
| **Unconfigured** | Node vừa được tạo, chưa có tài nguyên nào được cấp phát. |
| **Inactive** | Tài nguyên đã được khởi tạo (mở serial, load config), nhưng node chưa xử lý dữ liệu. |
| **Active** | Node đang hoạt động đầy đủ: publish, subscribe, timer chạy. |
| **Finalized** | Node đang dọn dẹp và chuẩn bị bị hủy. Không thể quay lại. |

### Khi nào nên dùng Lifecycle Node?

- **Khởi tạo phần cứng an toàn**: Mở cổng serial, kết nối camera, khởi động motor driver trước khi bắt đầu xử lý.
- **Startup/Shutdown có kiểm soát**: Đảm bảo dữ liệu không bị mất khi tắt node.
- **Hệ thống lớn với nhiều node**: Khởi động tuần tự theo thứ tự phụ thuộc.
- **Tiết kiệm tài nguyên**: Chuyển node sang Inactive thay vì kill process khi tạm dừng.

## 3. Các callback Lifecycle

Khi chuyển trạng thái, ROS2 gọi các callback tương ứng. Bạn override chúng để thực hiện logic:

| Callback | Khi nào gọi | Nên làm gì |
|----------|------------|-----------|
| `on_configure` | Unconfigured → Inactive | Khởi tạo tài nguyên: mở serial, tạo publisher/subscriber (nhưng chưa bắt đầu timer). |
| `on_activate` | Inactive → Active | Bắt đầu timer, bật callback chính, node bắt đầu xử lý. |
| `on_deactivate` | Active → Inactive | Dừng timer, tạm dừng xử lý, giữ tài nguyên. |
| `on_cleanup` | Inactive → Unconfigured | Đóng serial, hủy publisher/subscriber, giải phóng bộ nhớ. |
| `on_shutdown` | Bất kỳ → Finalized | Dọn dẹp khẩn cấp, đóng tất cả kết nối. |

## 4. Ví dụ: LifecycleSensor với cổng Serial

Giả sử bạn có một cảm biến kết nối qua cổng UART. Bạn muốn mở cổng khi configure, bắt đầu đọc dữ liệu khi activate, và đóng cổng khi cleanup.

### Package setup

```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_cmake lifecycle_sensor --dependencies rclcpp rclcpp_lifecycle std_msgs
```

### lifecycle_sensor.cpp

```cpp
#include <memory>
#include <string>

#include "rclcpp/rclcpp.hpp"
#include "rclcpp_lifecycle/lifecycle_node.hpp"
#include "rclcpp_lifecycle/lifecycle_publisher.hpp"
#include "std_msgs/msg/float32.hpp"

using namespace std::chrono_literals;

class LifecycleSensor : public rclcpp_lifecycle::LifecycleNode
{
public:
  explicit LifecycleSensor(const rclcpp::NodeOptions & options)
  : rclcpp_lifecycle::LifecycleNode("lifecycle_sensor", options)
  {
    RCLCPP_INFO(get_logger(), "LifecycleSensor created in Unconfigured state");
  }

  // Callback 1: Configure - mở serial, tạo publisher
  rclcpp_lifecycle::node_interfaces::LifecycleNodeInterface::CallbackReturn
  on_configure(const rclcpp_lifecycle::State & previous_state) override
  {
    RCLCPP_INFO(get_logger(), "Configuring...");

    // Giả lập mở cổng serial
    serial_port_ = "/dev/ttyUSB0";
    baud_rate_ = 115200;
    RCLCPP_INFO(get_logger(), "Opened serial port %s at %d baud", serial_port_.c_str(), baud_rate_);

    // Tạo publisher nhưng chưa publish
    publisher_ = this->create_publisher<std_msgs::msg::Float32>("sensor_data", 10);

    RCLCPP_INFO(get_logger(), "Configured successfully");
    return rclcpp_lifecycle::node_interfaces::LifecycleNodeInterface::CallbackReturn::SUCCESS;
  }

  // Callback 2: Activate - bắt đầu timer, bắt đầu đọc dữ liệu
  rclcpp_lifecycle::node_interfaces::LifecycleNodeInterface::CallbackReturn
  on_activate(const rclcpp_lifecycle::State & previous_state) override
  {
    RCLCPP_INFO(get_logger(), "Activating...");

    // Kích hoạt publisher
    publisher_->on_activate();

    // Bắt đầu timer đọc dữ liệu
    timer_ = this->create_wall_timer(
      500ms, std::bind(&LifecycleSensor::read_sensor, this));

    RCLCPP_INFO(get_logger(), "Activated successfully");
    return rclcpp_lifecycle::node_interfaces::LifecycleNodeInterface::CallbackReturn::SUCCESS;
  }

  // Callback 3: Deactivate - dừng timer, tạm dừng đọc dữ liệu
  rclcpp_lifecycle::node_interfaces::LifecycleNodeInterface::CallbackReturn
  on_deactivate(const rclcpp_lifecycle::State & previous_state) override
  {
    RCLCPP_INFO(get_logger(), "Deactivating...");

    // Hủy timer
    timer_.reset();

    // Vô hiệu hóa publisher
    publisher_->on_deactivate();

    RCLCPP_INFO(get_logger(), "Deactivated successfully");
    return rclcpp_lifecycle::node_interfaces::LifecycleNodeInterface::CallbackReturn::SUCCESS;
  }

  // Callback 4: Cleanup - đóng serial, hủy publisher
  rclcpp_lifecycle::node_interfaces::LifecycleNodeInterface::CallbackReturn
  on_cleanup(const rclcpp_lifecycle::State & previous_state) override
  {
    RCLCPP_INFO(get_logger(), "Cleaning up...");

    // Đóng serial
    RCLCPP_INFO(get_logger(), "Closed serial port %s", serial_port_.c_str());
    serial_port_.clear();

    // Hủy publisher
    publisher_.reset();

    RCLCPP_INFO(get_logger(), "Cleaned up successfully");
    return rclcpp_lifecycle::node_interfaces::LifecycleNodeInterface::CallbackReturn::SUCCESS;
  }

  // Callback 5: Shutdown - dọn dẹp khẩn cấp
  rclcpp_lifecycle::node_interfaces::LifecycleNodeInterface::CallbackReturn
  on_shutdown(const rclcpp_lifecycle::State & previous_state) override
  {
    RCLCPP_INFO(get_logger(), "Shutting down from state %s...", previous_state.label().c_str());

    // Dừng timer nếu đang chạy
    timer_.reset();

    // Đóng serial nếu đang mở
    if (!serial_port_.empty()) {
      RCLCPP_INFO(get_logger(), "Emergency close serial port %s", serial_port_.c_str());
      serial_port_.clear();
    }

    // Hủy publisher
    publisher_.reset();

    RCLCPP_INFO(get_logger(), "Shutdown successfully");
    return rclcpp_lifecycle::node_interfaces::LifecycleNodeInterface::CallbackReturn::SUCCESS;
  }

private:
  void read_sensor()
  {
    // Giả lập đọc dữ liệu từ cảm biến
    auto msg = std_msgs::msg::Float32();
    msg.data = 25.0f + static_cast<float>(rand()) / RAND_MAX * 5.0f;  // 25-30 độ C
    RCLCPP_INFO(get_logger(), "Publishing: %.2f", msg.data);
    publisher_->publish(msg);
  }

  std::string serial_port_;
  int baud_rate_;
  rclcpp::TimerBase::SharedPtr timer_;
  rclcpp_lifecycle::LifecyclePublisher<std_msgs::msg::Float32>::SharedPtr publisher_;
};

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);

  rclcpp::executors::SingleThreadedExecutor exe;
  std::shared_ptr<LifecycleSensor> node = std::make_shared<LifecycleSensor>(rclcpp::NodeOptions());

  exe.add_node(node->get_node_base_interface());
  exe.spin();

  rclcpp::shutdown();
  return 0;
}
```

### CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.8)
project(lifecycle_sensor)

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(rclcpp_lifecycle REQUIRED)
find_package(std_msgs REQUIRED)

add_executable(lifecycle_sensor_node src/lifecycle_sensor.cpp)
ament_target_dependencies(lifecycle_sensor_node rclcpp rclcpp_lifecycle std_msgs)

install(TARGETS lifecycle_sensor_node
  DESTINATION lib/${PROJECT_NAME})

ament_package()
```

### Build và chạy

```bash
cd ~/ros2_ws
colcon build --packages-select lifecycle_sensor
source install/setup.bash
ros2 run lifecycle_sensor lifecycle_sensor_node
```

## 5. Điều khiển Lifecycle qua CLI

Lifecycle Node không tự chuyển trạng thái. Bạn phải ra lệnh từ bên ngoài.

### Xem trạng thái hiện tại

```bash
ros2 lifecycle list /lifecycle_sensor
```

Output:
```
Unconfigured [1]
    start [2]
Inactive [3]
    activate [4]
    cleanup [5]
Active [6]
    deactivate [7]
Finalized [8]
    destroy [9]
```

### Chuyển trạng thái

```bash
# Unconfigured → Inactive
ros2 lifecycle set /lifecycle_sensor configure

# Inactive → Active
ros2 lifecycle set /lifecycle_sensor activate

# Active → Inactive
ros2 lifecycle set /lifecycle_sensor deactivate

# Inactive → Unconfigured
ros2 lifecycle set /lifecycle_sensor cleanup

# Bất kỳ → Finalized
ros2 lifecycle set /lifecycle_sensor shutdown
```

### Kiểm tra trạng thái

```bash
ros2 lifecycle get /lifecycle_sensor
```

## 6. Launch Lifecycle Node

Bạn có thể tự động chuyển trạng thái trong launch file bằng `LifecycleNode` action và `EmitEvent`.

### lifecycle_launch.py

```python
from launch import LaunchDescription
from launch.actions import EmitEvent
from launch_ros.actions import LifecycleNode
from launch_ros.events.lifecycle import ChangeState
from lifecycle_msgs.msg import Transition

def generate_launch_description():
    # Tạo Lifecycle Node
    lifecycle_node = LifecycleNode(
        package='lifecycle_sensor',
        executable='lifecycle_sensor_node',
        name='lifecycle_sensor',
        namespace='',
        output='screen'
    )

    # Tự động chuyển sang Configure sau khi launch
    configure_event = EmitEvent(
        event=ChangeState(
            lifecycle_node_matcher=lambda node: node == lifecycle_node,
            transition_id=Transition.TRANSITION_CONFIGURE,
        )
    )

    # Tự động chuyển sang Activate sau khi Configure thành công
    activate_event = EmitEvent(
        event=ChangeState(
            lifecycle_node_matcher=lambda node: node == lifecycle_node,
            transition_id=Transition.TRANSITION_ACTIVATE,
        )
    )

    return LaunchDescription([
        lifecycle_node,
        configure_event,
        activate_event,
    ])
```

Chạy launch:

```bash
ros2 launch lifecycle_sensor lifecycle_launch.py
```

## 7. Lưu ý quan trọng

- **Publisher trong Lifecycle**: Dùng `rclcpp_lifecycle::LifecyclePublisher` thay vì `rclcpp::Publisher`. Nó tự động chặn publish khi node ở trạng thái Inactive.
- **Callback phải trả về SUCCESS hoặc FAILURE**: Nếu `on_configure` trả về FAILURE, node ở lại Unconfigured.
- **Không dùng `rclcpp::spin` trực tiếp**: Lifecycle Node cần `SingleThreadedExecutor` và `add_node(node->get_node_base_interface())`.
- **Tên topic vẫn hiển thị khi Inactive**: Publisher đã tồn tại nhưng không publish. Điều này giúp các node khác biết topic sẽ sớm có dữ liệu.

## 8. Tóm tắt

| Khái niệm | Ý nghĩa |
|-----------|---------|
| Unconfigured | Node mới tạo, chưa có tài nguyên |
| Inactive | Tài nguyên sẵn sàng, đang chờ lệnh activate |
| Active | Node đang chạy đầy đủ |
| Finalized | Node đang bị hủy |
| `on_configure` | Mở tài nguyên, tạo publisher/subscriber |
| `on_activate` | Bắt đầu timer, kích hoạt publisher |
| `on_deactivate` | Dừng timer, vô hiệu hóa publisher |
| `on_cleanup` | Đóng tài nguyên, hủy publisher/subscriber |
| `ros2 lifecycle set` | CLI để chuyển trạng thái |

---

**Bài tiếp theo:** [Bài 37: Component Node](../37.component-node/component-node.md)
