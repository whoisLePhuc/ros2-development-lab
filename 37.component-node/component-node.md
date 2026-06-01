# Bài 37: Component Node

## 1. Vấn đề: Mỗi node = một process

Trong ROS2, khi bạn chạy `ros2 run my_pkg my_node`, mỗi node là một process độc lập. Nếu hệ thống của bạn có 10 node, bạn sẽ có 10 process.

Vấn đề:
- **Tốn bộ nhớ**: Mỗi process có overhead của OS (stack, heap, file descriptors).
- **Chậm do IPC**: Node A publish message, Node B nhận qua shared memory hoặc socket. Dù ROS2 dùng zero-copy khi có thể, việc chuyển đổi giữa các process vẫn có chi phí.
- **Khó đồng bộ**: Khó đảm bảo các node khởi động đúng thứ tự.

**Component Node** giải quyết vấn đề này bằng cách cho phép nhiều node chạy trong **cùng một process**.

## 2. Component Node là gì?

Component (hay composable node) là một node được biên dịch thành **shared library** (.so) thay vì executable. Bạn load nhiều component vào một **container** (một process duy nhất).

Lợi ích:
- **Zero-copy thực sự**: Các node trong cùng process chia sẻ bộ nhớ. Message không cần serialize/deserialize.
- **Giảm overhead**: Một process thay vì nhiều process.
- **Khởi động nhanh**: Load library nhanh hơn fork process.
- **Dễ quản lý**: Container quản lý vòng đời của tất cả component bên trong.

## 3. Viết Component trong C++

Component trong C++ sử dụng macro `RCLCPP_COMPONENTS_REGISTER_NODE`.

### Package setup

```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_cmake my_components --dependencies rclcpp rclcpp_components component_manager std_msgs
```

### talker_component.cpp

```cpp
#include "rclcpp/rclcpp.hpp"
#include "rclcpp_components/register_node_macro.hpp"
#include "std_msgs/msg/string.hpp"

#include <chrono>

using namespace std::chrono_literals;

namespace my_components
{

class TalkerComponent : public rclcpp::Node
{
public:
  explicit TalkerComponent(const rclcpp::NodeOptions & options)
  : Node("talker_component", options)
  {
    publisher_ = this->create_publisher<std_msgs::msg::String>("chatter", 10);
    timer_ = this->create_wall_timer(
      1s, std::bind(&TalkerComponent::timer_callback, this));
    RCLCPP_INFO(this->get_logger(), "TalkerComponent loaded");
  }

private:
  void timer_callback()
  {
    auto msg = std_msgs::msg::String();
    msg.data = "Hello from component: " + std::to_string(count_++);
    RCLCPP_INFO(this->get_logger(), "Publishing: '%s'", msg.data.c_str());
    publisher_->publish(msg);
  }

  rclcpp::Publisher<std_msgs::msg::String>::SharedPtr publisher_;
  rclcpp::TimerBase::SharedPtr timer_;
  size_t count_ = 0;
};

}  // namespace my_components

// Đăng ký component với plugin system
RCLCPP_COMPONENTS_REGISTER_NODE(my_components::TalkerComponent)
```

### listener_component.cpp

```cpp
#include "rclcpp/rclcpp.hpp"
#include "rclcpp_components/register_node_macro.hpp"
#include "std_msgs/msg/string.hpp"

namespace my_components
{

class ListenerComponent : public rclcpp::Node
{
public:
  explicit ListenerComponent(const rclcpp::NodeOptions & options)
  : Node("listener_component", options)
  {
    subscription_ = this->create_subscription<std_msgs::msg::String>(
      "chatter",
      10,
      std::bind(&ListenerComponent::topic_callback, this, std::placeholders::_1));
    RCLCPP_INFO(this->get_logger(), "ListenerComponent loaded");
  }

private:
  void topic_callback(const std_msgs::msg::String::SharedPtr msg)
  {
    RCLCPP_INFO(this->get_logger(), "I heard: '%s'", msg->data.c_str());
  }

  rclcpp::Subscription<std_msgs::msg::String>::SharedPtr subscription_;
};

}  // namespace my_components

RCLCPP_COMPONENTS_REGISTER_NODE(my_components::ListenerComponent)
```

### CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.8)
project(my_components)

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(rclcpp_components REQUIRED)
find_package(std_msgs REQUIRED)
find_package(component_manager REQUIRED)

# Tạo shared library cho Talker
add_library(talker_component SHARED src/talker_component.cpp)
ament_target_dependencies(talker_component rclcpp rclcpp_components std_msgs)
rclcpp_components_register_nodes(talker_component "my_components::TalkerComponent")

# Tạo shared library cho Listener
add_library(listener_component SHARED src/listener_component.cpp)
ament_target_dependencies(listener_component rclcpp rclcpp_components std_msgs)
rclcpp_components_register_nodes(listener_component "my_components::ListenerComponent")

# Cài đặt libraries
install(TARGETS talker_component listener_component
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin)

ament_package()
```

### Build

```bash
cd ~/ros2_ws
colcon build --packages-select my_components
source install/setup.bash
```

## 4. Chạy Component với Container

### Cách 1: Dùng component_container executable

```bash
# Terminal 1: Khởi động container
ros2 run rclcpp_components component_container

# Terminal 2: Load Talker vào container
ros2 component load /ComponentManager my_components my_components::TalkerComponent

# Terminal 3: Load Listener vào container
ros2 component load /ComponentManager my_components my_components::ListenerComponent
```

### Cách 2: Dùng component_container_isolated

Mỗi component chạy trong thread riêng (giống multi-threaded executor):

```bash
ros2 run rclcpp_components component_container_isolated
```

### Cách 3: Dùng launch file (khuyến nghị)

```python
from launch import LaunchDescription
from launch_ros.actions import ComposableNodeContainer
from launch_ros.descriptions import ComposableNode

def generate_launch_description():
    container = ComposableNodeContainer(
        name='my_container',
        namespace='',
        package='rclcpp_components',
        executable='component_container',
        composable_node_descriptions=[
            ComposableNode(
                package='my_components',
                plugin='my_components::TalkerComponent',
                name='talker'),
            ComposableNode(
                package='my_components',
                plugin='my_components::ListenerComponent',
                name='listener'),
        ],
        output='screen',
    )

    return LaunchDescription([container])
```

Chạy:

```bash
ros2 launch my_components components_launch.py
```

## 5. Component trong Python

Python không hỗ trợ component theo cùng cách C++ (vì Python không biên dịch thành shared library). Tuy nhiên, bạn có thể dùng `rclpy` với `MultiThreadedExecutor` để chạy nhiều node trong cùng process.

### multi_node_process.py

```python
import rclpy
from rclpy.executors import MultiThreadedExecutor
from rclpy.node import Node
from std_msgs.msg import String

class TalkerNode(Node):
    def __init__(self):
        super().__init__('talker_py')
        self.pub = self.create_publisher(String, 'chatter_py', 10)
        self.timer = self.create_timer(1.0, self.timer_cb)
        self.count = 0

    def timer_cb(self):
        msg = String()
        msg.data = f'Hello from Python: {self.count}'
        self.get_logger().info(f'Publishing: {msg.data}')
        self.pub.publish(msg)
        self.count += 1

class ListenerNode(Node):
    def __init__(self):
        super().__init__('listener_py')
        self.sub = self.create_subscription(
            String, 'chatter_py', self.listener_cb, 10)

    def listener_cb(self, msg):
        self.get_logger().info(f'I heard: {msg.data}')

def main():
    rclpy.init()

    talker = TalkerNode()
    listener = ListenerNode()

    executor = MultiThreadedExecutor()
    executor.add_node(talker)
    executor.add_node(listener)

    try:
        executor.spin()
    except KeyboardInterrupt:
        pass

    talker.destroy_node()
    listener.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

Lưu ý: Python không có zero-copy giữa các node trong cùng process như C++. Message vẫn được serialize qua DDS.

## 6. Unload và quản lý Component

### Liệt kê component trong container

```bash
ros2 component list
```

### Unload component

```bash
# Xem UID của component
ros2 component list

# Unload theo UID
ros2 component unload /ComponentManager 1 2
```

### Thay đổi parameter khi load

```bash
ros2 component load /ComponentManager my_components my_components::TalkerComponent \
  --param frequency:=2.0 --param topic_name:="my_topic"
```

## 7. So sánh: Node thường vs Component

| Đặc điểm | Node thường | Component |
|----------|------------|-----------|
| Số process | Mỗi node 1 process | Nhiều node 1 process |
| Zero-copy | Không (IPC qua DDS) | Có (shared memory) |
| Khởi động | Fork process | Load library |
| Bộ nhớ | Cao | Thấp |
| Debug | Dễ (tách biệt) | Khó hơn (cùng process) |
| Crash isolation | Tốt (node A crash không ảnh hưởng B) | Kém (container crash = tất cả die) |
| Ngôn ngữ | C++, Python | Chủ yếu C++ |

## 8. Khi nào dùng Component?

- **Dùng Component khi**:
  - Các node trao đổi dữ liệu lớn (pointcloud, image) thường xuyên.
  - Hệ thống có nhiều node nhỏ liên quan chặt chẽ (pipeline xử lý ảnh).
  - Cần tối ưu latency và throughput.

- **Dùng Node thường khi**:
  - Các node độc lập, ít liên quan đến nhau.
  - Cần crash isolation (node A crash không được ảnh hưởng B).
  - Viết bằng Python.

## 9. Tóm tắt

| Khái niệm | Ý nghĩa |
|-----------|---------|
| Component | Node biên dịch thành shared library |
| Container | Process chứa nhiều component |
| `RCLCPP_COMPONENTS_REGISTER_NODE` | Macro đăng ký component |
| `component_container` | Executable chạy container |
| `ComposableNodeContainer` | Launch action tạo container |
| Zero-copy | Chia sẻ bộ nhớ giữa node trong cùng process |

---

**Bài tiếp theo:** [Bài 38: DDS Deep Dive](../38.dds-deep-dive/dds-deep-dive.md)
