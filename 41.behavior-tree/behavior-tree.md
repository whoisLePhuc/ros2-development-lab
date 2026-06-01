# Bài 41: Behavior Tree

## 1. Behavior Tree là gì?

**Behavior Tree (BT)** là một cách tổ chức logic điều khiển robot, thay thế cho **Finite State Machine (FSM)**.

### So sánh BT vs FSM

| Đặc điểm | FSM | Behavior Tree |
|----------|-----|---------------|
| Cấu trúc | Các state chuyển đổi qua transition | Cây các node với root ở trên cùng |
| Mở rộng | Thêm state = sửa nhiều transition | Thêm node = thêm nhánh trên cây |
| Tái sử dụng | Khó (state liên kết chặt) | Dễ (node độc lập) |
| Đọc hiểu | Khó với hệ thống lớn | Dễ (cây có thể đọc từ trên xuống) |
| Hierarchical | Có thể có | Tự nhiên (cây lồng nhau) |

### Ví dụ thực tế

FSM cho robot navigation:
```
Idle → [start] → Planning → [success] → Moving → [arrived] → Idle
                    ↓ [fail]                    ↓ [obstacle]
                 Recovery ←──────────────────────┘
```

BT cho cùng bài toán:
```
Sequence
├── CheckBattery
├── Sequence
│   ├── ComputePathToPose
│   └── FollowPath
└── Recovery
    ├── ClearCostmap
    └── ComputePathToPose
```

## 2. Các loại node trong Behavior Tree

### Control Nodes (điều khiển luồng)

| Node | Ký hiệu | Hoạt động |
|------|---------|-----------|
| **Sequence** | `→` | Chạy con từ trái sang phải. Nếu một con FAIL, dừng và trả về FAIL. Nếu tất cả SUCCESS, trả về SUCCESS. |
| **Fallback** (Selector) | `?` | Chạy con từ trái sang phải. Nếu một con SUCCESS, dừng và trả về SUCCESS. Nếu tất cả FAIL, trả về FAIL. |
| **Parallel** | `⇉` | Chạy tất cả con đồng thời. Trả về SUCCESS/FAIL dựa trên policy (thường cần N con SUCCESS). |

### Execution Nodes (lá cây)

| Node | Trả về | Ví dụ |
|------|--------|-------|
| **Action** | SUCCESS / FAIL / RUNNING | `MoveToGoal`, `PickObject` |
| **Condition** | SUCCESS / FAIL | `IsBatteryLow?`, `IsPathClear?` |

### Decorator Nodes (bọc một node)

| Node | Tác dụng |
|------|----------|
| **Inverter** | Đảo ngược kết quả (SUCCESS → FAIL, FAIL → SUCCESS) |
| **Retry** | Thử lại N lần nếu FAIL |
| **Repeat** | Lặp lại N lần |
| **Timeout** | Trả về FAIL nếu con chạy quá lâu |
| **ForceSuccess** | Bất kể con trả gì, luôn trả SUCCESS |

### Trạng thái node

Mỗi node trả về một trong 3 trạng thái:
- **SUCCESS**: Nhiệm vụ hoàn thành.
- **FAIL**: Nhiệm vụ thất bại.
- **RUNNING**: Nhiệm vụ đang thực hiện (chưa xong, sẽ tiếp tục ở tick tiếp theo).

## 3. Nav2 Behavior Tree

Nav2 (Navigation2) sử dụng Behavior Tree để điều khiển robot di chuyển.

### Ví dụ BT XML trong Nav2

```xml
<root main_tree_to_execute="MainTree">
  <BehaviorTree ID="MainTree">
    <Sequence name="NavigateWithReplanning">
      <!-- Kiểm tra điều kiện -->
      <IsBatteryLow battery_topic="/battery" min_battery="0.2"/>

      <!-- Compute path đến goal -->
      <ComputePathToPose goal="{goal}" path="{path}" planner_id="GridBased"/>

      <!-- Theo dõi path -->
      <FollowPath path="{path}" controller_id="FollowPath"/>

      <!-- Nếu bị kẹt, chạy recovery -->
      <Fallback name="RecoveryFallback">
        <Sequence name="ClearAndReplan">
          <ClearEntireCostmap name="ClearGlobalCostmap" service_name="/global_costmap/clear_entirely"/>
          <ClearEntireCostmap name="ClearLocalCostmap" service_name="/local_costmap/clear_entirely"/>
          <ComputePathToPose goal="{goal}" path="{path}" planner_id="GridBased"/>
          <FollowPath path="{path}" controller_id="FollowPath"/>
        </Sequence>
        <Spin spin_dist="1.57"/>
        <Wait wait_duration="5"/>
      </Fallback>
    </Sequence>
  </BehaviorTree>
</root>
```

### Các node Nav2 phổ biến

| Node | Loại | Chức năng |
|------|------|-----------|
| `ComputePathToPose` | Action | Tính toán path từ vị trí hiện tại đến goal |
| `FollowPath` | Action | Điều khiển robot theo path |
| `Spin` | Action | Xoay robot một góc |
| `Wait` | Action | Đợi một khoảng thời gian |
| `BackUp` | Action | Lùi robot một khoảng |
| `ClearEntireCostmap` | Action | Xóa costmap |
| `IsBatteryLow` | Condition | Kiểm tra pin |
| `GoalReached` | Condition | Kiểm tra đã đến đích |
| `RateController` | Decorator | Giới hạn tần suất tick |

### Recovery behaviors trong Nav2

Khi robot bị kẹt, Nav2 chạy các hành vi recovery theo thứ tự:

1. **Clear costmap**: Xóa vùng nghi ngờ có vật cản.
2. **Spin**: Xoay robot để quét lại môi trường.
3. **BackUp**: Lùi lại một đoạn.
4. **Wait**: Đợi một lúc rồi thử lại.

```xml
<RecoveryNode number_of_retries="6" name="NavigateRecovery">
  <Sequence name="Navigation">
    <ComputePathToPose goal="{goal}" path="{path}"/>
    <FollowPath path="{path}"/>
  </Sequence>
  <Sequence name="RecoveryActions">
    <ClearEntireCostmap service_name="/local_costmap/clear_entirely"/>
    <Spin spin_dist="1.57"/>
    <BackUp backup_dist="0.15" backup_speed="0.025"/>
    <Wait wait_duration="5.0"/>
  </Sequence>
</RecoveryNode>
```

## 4. Viết Custom Behavior Tree trong Python

Thư viện `py_trees` là implementation Behavior Tree phổ biến trong Python.

### Cài đặt

```bash
pip install py-trees
```

### Ví dụ: Robot patrol với py_trees

```python
import py_trees
import random
import time

# ========== Action Nodes ==========

class MoveToWaypoint(py_trees.behaviour.Behaviour):
    def __init__(self, name, waypoint):
        super().__init__(name)
        self.waypoint = waypoint
        self.start_time = None

    def update(self):
        if self.start_time is None:
            self.start_time = time.time()
            print(f"  [MoveToWaypoint] Moving to {self.waypoint}...")
            return py_trees.common.Status.RUNNING

        # Giả lập di chuyển mất 2 giây
        if time.time() - self.start_time < 2.0:
            return py_trees.common.Status.RUNNING

        print(f"  [MoveToWaypoint] Arrived at {self.waypoint}")
        self.start_time = None
        return py_trees.common.Status.SUCCESS

class CheckBattery(py_trees.behaviour.Behaviour):
    def __init__(self, name, threshold=20.0):
        super().__init__(name)
        self.threshold = threshold

    def update(self):
        battery = random.uniform(10, 100)  # Giả lập pin
        if battery < self.threshold:
            print(f"  [CheckBattery] LOW: {battery:.1f}%")
            return py_trees.common.Status.FAILURE
        print(f"  [CheckBattery] OK: {battery:.1f}%")
        return py_trees.common.Status.SUCCESS

class ChargeBattery(py_trees.behaviour.Behaviour):
    def __init__(self, name):
        super().__init__(name)
        self.start_time = None

    def update(self):
        if self.start_time is None:
            self.start_time = time.time()
            print("  [ChargeBattery] Charging...")
            return py_trees.common.Status.RUNNING

        if time.time() - self.start_time < 3.0:
            return py_trees.common.Status.RUNNING

        print("  [ChargeBattery] Fully charged")
        self.start_time = None
        return py_trees.common.Status.SUCCESS

class DetectObstacle(py_trees.behaviour.Behaviour):
    def update(self):
        obstacle = random.random() < 0.3  # 30% xác suất có vật cản
        if obstacle:
            print("  [DetectObstacle] OBSTACLE DETECTED")
            return py_trees.common.Status.FAILURE
        print("  [DetectObstacle] Path clear")
        return py_trees.common.Status.SUCCESS

class AvoidObstacle(py_trees.behaviour.Behaviour):
    def update(self):
        print("  [AvoidObstacle] Avoiding...")
        time.sleep(1.0)
        return py_trees.common.Status.SUCCESS

# ========== Xây dựng cây ==========

def create_patrol_tree():
    """Tạo cây BT cho nhiệm vụ patrol."""

    # Root: Sequence chính
    root = py_trees.composites.Sequence(name="PatrolMission", memory=True)

    # Nhánh 1: Kiểm tra pin
    battery_check = CheckBattery(name="CheckBattery", threshold=20.0)

    # Nhánh 2: Nếu pin thấp → sạc (Fallback)
    charge_fallback = py_trees.composites.Selector(name="BatteryFallback")
    charge_fallback.add_children([
        battery_check,
        ChargeBattery(name="ChargeBattery")
    ])

    # Nhánh 3: Di chuyển qua các waypoint
    patrol_sequence = py_trees.composites.Sequence(name="PatrolSequence", memory=True)
    for i, wp in enumerate(["A", "B", "C"]):
        # Mỗi waypoint: kiểm tra vật cản → tránh nếu có → di chuyển
        waypoint_fallback = py_trees.composites.Selector(name=f"Waypoint{wp}Fallback")
        waypoint_sequence = py_trees.composites.Sequence(name=f"Waypoint{wp}Seq", memory=True)

        waypoint_sequence.add_children([
            DetectObstacle(name=f"DetectObstacle{wp}"),
            MoveToWaypoint(name=f"MoveTo{wp}", waypoint=wp)
        ])

        waypoint_fallback.add_children([
            waypoint_sequence,
            AvoidObstacle(name=f"AvoidObstacle{wp}")
        ])

        patrol_sequence.add_child(waypoint_fallback)

    # Gắn tất cả vào root
    root.add_children([charge_fallback, patrol_sequence])

    return root

# ========== Chạy ==========

def main():
    tree = create_patrol_tree()
    tree.setup_with_descendants()

    print("\n=== Behavior Tree Patrol ===\n")
    print(py_trees.display.unicode_tree(tree))

    # Tick cây liên tục
    for tick in range(20):
        print(f"\n--- Tick {tick} ---")
        tree.tick_once()
        time.sleep(0.5)

if __name__ == '__main__':
    main()
```

### Output mẫu

```
=== Behavior Tree Patrol ===

PatrolMission [
├── BatteryFallback ?
│   ├── CheckBattery ✓
│   └── ChargeBattery ✗
└── PatrolSequence →
    ├── WaypointAFallback ?
    │   ├── WaypointASeq →
    │   │   ├── DetectObstacle ✓
    │   │   └── MoveToWaypoint [RUNNING]
    │   └── AvoidObstacle ✗
    ...
```

## 5. Tích hợp py_trees với ROS2

```python
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist
import py_trees

class ROS2BehaviorTree(Node):
    def __init__(self):
        super().__init__('bt_ros2_node')
        self.cmd_vel_pub = self.create_publisher(Twist, 'cmd_vel', 10)
        self.tree = self.create_tree()
        self.timer = self.create_timer(0.1, self.tick_tree)

    def create_tree(self):
        # Tạo cây BT tương tự ví dụ trên
        root = py_trees.composites.Sequence(name="ROS2Patrol", memory=True)
        # ... thêm node
        return py_trees.trees.BehaviourTree(root)

    def tick_tree(self):
        self.tree.tick()
        # Các action node sẽ publish cmd_vel khi RUNNING

def main():
    rclpy.init()
    node = ROS2BehaviorTree()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

## 6. Tóm tắt

| Khái niệm | Ý nghĩa |
|-----------|---------|
| Sequence (`→`) | Chạy con lần lượt, dừng nếu một con FAIL |
| Fallback (`?`) | Chạy con lần lượt, dừng nếu một con SUCCESS |
| Parallel (`⇉`) | Chạy tất cả con đồng thời |
| Action | Node thực hiện nhiệm vụ (có thể RUNNING) |
| Condition | Node kiểm tra điều kiện (SUCCESS/FAIL ngay) |
| Decorator | Bọc node để thay đổi hành vi (Retry, Timeout...) |
| `ComputePathToPose` | Nav2: Tính path |
| `FollowPath` | Nav2: Theo path |
| Recovery | Hành vi khi robot bị kẹt |

---

**Bài tiếp theo:** [Bài 42: ROS2 + AI](../42.ros2-ai/ros2-ai.md)
