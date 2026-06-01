# Bài 21: URDF — Robot Description

URDF (Unified Robot Description Format) là định dạng XML để mô tả robot trong ROS2. Nó định nghĩa các link (bộ phận vật lý), joint (khớp nối), và hình học của robot.

## 1. Cấu trúc URDF cơ bản

```xml
<?xml version="1.0"?>
<robot name="my_robot">
    <!-- Links -->
    <link name="base_link">
        ...
    </link>

    <!-- Joints -->
    <joint name="base_to_wheel" type="continuous">
        ...
    </joint>
</robot>
```

Mọi URDF đều có:

- Một thẻ `<robot>` gốc với thuộc tính `name`
- Các `<link>` định nghĩa bộ phận
- Các `<joint>` định nghĩa kết nối giữa các link

## 2. Link elements

Mỗi link có 3 thành phần:

### 2.1. Visual

Hình dạng hiển thị trong RViz2/Gazebo.

```xml
<link name="base_link">
    <visual>
        <origin xyz="0 0 0.1" rpy="0 0 0"/>
        <geometry>
            <box size="0.5 0.3 0.1"/>
        </geometry>
        <material name="blue">
            <color rgba="0 0 1 1"/>
        </material>
    </visual>
</link>
```

### 2.2. Collision

Hình dạng dùng cho tính toán va chạm (thường đơn giản hơn visual).

```xml
<link name="base_link">
    <collision>
        <geometry>
            <box size="0.5 0.3 0.1"/>
        </geometry>
    </collision>
</link>
```

### 2.3. Inertial

Khối lượng và momen quán tính, cần cho simulation.

```xml
<link name="base_link">
    <inertial>
        <origin xyz="0 0 0" rpy="0 0 0"/>
        <mass value="5.0"/>
        <inertia ixx="0.1" ixy="0" ixz="0"
                 iyy="0.1" iyz="0"
                 izz="0.1"/>
    </inertial>
</link>
```

## 3. Joint types

| Type | Mô tả | Ví dụ |
|------|-------|-------|
| `fixed` | Không chuyển động | Gắn cảm biến vào thân |
| `continuous` | Xoay vô hạn | Bánh xe |
| `revolute` | Xoay trong giới hạn | Khớp tay robot |
| `prismatic` | Tịnh tiến | Xi lanh thủy lực |
| `floating` | 6 DOF tự do | Drone |
| `planar` | Tịnh tiến 2D | Bàn trượt |

### Ví dụ joint

```xml
<joint name="left_wheel_joint" type="continuous">
    <parent link="base_link"/>
    <child link="left_wheel"/>
    <origin xyz="0 0.2 0" rpy="0 0 0"/>
    <axis xyz="0 1 0"/>
</joint>

<joint name="head_joint" type="revolute">
    <parent link="base_link"/>
    <child link="head"/>
    <origin xyz="0.2 0 0.3" rpy="0 0 0"/>
    <axis xyz="0 0 1"/>
    <limit lower="-1.57" upper="1.57" effort="10" velocity="1.0"/>
</joint>
```

## 4. Geometry types

```xml
<!-- Hình hộp -->
<geometry>
    <box size="width height depth"/>
</geometry>

<!-- Hình trụ -->
<geometry>
    <cylinder radius="0.1" length="0.05"/>
</geometry>

<!-- Hình cầu -->
<geometry>
    <sphere radius="0.05"/>
</geometry>

<!-- File mesh -->
<geometry>
    <mesh filename="package://my_pkg/meshes/base.dae" scale="1 1 1"/>
</geometry>
```

## 5. Material và Color

Định nghĩa material ở đầu file để tái sử dụng:

```xml
<robot name="my_robot">
    <material name="blue">
        <color rgba="0 0 0.8 1"/>
    </material>
    <material name="black">
        <color rgba="0 0 0 1"/>
    </material>
    <material name="white">
        <color rgba="1 1 1 1"/>
    </material>

    <link name="base_link">
        <visual>
            <geometry>
                <box size="0.5 0.3 0.1"/>
            </geometry>
            <material name="blue"/>
        </visual>
    </link>
</robot>
```

## 6. robot_state_publisher

Node này đọc URDF và publish TF frames dựa trên joint states.

```python
from launch import LaunchDescription
from launch_ros.actions import Node
from launch.substitutions import Command

def generate_launch_description():
    return LaunchDescription([
        Node(
            package='robot_state_publisher',
            executable='robot_state_publisher',
            name='robot_state_publisher',
            output='screen',
            parameters=[{
                'robot_description': open('robot.urdf').read()
            }]
        )
    ])
```

Hoặc dùng xacro (sẽ học ở bài sau):

```python
parameters=[{
    'robot_description': Command([
        'xacro ', 'robot.urdf.xacro'
    ])
}]
```

## 7. Kiểm tra URDF

```bash
# Cài đặt nếu chưa có
sudo apt install liburdfdom-tools

# Kiểm tra cú pháp URDF
check_urdf robot.urdf

# Xem cấu trúc tree
urdf_to_graphiz robot.urdf
# Tạo file robot.gv và robot.pdf
```

## 8. Ví dụ: Robot 2 bánh

```xml
<?xml version="1.0"?>
<robot name="diff_drive_robot">
    <!-- Materials -->
    <material name="blue">
        <color rgba="0 0 0.8 1"/>
    </material>
    <material name="black">
        <color rgba="0.1 0.1 0.1 1"/>
    </material>

    <!-- Base Link -->
    <link name="base_link">
        <visual>
            <geometry>
                <box size="0.5 0.3 0.1"/>
            </geometry>
            <material name="blue"/>
        </visual>
        <collision>
            <geometry>
                <box size="0.5 0.3 0.1"/>
            </geometry>
        </collision>
        <inertial>
            <mass value="2.0"/>
            <inertia ixx="0.01" ixy="0" ixz="0"
                     iyy="0.01" iyz="0"
                     izz="0.01"/>
        </inertial>
    </link>

    <!-- Left Wheel -->
    <link name="left_wheel">
        <visual>
            <geometry>
                <cylinder radius="0.05" length="0.04"/>
            </geometry>
            <material name="black"/>
        </visual>
        <collision>
            <geometry>
                <cylinder radius="0.05" length="0.04"/>
            </geometry>
        </collision>
        <inertial>
            <mass value="0.5"/>
            <inertia ixx="0.001" ixy="0" ixz="0"
                     iyy="0.001" iyz="0"
                     izz="0.001"/>
        </inertial>
    </link>

    <!-- Right Wheel -->
    <link name="right_wheel">
        <visual>
            <geometry>
                <cylinder radius="0.05" length="0.04"/>
            </geometry>
            <material name="black"/>
        </visual>
        <collision>
            <geometry>
                <cylinder radius="0.05" length="0.04"/>
            </geometry>
        </collision>
        <inertial>
            <mass value="0.5"/>
            <inertia ixx="0.001" ixy="0" ixz="0"
                     iyy="0.001" iyz="0"
                     izz="0.001"/>
        </inertial>
    </link>

    <!-- Caster Wheel -->
    <link name="caster_wheel">
        <visual>
            <geometry>
                <sphere radius="0.03"/>
            </geometry>
            <material name="black"/>
        </visual>
        <collision>
            <geometry>
                <sphere radius="0.03"/>
            </geometry>
        </collision>
    </link>

    <!-- Joints -->
    <joint name="left_wheel_joint" type="continuous">
        <parent link="base_link"/>
        <child link="left_wheel"/>
        <origin xyz="0 0.17 0" rpy="1.5708 0 0"/>
        <axis xyz="0 0 1"/>
    </joint>

    <joint name="right_wheel_joint" type="continuous">
        <parent link="base_link"/>
        <child link="right_wheel"/>
        <origin xyz="0 -0.17 0" rpy="1.5708 0 0"/>
        <axis xyz="0 0 1"/>
    </joint>

    <joint name="caster_joint" type="fixed">
        <parent link="base_link"/>
        <child link="caster_wheel"/>
        <origin xyz="-0.2 0 -0.05" rpy="0 0 0"/>
    </joint>
</robot>
```

Lưu file này thành `diff_drive_robot.urdf`, sau đó kiểm tra:

```bash
check_urdf diff_drive_robot.urdf
urdf_to_graphiz diff_drive_robot.urdf
```

## Tóm tắt

- URDF mô tả robot bằng XML với `<link>` và `<joint>`
- Mỗi link có `visual`, `collision`, và `inertial`
- Các loại joint: fixed, continuous, revolute, prismatic
- Geometry: box, cylinder, sphere, mesh
- `robot_state_publisher` publish TF từ URDF
- `check_urdf` và `urdf_to_graphiz` giúp kiểm tra

---

**Bài tiếp theo:** [Bài 22: Xacro — Macro cho URDF](../22.xacro/xacro.md)
