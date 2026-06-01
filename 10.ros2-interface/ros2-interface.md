# Bài 9: Interface — Message, Service, Action

ROS2 nodes communicate through three interface types: **Message** (pub/sub), **Service** (request/response), and **Action** (goal/feedback/result). This lesson shows how to define your own interfaces and use them from Python code.

---

## 1. Why Custom Interfaces Matter

Built-in types like `std_msgs/String` or `geometry_msgs/Twist` cover common cases. Real robots need domain-specific data: a temperature sensor reading, a target temperature setpoint, or a navigation goal with progress updates. Custom interfaces keep your code clean and self-documenting.

**Convention in ROS2:** always place interfaces in a **dedicated package**. This lets any node, Python or C++, depend on the same definitions without circular dependencies.

---

## 2. Create an Interface Package

```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_cmake my_robot_interfaces
cd my_robot_interfaces
```

Remove the default `src/` folder. Interface packages only need:

```
my_robot_interfaces/
├── msg/
│   └── Temperature.msg
├── srv/
│   └── SetTarget.srv
├── action/
│   └── Navigate.action
├── CMakeLists.txt
└── package.xml
```

---

## 3. Define a Message (.msg)

Create `msg/Temperature.msg`:

```
float64 value
string unit
int32 sensor_id
```

Each line is `type field_name`. Supported types mirror ROS2 built-in types: `int8` through `int64`, `uint8` through `uint64`, `float32`, `float64`, `string`, `bool`, plus arrays like `float64[]` and fixed arrays like `float64[3]`.

---

## 4. Define a Service (.srv)

Create `srv/SetTarget.srv`:

```
float64 target_temperature
string unit
---
bool success
string message
```

The `---` separates the **request** (top) from the **response** (bottom).

---

## 5. Define an Action (.action)

Create `action/Navigate.action`:

```
geometry_msgs/PoseStamped target_pose
---
geometry_msgs/PoseStamped final_pose
---
float32 percent_complete
string status
```

The action format has three sections separated by `---`:

1. **Goal** (what the client sends)
2. **Result** (what the server sends when done)
3. **Feedback** (what the server streams during execution)

---

## 6. Build the Package

### package.xml

Add these dependencies:

```xml
<buildtool_depend>ament_cmake</buildtool_depend>
<build_depend>rosidl_default_generators</build_depend>
<exec_depend>rosidl_default_runtime</exec_depend>
<member_of_group>rosidl_interface_packages</member_of_group>

<!-- Only needed if your .action references other packages -->
<depend>geometry_msgs</depend>
```

### CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.8)
project(my_robot_interfaces)

find_package(ament_cmake REQUIRED)
find_package(rosidl_default_generators REQUIRED)
find_package(geometry_msgs REQUIRED)

rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/Temperature.msg"
  "srv/SetTarget.srv"
  "action/Navigate.action"
  DEPENDENCIES geometry_msgs
)

ament_package()
```

The `rosidl_generate_interfaces` macro tells the build system to generate Python and C++ code from your `.msg`, `.srv`, and `.action` files.

### Build

```bash
cd ~/ros2_ws
colcon build --packages-select my_robot_interfaces
source install/setup.bash
```

---

## 7. Use Interfaces from Python

### Publisher with custom message

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from my_robot_interfaces.msg import Temperature

class TempPublisher(Node):
    def __init__(self):
        super().__init__('temp_publisher')
        self.pub = self.create_publisher(Temperature, 'temperature', 10)
        self.timer = self.create_timer(1.0, self.publish_temp)
        self.sensor_id = 1

    def publish_temp(self):
        msg = Temperature()
        msg.value = 25.5
        msg.unit = 'celsius'
        msg.sensor_id = self.sensor_id
        self.pub.publish(msg)
        self.get_logger().info(f'Published: {msg.value} {msg.unit}')

def main(args=None):
    rclpy.init(args=args)
    node = TempPublisher()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

### Service server with custom service

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from my_robot_interfaces.srv import SetTarget

class TempController(Node):
    def __init__(self):
        super().__init__('temp_controller')
        self.srv = self.create_service(SetTarget, 'set_target', self.set_target_callback)
        self.current_target = 22.0

    def set_target_callback(self, request, response):
        self.current_target = request.target_temperature
        response.success = True
        response.message = f'Target set to {self.current_target} {request.unit}'
        self.get_logger().info(response.message)
        return response

def main(args=None):
    rclpy.init(args=args)
    node = TempController()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

### Action server with custom action

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from rclpy.action import ActionServer
from my_robot_interfaces.action import Navigate
import time

class NavigateServer(Node):
    def __init__(self):
        super().__init__('navigate_server')
        self._action_server = ActionServer(
            self,
            Navigate,
            'navigate',
            self.execute_callback
        )

    def execute_callback(self, goal_handle):
        self.get_logger().info('Executing navigation goal...')
        feedback_msg = Navigate.Feedback()
        result_msg = Navigate.Result()

        for i in range(1, 11):
            feedback_msg.percent_complete = float(i * 10)
            feedback_msg.status = 'navigating'
            goal_handle.publish_feedback(feedback_msg)
            time.sleep(0.5)

        goal_handle.succeed()
        result_msg.final_pose = goal_handle.request.target_pose
        return result_msg

def main(args=None):
    rclpy.init(args=args)
    node = NavigateServer()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

---

## 8. Verify with CLI

```bash
# List all interfaces in your package
ros2 interface package my_robot_interfaces

# Show message definition
ros2 interface show my_robot_interfaces/msg/Temperature

# Show service definition
ros2 interface show my_robot_interfaces/srv/SetTarget

# Show action definition
ros2 interface show my_robot_interfaces/action/Navigate
```

---

## 9. Key Takeaways

- Keep interfaces in a **separate, dedicated package**.
- Use `rosidl_generate_interfaces` in `CMakeLists.txt`.
- Reference other message packages (like `geometry_msgs`) in `DEPENDENCIES`.
- Generated code is imported as `from my_robot_interfaces.msg import Temperature`.

---

## Bài tiếp theo

[Bài 10: QoS — Quality of Service](../11.ros2-qos/ros2-qos.md)
