# Bài 14: ROS2 Package

A **package** is the smallest unit of build and distribution in ROS2. This lesson compares Python and C++ packages, shows how to create each type, and demonstrates adding a new node to an existing package.

---

## 1. Python vs C++ Package

| Aspect | Python | C++ |
|--------|--------|-----|
| Build type | `ament_python` | `ament_cmake` |
| Build file | `setup.py` / `setup.cfg` | `CMakeLists.txt` |
| Compilation | None (interpreted) | Compiled to binary |
| Entry point | `entry_points` in `setup.py` | `add_executable` + `install` |
| Speed | Slower, great for prototyping | Faster, great for performance |
| Dependencies | `package.xml` + `setup.py` | `package.xml` + `CMakeLists.txt` |

Many projects use both: Python for high-level logic, C++ for drivers and heavy computation.

---

## 2. Create a Python Package

```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_python --node-name talker py_talker
```

This generates:

```
py_talker/
├── package.xml
├── setup.cfg
├── setup.py
├── py_talker/
│   ├── __init__.py
│   └── talker.py
└── resource/
    └── py_talker
```

### setup.py

```python
from setuptools import setup

package_name = 'py_talker'

setup(
    name=package_name,
    version='0.0.1',
    packages=[package_name],
    data_files=[
        ('share/ament_index/resource_index/packages',
            ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
    ],
    install_requires=['setuptools'],
    zip_safe=True,
    maintainer='your_name',
    maintainer_email='your_email@example.com',
    description='A simple talker node',
    license='Apache License 2.0',
    tests_require=['pytest'],
    entry_points={
        'console_scripts': [
            'talker = py_talker.talker:main',
        ],
    },
)
```

### package.xml

```xml
<?xml version="1.0"?>
<?xml-model href="http://download.ros.org/schema/package_format3.xsd" schematypens="http://www.w3.org/2001/XMLSchema"?>
<package format="3">
  <name>py_talker</name>
  <version>0.0.1</version>
  <description>A simple talker node</description>
  <maintainer email="your_email@example.com">your_name</maintainer>
  <license>Apache License 2.0</license>

  <depend>rclpy</depend>
  <depend>std_msgs</depend>

  <test_depend>ament_copyright</test_depend>
  <test_depend>ament_flake8</test_depend>
  <test_depend>ament_pep257</test_depend>
  <test_depend>ament_pytest</test_depend>

  <export>
    <build_type>ament_python</build_type>
  </export>
</package>
```

### talker.py

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from std_msgs.msg import String

class Talker(Node):
    def __init__(self):
        super().__init__('talker')
        self.pub = self.create_publisher(String, 'chatter', 10)
        self.timer = self.create_timer(1.0, self.timer_callback)
        self.count = 0

    def timer_callback(self):
        msg = String()
        msg.data = f'Hello ROS2: {self.count}'
        self.pub.publish(msg)
        self.get_logger().info(f'Publishing: "{msg.data}"')
        self.count += 1

def main(args=None):
    rclpy.init(args=args)
    node = Talker()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

Build and run:

```bash
cd ~/ros2_ws
colcon build --packages-select py_talker --symlink-install
source install/setup.bash
ros2 run py_talker talker
```

---

## 3. Create a C++ Package

```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_cmake --node-name listener cpp_listener
```

This generates:

```
cpp_listener/
├── CMakeLists.txt
├── package.xml
└── src/
    └── listener.cpp
```

### CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.8)
project(cpp_listener)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)

add_executable(listener src/listener.cpp)
ament_target_dependencies(listener rclcpp std_msgs)

install(TARGETS
  listener
  DESTINATION lib/${PROJECT_NAME})

ament_package()
```

### package.xml

```xml
<?xml version="1.0"?>
<?xml-model href="http://download.ros.org/schema/package_format3.xsd" schematypens="http://www.w3.org/2001/XMLSchema"?>
<package format="3">
  <name>cpp_listener</name>
  <version>0.0.1</version>
  <description>A simple listener node</description>
  <maintainer email="your_email@example.com">your_name</maintainer>
  <license>Apache License 2.0</license>

  <buildtool_depend>ament_cmake</buildtool_depend>
  <depend>rclcpp</depend>
  <depend>std_msgs</depend>

  <test_depend>ament_lint_auto</test_depend>
  <test_depend>ament_lint_common</test_depend>

  <export>
    <build_type>ament_cmake</build_type>
  </export>
</package>
```

### listener.cpp

```cpp
#include <memory>
#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"

class Listener : public rclcpp::Node
{
public:
  Listener()
  : Node("listener")
  {
    sub_ = this->create_subscription<std_msgs::msg::String>(
      "chatter", 10,
      [this](const std_msgs::msg::String::SharedPtr msg) {
        RCLCPP_INFO(this->get_logger(), "I heard: '%s'", msg->data.c_str());
      });
  }

private:
  rclcpp::Subscription<std_msgs::msg::String>::SharedPtr sub_;
};

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<Listener>());
  rclcpp::shutdown();
  return 0;
}
```

Build and run:

```bash
cd ~/ros2_ws
colcon build --packages-select cpp_listener
source install/setup.bash
ros2 run cpp_listener listener
```

---

## 4. Add a New Node to an Existing Package

### Python: add a listener

Create `py_talker/py_talker/listener.py`:

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from std_msgs.msg import String

class Listener(Node):
    def __init__(self):
        super().__init__('listener')
        self.sub = self.create_subscription(
            String, 'chatter', self.listener_callback, 10)

    def listener_callback(self, msg):
        self.get_logger().info(f'I heard: "{msg.data}"')

def main(args=None):
    rclpy.init(args=args)
    node = Listener()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

Edit `setup.py` entry points:

```python
entry_points={
    'console_scripts': [
        'talker = py_talker.talker:main',
        'listener = py_talker.listener:main',
    ],
},
```

Rebuild:

```bash
colcon build --packages-select py_talker --symlink-install
```

### C++: add a talker

Create `cpp_listener/src/talker.cpp`:

```cpp
#include <memory>
#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"

class Talker : public rclcpp::Node
{
public:
  Talker()
  : Node("talker"), count_(0)
  {
    pub_ = this->create_publisher<std_msgs::msg::String>("chatter", 10);
    timer_ = this->create_wall_timer(
      std::chrono::seconds(1),
      [this]() {
        auto msg = std_msgs::msg::String();
        msg.data = "Hello ROS2: " + std::to_string(count_++);
        RCLCPP_INFO(this->get_logger(), "Publishing: '%s'", msg.data.c_str());
        pub_->publish(msg);
      });
  }

private:
  rclcpp::Publisher<std_msgs::msg::String>::SharedPtr pub_;
  rclcpp::TimerBase::SharedPtr timer_;
  size_t count_;
};

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<Talker>());
  rclcpp::shutdown();
  return 0;
}
```

Edit `CMakeLists.txt`:

```cmake
add_executable(talker src/talker.cpp)
ament_target_dependencies(talker rclcpp std_msgs)

install(TARGETS
  listener
  talker
  DESTINATION lib/${PROJECT_NAME})
```

Rebuild:

```bash
colcon build --packages-select cpp_listener
```

---

## 5. Talker-Listener Package Example

A complete `talker_listener` package often contains both nodes:

```
talker_listener/
├── package.xml
├── setup.py
├── talker_listener/
│   ├── __init__.py
│   ├── talker.py
│   └── listener.py
└── resource/
    └── talker_listener
```

Run both in separate terminals:

```bash
ros2 run talker_listener talker
ros2 run talker_listener listener
```

---

## 6. Key Takeaways

- Python packages use `ament_python`, `setup.py`, and `entry_points`.
- C++ packages use `ament_cmake`, `CMakeLists.txt`, and `add_executable`.
- You can add multiple nodes to one package. Just register each entry point.
- `--symlink-install` makes Python development faster.

---

## Bài tiếp theo

[Bài 15: Launch File](../15.launch-file/launch-file.md)
