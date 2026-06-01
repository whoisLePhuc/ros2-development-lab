# Bài 7: Action — Goal, Feedback, Result

## 1. Action là gì?

**Action** là mô hình giao tiếp trong ROS2 dành cho các **tác vụ dài hạn, cần theo dõi tiến trình**:

- **Goal**: Client gửi mục tiêu cần đạt được
- **Feedback**: Server gửi cập nhật tiến trình trong khi thực hiện
- **Result**: Server gửi kết quả cuối cùng khi hoàn tất

Action phù hợp cho: di chuyển đến điểm đích, quét bản đồ, nhặt vật thể, hoặc bất kỳ tác vụ nào mất nhiều thời gian.

## 2. Cấu trúc file .action

File `.action` có ba phần ngăn cách bởi `---`:

```action
# Goal
int32 order
---
# Result
int32[] sequence
---
# Feedback
int32[] partial_sequence
```

Ví dụ trên là Fibonacci action: goal là số thứ tự, feedback là dãy tạm thời, result là dãy hoàn chỉnh.

## 3. Action Server — Thực hiện tác vụ

### Ví dụ: Fibonacci Action Server

```python
import rclpy
from rclpy.node import Node
from rclpy.action import ActionServer
from rclpy.action.server import ServerGoalHandle
from my_action_interfaces.action import Fibonacci


class FibonacciActionServer(Node):
    def __init__(self):
        super().__init__('fibonacci_action_server')
        self.action_server = ActionServer(
            self,
            Fibonacci,
            'fibonacci',
            self.execute_callback,
            goal_callback=self.goal_callback,
            cancel_callback=self.cancel_callback
        )
        self.get_logger().info('Fibonacci Action Server đã sẵn sàng')

    def goal_callback(self, goal_request):
        # Chấp nhận hoặc từ chối goal
        self.get_logger().info(f'Nhận goal: order={goal_request.order}')
        if goal_request.order < 0:
            self.get_logger().warn('Từ chối goal: order phải >= 0')
            return rclpy.action.GoalResponse.REJECT
        return rclpy.action.GoalResponse.ACCEPT

    def cancel_callback(self, goal_handle):
        # Chấp nhận hoặc từ chối yêu cầu hủy
        self.get_logger().info('Nhận yêu cầu hủy goal')
        return rclpy.action.CancelResponse.ACCEPT

    def execute_callback(self, goal_handle: ServerGoalHandle):
        order = goal_handle.request.order
        self.get_logger().info(f'Bắt đầu tính Fibonacci đến order {order}')

        feedback_msg = Fibonacci.Feedback()
        feedback_msg.partial_sequence = [0, 1]

        for i in range(1, order):
            if goal_handle.is_cancel_requested:
                goal_handle.canceled()
                self.get_logger().info('Goal đã bị hủy')
                return Fibonacci.Result()

            feedback_msg.partial_sequence.append(
                feedback_msg.partial_sequence[i] + feedback_msg.partial_sequence[i - 1]
            )
            self.get_logger().info(f'Feedback: {feedback_msg.partial_sequence}')
            goal_handle.publish_feedback(feedback_msg)
            # Giả lập tác vụ mất thời gian
            import time
            time.sleep(1)

        goal_handle.succeed()
        result = Fibonacci.Result()
        result.sequence = feedback_msg.partial_sequence
        self.get_logger().info(f'Hoàn tất: {result.sequence}')
        return result


def main(args=None):
    rclpy.init(args=args)
    node = FibonacciActionServer()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

### Giải thích

- `ActionServer()`: Tạo action server với kiểu `Fibonacci`, tên `fibonacci`
- `goal_callback()`: Kiểm tra và quyết định chấp nhận/từ chối goal
- `cancel_callback()`: Xử lý yêu cầu hủy từ client
- `execute_callback()`: Thực hiện tác vụ chính, gửi feedback và trả về result
- `publish_feedback()`: Gửi cập nhật tiến trình cho client
- `succeed()`: Đánh dấu goal hoàn thành thành công

## 4. Action Client — Gửi goal và nhận kết quả

### Ví dụ: Fibonacci Action Client

```python
import rclpy
from rclpy.node import Node
from rclpy.action import ActionClient
from my_action_interfaces.action import Fibonacci
import sys


class FibonacciActionClient(Node):
    def __init__(self):
        super().__init__('fibonacci_action_client')
        self.action_client = ActionClient(self, Fibonacci, 'fibonacci')

    def send_goal(self, order):
        # Chờ server sẵn sàng
        self.get_logger().info('Đang chờ action server...')
        if not self.action_client.wait_for_server(timeout_sec=10.0):
            self.get_logger().error('Không tìm thấy action server')
            return

        goal_msg = Fibonacci.Goal()
        goal_msg.order = order

        self.get_logger().info(f'Gửi goal: order={order}')
        self.send_goal_future = self.action_client.send_goal_async(
            goal_msg,
            feedback_callback=self.feedback_callback
        )
        self.send_goal_future.add_done_callback(self.goal_response_callback)

    def goal_response_callback(self, future):
        goal_handle = future.result()
        if not goal_handle.accepted:
            self.get_logger().info('Goal bị từ chối')
            return

        self.get_logger().info('Goal được chấp nhận')
        self.get_result_future = goal_handle.get_result_async()
        self.get_result_future.add_done_callback(self.get_result_callback)

    def feedback_callback(self, feedback_msg):
        partial = feedback_msg.feedback.partial_sequence
        self.get_logger().info(f'Nhận feedback: {partial}')

    def get_result_callback(self, future):
        result = future.result().result
        self.get_logger().info(f'Kết quả: {result.sequence}')
        rclpy.shutdown()


def main(args=None):
    rclpy.init(args=args)
    node = FibonacciActionClient()

    if len(sys.argv) != 2:
        node.get_logger().info('Usage: python3 client.py <order>')
        return

    node.send_goal(int(sys.argv[1]))
    rclpy.spin(node)
    node.destroy_node()


if __name__ == '__main__':
    main()
```

### Giải thích

- `ActionClient()`: Tạo action client
- `wait_for_server()`: Chờ action server khởi động
- `send_goal_async()`: Gửi goal không đồng bộ, đăng ký feedback callback
- `feedback_callback()`: Nhận cập nhật tiến trình
- `goal_response_callback()`: Nhận phản hồi chấp nhận/từ chối goal
- `get_result_callback()`: Nhận kết quả cuối cùng

## 5. So sánh Topic, Service và Action

| Đặc điểm | Topic | Service | Action |
|----------|-------|---------|--------|
| Mô hình | Một-to-nhiều | Một-một | Một-một |
| Đồng bộ | Không | Có | Không (long-running) |
| Phản hồi | Không | Một lần | Nhiều lần (feedback) |
| Có thể hủy | Không | Không | Có |
| Phù hợp | Dữ liệu liên tục | Tác vụ ngắn | Tác vụ dài, cần theo dõi |
| Ví dụ | Sensor, camera | Tính toán, reset | Di chuyển, quét bản đồ |

## 6. CLI — Công cụ dòng lệnh cho Action

```bash
# Liệt kê tất cả action đang hoạt động
ros2 action list

# Xem thông tin chi tiết action
ros2 action info /fibonacci

# Gửi goal từ CLI
ros2 action send_goal /fibonacci my_action_interfaces/action/Fibonacci '{order: 10}'

# Gửi goal và hiển thị feedback
ros2 action send_goal /fibonacci my_action_interfaces/action/Fibonacci '{order: 10}' --feedback
```

## 7. Tóm tắt

- **Action** dùng cho tác vụ dài hạn với goal, feedback và result
- **Action Server** cung cấp `execute_callback`, `goal_callback`, `cancel_callback`
- **Action Client** gửi goal bằng `send_goal_async()`, nhận feedback và result qua callback
- Action cho phép **hủy tác vụ** giữa chừng, điều mà Service không làm được
- Chọn **Topic** cho dữ liệu liên tục, **Service** cho tác vụ ngắn, **Action** cho tác vụ dài

---

**Bài tiếp theo:** [Bài 8: Parameter — Cấu hình động](../09.ros2-parameter/ros2-parameter.md)
