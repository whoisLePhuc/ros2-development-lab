# Bài 40: MoveIt2

## 1. MoveIt2 là gì?

**MoveIt2** là framework motion planning cho robot arms trong ROS2. Nó cung cấp:

- **Motion Planning**: Tính toán quỹ đạo từ vị trí A đến B, tránh va chạm.
- **Kinematics**: Tính toán forward/inverse kinematics.
- **Collision Checking**: Kiểm tra va chạm với môi trường và chính robot.
- **Trajectory Execution**: Gửi lệnh đến controller để thực thi.

MoveIt2 hỗ trợ hầu hết các robot arm phổ biến: UR5, Franka, Kinova, Fanuc, ABB...

## 2. Kiến trúc MoveIt2

```
User / Application
    └── MoveGroup (C++ / Python API)
            ├── Planning Pipeline (OMPL, CHOMP, STOMP...)
            ├── Kinematics Solver (KDL, TRAC-IK, BioIK...)
            ├── Collision Checking (FCL, Bullet)
            └── Trajectory Execution Manager
                    └── Controller (ros2_control)
                            └── Hardware
```

### Các thành phần chính

| Thành phần | Vai trò |
|-----------|---------|
| **MoveGroup** | Node trung tâm, nhận request từ user, phối hợp các thành phần khác. |
| **Planning Pipeline** | Chứa planner plugin (OMPL, Pilz). Tính toán quỹ đạo. |
| **Kinematics** | Tính inverse kinematics: từ vị trí end-effector → góc khớp. |
| **Collision Checking** | Kiểm tra va chạm giữa robot, vật thể, và môi trường. |
| **Trajectory Execution** | Gửi trajectory đến controller, theo dõi execution. |

### OMPL (Open Motion Planning Library)

OMPL là planner mặc định. Nó cung cấp nhiều thuật toán:

- **RRT (Rapidly-exploring Random Tree)**: Nhanh, phù hợp không gian lớn.
- **RRTConnect**: Bi-directional RRT, nhanh hơn cho nhiều bài toán.
- **PRM (Probabilistic Roadmap)**: Phù hợp multi-query (nhiều điểm đến).
- **CHOMP**: Gradient-based, tối ưu quỹ đạo mượt.

### Kinematics Solvers

| Solver | Đặc điểm |
|--------|---------|
| **KDL** | Mặc định, đa năng, chậm hơn cho robot 7 DOF. |
| **TRAC-IK** | Nhanh hơn KDL, hỗ trợ constraints. |
| **BioIK** | Tối ưu cho bài toán có nhiều mục tiêu (vị trí + hướng). |

### Collision Checking (FCL)

FCL (Flexible Collision Library) kiểm tra va chạm bằng cách xây dựng **Octree** và **Bounding Volume Hierarchy**.

## 3. Cài đặt MoveIt2

```bash
# Cài đặt MoveIt2 và dependencies
sudo apt install ros-humble-moveit
sudo apt install ros-humble-moveit-resources
sudo apt install ros-humble-ros2-control ros-humble-ros2-controllers
```

## 4. MoveIt Setup Assistant

Setup Assistant giúp bạn tạo cấu hình MoveIt cho robot mà không cần viết file YAML thủ công.

### Bước 1: Mở Setup Assistant

```bash
ros2 launch moveit_setup_assistant setup_assistant.launch.py
```

### Bước 2: Load URDF

- Chọn "Create New MoveIt Configuration Package".
- Load file URDF của robot (ví dụ: `panda.urdf`).

### Bước 3: Tự động generate

Setup Assistant sẽ tự động:
1. **Self-Collisions**: Tính toán các cặp link không bao giờ va chạm (tối ưu hóa).
2. **Planning Groups**: Định nghĩa nhóm khớp (ví dụ: `panda_arm` gồm 7 khớp).
3. **Robot Poses**: Định nghĩa các pose đặc biệt (home, ready, pick).
4. **End Effector**: Định nghĩa gripper.
5. **Passive Joints**: Các khớp không điều khiển.
6. **Controllers**: Liên kết với ros2_control controllers.
7. **Simulation**: Tích hợp Gazebo.

### Bước 4: Generate package

- Chọn thư mục output (ví dụ: `~/ros2_ws/src/my_robot_moveit_config`).
- Nhấn "Generate Package".

### Bước 5: Build

```bash
cd ~/ros2_ws
colcon build --packages-select my_robot_moveit_config
source install/setup.bash
```

## 5. Python API: Điều khiển robot arm

### Khởi tạo MoveIt2

```python
import rclpy
from rclpy.node import Node
from moveit.planning import MoveItPy
from moveit.core.kinematic_constraints import construct_joint_constraint
from geometry_msgs.msg import PoseStamped
import numpy as np

class MoveItController(Node):
    def __init__(self):
        super().__init__('moveit_controller')

        # Khởi tạo MoveIt2
        self.moveit = MoveItPy(node_name="moveit_py")
        self.robot = self.moveit.get_planning_component("panda_arm")

        self.get_logger().info("MoveIt2 initialized")

    def move_to_joint_goal(self, joint_values):
        """Di chuyển đến cấu hình khớp cụ thể."""
        self.robot.set_start_state_to_current_state()

        # Thiết lập goal
        self.robot.set_goal_state(joint_values=joint_values)

        # Plan
        plan_result = self.robot.plan()

        if plan_result:
            self.get_logger().info("Planning succeeded, executing...")
            self.robot.execute()
        else:
            self.get_logger().error("Planning failed")

    def move_to_pose_goal(self, target_pose: PoseStamped):
        """Di chuyển end-effector đến pose cụ thể."""
        self.robot.set_start_state_to_current_state()

        # Thiết lập goal là pose
        self.robot.set_goal_state(pose_stamped_msg=target_pose, pose_link="panda_link8")

        # Plan với OMPL
        plan_result = self.robot.plan()

        if plan_result:
            self.get_logger().info("Planning succeeded, executing...")
            self.robot.execute()
        else:
            self.get_logger().error("Planning failed")

def main():
    rclpy.init()

    # Cần pass args cho MoveItPy
    import sys
    node = MoveItController()

    # Ví dụ: Di chuyển đến pose
    target = PoseStamped()
    target.header.frame_id = "panda_link0"
    target.pose.position.x = 0.5
    target.pose.position.y = 0.0
    target.pose.position.z = 0.5
    target.pose.orientation.w = 1.0

    node.move_to_pose_goal(target)

    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

### Plan và Execute riêng biệt

```python
# Plan trước, kiểm tra, rồi execute
plan_result = self.robot.plan()

if plan_result:
    trajectory = plan_result.trajectory
    self.get_logger().info(f"Trajectory duration: {trajectory.duration} seconds")

    # Kiểm tra trajectory trước khi execute
    if trajectory.duration < 10.0:  # Chỉ execute nếu dưới 10 giây
        self.robot.execute()
    else:
        self.get_logger().warn("Trajectory too long, aborting")
```

### Thêm collision object

```python
from moveit.core.planning_scene import PlanningScene
from shape_msgs.msg import SolidPrimitive

# Tạo collision object
collision_object = SolidPrimitive()
collision_object.type = SolidPrimitive.BOX
collision_object.dimensions = [0.1, 0.1, 0.5]  # Box 10x10x50 cm

# Thêm vào planning scene
self.moveit.get_planning_scene().add_object(
    id="obstacle",
    pose=obstacle_pose,
    primitives=[collision_object]
)
```

## 6. Chạy demo với Panda Robot

```bash
# Terminal 1: Khởi động MoveIt demo (RViz + robot)
ros2 launch moveit_resources_panda_moveit_config demo.launch.py

# Terminal 2: Chạy Python script điều khiển
ros2 run my_pkg moveit_controller
```

Trong RViz:
- Kéo interactive marker để đặt goal pose.
- Nhấn "Plan" để xem quỹ đạo.
- Nhấn "Execute" để robot di chuyển.

## 7. Tích hợp với ros2_control

MoveIt2 không gửi lệnh trực tiếp đến hardware. Nó gửi trajectory đến **ros2_control** controller:

```
MoveIt2 → Trajectory Execution Manager
    └── JointTrajectoryController (ros2_control)
            └── Hardware Interface
                    └── Robot Motors
```

Cấu hình trong `controllers.yaml`:

```yaml
controller_manager:
  ros__parameters:
    update_rate: 100  # Hz

    joint_trajectory_controller:
      type: joint_trajectory_controller/JointTrajectoryController

    joint_state_broadcaster:
      type: joint_state_broadcaster/JointStateBroadcaster

joint_trajectory_controller:
  ros__parameters:
    joints:
      - panda_joint1
      - panda_joint2
      - panda_joint3
      - panda_joint4
      - panda_joint5
      - panda_joint6
      - panda_joint7
    command_interfaces:
      - position
    state_interfaces:
      - position
      - velocity
```

## 8. Tóm tắt

| Khái niệm | Ý nghĩa |
|-----------|---------|
| MoveGroup | Node trung tâm điều phối motion planning |
| OMPL | Thư viện planning mặc định (RRT, PRM...) |
| KDL / TRAC-IK | Kinematics solver |
| FCL | Collision checking library |
| Setup Assistant | Tool tạo config MoveIt từ URDF |
| `plan()` | Tính toán quỹ đạo |
| `execute()` | Thực thi quỹ đạo |
| ros2_control | Lớp điều khiển hardware cho MoveIt |

---

**Bài tiếp theo:** [Bài 41: Behavior Tree](../41.behavior-tree/behavior-tree.md)
