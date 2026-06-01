# Bài 22: Xacro — Macro cho URDF

URDF thuần túy nhanh chóng trở nên dài và lặp lại. Xacro (XML Macro) giúp bạn viết URDF ngắn gọn hơn bằng cách dùng biến, macro, và include.

## 1. Vấn đề với URDF thuần

Xem lại robot 2 bánh ở bài trước. Bánh trái và bánh phải gần như giống hệt nhau, nhưng bạn phải copy-paste toàn bộ `<link>` và `<joint>`. Nếu muốn đổi kích thước bánh, bạn phải sửa nhiều chỗ.

Xacro giải quyết điều này.

## 2. xacro:property

Định nghĩa hằng số để tái sử dụng.

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro" name="my_robot">

    <!-- Định nghĩa properties -->
    <xacro:property name="wheel_radius" value="0.05"/>
    <xacro:property name="wheel_width" value="0.04"/>
    <xacro:property name="base_length" value="0.5"/>
    <xacro:property name="base_width" value="0.3"/>
    <xacro:property name="base_height" value="0.1"/>
    <xacro:property name="wheel_offset_y" value="0.17"/>

    <!-- Dùng properties -->
    <link name="base_link">
        <visual>
            <geometry>
                <box size="${base_length} ${base_width} ${base_height}"/>
            </geometry>
        </visual>
    </link>

</robot>
```

## 3. xacro:macro

Định nghĩa template cho các phần lặp lại.

```xml
<!-- Macro định nghĩa một bánh xe -->
<xacro:macro name="wheel" params="prefix reflect">
    <link name="${prefix}_wheel">
        <visual>
            <geometry>
                <cylinder radius="${wheel_radius}" length="${wheel_width}"/>
            </geometry>
            <material name="black"/>
        </visual>
        <collision>
            <geometry>
                <cylinder radius="${wheel_radius}" length="${wheel_width}"/>
            </geometry>
        </collision>
        <inertial>
            <mass value="0.5"/>
            <inertia ixx="0.001" ixy="0" ixz="0"
                     iyy="0.001" iyz="0"
                     izz="0.001"/>
        </inertial>
    </link>

    <joint name="${prefix}_wheel_joint" type="continuous">
        <parent link="base_link"/>
        <child link="${prefix}_wheel"/>
        <origin xyz="0 ${reflect * wheel_offset_y} 0" rpy="${pi/2} 0 0"/>
        <axis xyz="0 0 1"/>
    </joint>
</xacro:macro>

<!-- Gọi macro 2 lần -->
<xacro:wheel prefix="left" reflect="1"/>
<xacro:wheel prefix="right" reflect="-1"/>
```

Giải thích:

- `params="prefix reflect"`: Khai báo tham số macro nhận vào
- `${prefix}`: Thay thế bằng giá trị truyền vào
- `${reflect}`: Dùng để đảo dấu vị trí (trái/phải)
- `${pi/2}`: Xacro hỗ trợ hằng số `pi`

## 4. Biểu thức toán học

Xacro hỗ trợ các phép toán trong `${}`:

```xml
<xacro:property name="wheel_radius" value="0.05"/>
<xacro:property name="wheel_diameter" value="${2 * wheel_radius}"/>
<xacro:property name="wheel_circumference" value="${2 * pi * wheel_radius}"/>

<!-- Phép toán phức tạp hơn -->
<origin xyz="${base_length/2} ${base_width/2 - wheel_width/2} 0" 
        rpy="${pi/2} 0 0"/>
```

Các hàm và hằng có sẵn:

- `pi`: Số pi
- `sin()`, `cos()`, `tan()`: Lượng giác
- `sqrt()`: Căn bậc hai
- `abs()`: Giá trị tuyệt đối

## 5. xacro:include

Chia URDF thành nhiều file để quản lý dễ hơn.

```xml
<!-- robot.xacro -->
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro" name="my_robot">

    <!-- Include file chứa properties -->
    <xacro:include filename="$(find my_pkg)/urdf/properties.xacro"/>

    <!-- Include file chứa macro bánh xe -->
    <xacro:include filename="$(find my_pkg)/urdf/wheel.xacro"/>

    <!-- Include file chứa macro cảm biến -->
    <xacro:include filename="$(find my_pkg)/urdf/sensors.xacro"/>

    <!-- Base link -->
    <link name="base_link">
        <visual>
            <geometry>
                <box size="${base_length} ${base_width} ${base_height}"/>
            </geometry>
        </visual>
    </link>

    <!-- Gọi macro từ file include -->
    <xacro:wheel prefix="left" reflect="1"/>
    <xacro:wheel prefix="right" reflect="-1"/>
    <xacro:lidar_mount prefix="front"/>

</robot>
```

## 6. Ví dụ: Robot 4 bánh dùng Xacro

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro" name="four_wheel_robot">

    <!-- ===== PROPERTIES ===== -->
    <xacro:property name="base_length" value="0.6"/>
    <xacro:property name="base_width" value="0.4"/>
    <xacro:property name="base_height" value="0.15"/>
    <xacro:property name="wheel_radius" value="0.08"/>
    <xacro:property name="wheel_width" value="0.06"/>
    <xacro:property name="wheel_mass" value="1.0"/>
    <xacro:property name="base_mass" value="10.0"/>

    <!-- ===== MATERIALS ===== -->
    <material name="blue">
        <color rgba="0 0 0.8 1"/>
    </material>
    <material name="black">
        <color rgba="0.1 0.1 0.1 1"/>
    </material>

    <!-- ===== MACROS ===== -->
    <xacro:macro name="box_inertia" params="m w h d">
        <inertial>
            <mass value="${m}"/>
            <inertia ixx="${m*(h*h+d*d)/12}" ixy="0" ixz="0"
                     iyy="${m*(w*w+d*d)/12}" iyz="0"
                     izz="${m*(w*w+h*h)/12}"/>
        </inertial>
    </xacro:macro>

    <xacro:macro name="cylinder_inertia" params="m r h">
        <inertial>
            <mass value="${m}"/>
            <inertia ixx="${m*(3*r*r+h*h)/12}" ixy="0" ixz="0"
                     iyy="${m*(3*r*r+h*h)/12}" iyz="0"
                     izz="${m*r*r/2}"/>
        </inertial>
    </xacro:macro>

    <xacro:macro name="wheel" params="prefix x y">
        <link name="${prefix}_wheel">
            <visual>
                <geometry>
                    <cylinder radius="${wheel_radius}" length="${wheel_width}"/>
                </geometry>
                <material name="black"/>
            </visual>
            <collision>
                <geometry>
                    <cylinder radius="${wheel_radius}" length="${wheel_width}"/>
                </geometry>
            </collision>
            <xacro:cylinder_inertia m="${wheel_mass}" r="${wheel_radius}" h="${wheel_width}"/>
        </link>

        <joint name="${prefix}_wheel_joint" type="continuous">
            <parent link="base_link"/>
            <child link="${prefix}_wheel"/>
            <origin xyz="${x} ${y} 0" rpy="${-pi/2} 0 0"/>
            <axis xyz="0 0 1"/>
        </joint>
    </xacro:macro>

    <!-- ===== ROBOT STRUCTURE ===== -->
    <link name="base_link">
        <visual>
            <geometry>
                <box size="${base_length} ${base_width} ${base_height}"/>
            </geometry>
            <material name="blue"/>
        </visual>
        <collision>
            <geometry>
                <box size="${base_length} ${base_width} ${base_height}"/>
            </geometry>
        </collision>
        <xacro:box_inertia m="${base_mass}" w="${base_width}" h="${base_height}" d="${base_length}"/>
    </link>

    <!-- 4 bánh xe -->
    <xacro:wheel prefix="front_left"  x="${base_length/3}"  y="${base_width/2 + wheel_width/2}"/>
    <xacro:wheel prefix="front_right" x="${base_length/3}"  y="${-(base_width/2 + wheel_width/2)}"/>
    <xacro:wheel prefix="rear_left"   x="${-base_length/3}" y="${base_width/2 + wheel_width/2}"/>
    <xacro:wheel prefix="rear_right"  x="${-base_length/3}" y="${-(base_width/2 + wheel_width/2)}"/>

</robot>
```

## 7. Biên dịch Xacro sang URDF

```bash
# Biên dịch trực tiếp
xacro robot.xacro > robot.urdf

# Hoặc dùng ros2 run
ros2 run xacro xacro robot.xacro > robot.urdf

# Kiểm tra output
check_urdf robot.urdf
```

## 8. Dùng Xacro trong Launch File

Thay vì biên dịch thủ công, bạn có thể dùng `Command` substitution trong launch file.

```python
from launch import LaunchDescription
from launch_ros.actions import Node
from launch.substitutions import Command
from launch_ros.parameter_descriptions import ParameterValue

def generate_launch_description():
    # Tự động biên dịch xacro khi launch
    robot_description = ParameterValue(
        Command([
            'xacro ',
            'robot.xacro'
        ]),
        value_type=str
    )

    return LaunchDescription([
        Node(
            package='robot_state_publisher',
            executable='robot_state_publisher',
            name='robot_state_publisher',
            output='screen',
            parameters=[{
                'robot_description': robot_description
            }]
        ),
        Node(
            package='joint_state_publisher',
            executable='joint_state_publisher',
            name='joint_state_publisher'
        ),
        Node(
            package='rviz2',
            executable='rviz2',
            name='rviz2'
        )
    ])
```

Lưu ý:

- `Command` chạy shell command và lấy stdout làm giá trị
- `value_type=str` báo cho ROS biết đây là string
- `joint_state_publisher` publish giá trị joint giả lập để bạn xem robot trong RViz2

## Tóm tắt

- Xacro giảm lặp lại trong URDF
- `xacro:property` định nghĩa hằng số
- `xacro:macro` tạo template có tham số
- `${}` cho phép biểu thức toán học
- `xacro:include` chia file để quản lý dễ hơn
- Biên dịch bằng `xacro file.xacro > file.urdf`
- Dùng `Command` substitution trong launch file để tự động biên dịch

---

**Bài tiếp theo:** [Bài 23: Gazebo Simulation](../23.gazebo-sim/gazebo-sim.md)
