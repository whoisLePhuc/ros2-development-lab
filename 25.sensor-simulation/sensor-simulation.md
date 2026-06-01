# Bài 25: Sensor Simulation

Gazebo cho phép mô phỏng hầu hết các sensor robot dùng trong thực tế. Bài này đi sâu vào cấu hình LiDAR, camera, IMU, và GPS. Mỗi sensor đều có plugin riêng, topic output riêng, và tham số noise để mô phỏng điều kiện thực.

## 1. LiDAR (Ray Sensor)

LiDAR trong Gazebo dùng ray casting. Các tia laser được bắn ra, đo thời gian quay về để tính khoảng cách.

### Cấu hình đầy đủ

```xml
<link name="lidar_link">
  <collision>
    <geometry><cylinder radius="0.05" length="0.08"/></geometry>
  </collision>
  <visual>
    <geometry><cylinder radius="0.05" length="0.08"/></geometry>
  </visual>
  <inertial>
    <mass>0.1</mass>
    <inertia ixx="0.0001" iyy="0.0001" izz="0.0001"/>
  </inertial>

  <sensor name="lidar_sensor" type="ray">
    <always_on>true</always_on>
    <visualize>true</visualize>
    <update_rate>10</update_rate>
    <ray>
      <scan>
        <horizontal>
          <samples>360</samples>
          <resolution>1</resolution>
          <min_angle>-3.14159</min_angle>
          <max_angle>3.14159</max_angle>
        </horizontal>
      </scan>
      <range>
        <min>0.12</min>
        <max>10.0</max>
        <resolution>0.015</resolution>
      </range>
      <noise>
        <type>gaussian</type>
        <mean>0.0</mean>
        <stddev>0.01</stddev>
      </noise>
    </ray>
    <plugin name="lidar_plugin" filename="libgazebo_ros_ray_sensor.so">
      <ros>
        <remapping>~/out:=scan</remapping>
      </ros>
      <output_type>sensor_msgs/LaserScan</output_type>
      <frame_name>lidar_link</frame_name>
    </plugin>
  </sensor>
</link>
```

### Tham số quan trọng

| Tham số | Ý nghĩa | Gợi ý |
|---------|---------|-------|
| `<samples>` | Số tia laser | 360 cho 1 độ/mẫu, 720 cho 0.5 độ/mẫu |
| `<min_angle>` / `<max_angle>` | Góc quét (rad) | `-PI` đến `PI` cho 360 độ |
| `<min>` / `<max>` | Tầm đo (m) | 0.12 - 10m cho indoor, 0.2 - 30m cho outdoor |
| `<noise>` | Nhiễu Gaussian | `stddev=0.01` mô phỏng LiDAR rẻ, `0.001` cho LiDAR tốt |
| `<update_rate>` | Tần số cập nhật (Hz) | 10 Hz là chuẩn cho LiDAR 2D |

### Topic và message

- **Topic:** `/scan`
- **Message type:** `sensor_msgs/LaserScan`

```bash
ros2 topic echo /scan
# Hoặc visualize bằng RViz2, add LaserScan display
```

## 2. Camera

Camera trong Gazebo render hình ảnh từ scene 3D.

### Cấu hình đầy đủ

```xml
<link name="camera_link">
  <collision>
    <geometry><box size="0.05 0.05 0.05"/></geometry>
  </collision>
  <visual>
    <geometry><box size="0.05 0.05 0.05"/></geometry>
  </visual>
  <inertial>
    <mass>0.05</mass>
    <inertia ixx="0.00001" iyy="0.00001" izz="0.00001"/>
  </inertial>

  <sensor name="camera_sensor" type="camera">
    <camera>
      <horizontal_fov>1.047</horizontal_fov>
      <image>
        <width>640</width>
        <height>480</height>
        <format>R8G8B8</format>
      </image>
      <clip>
        <near>0.1</near>
        <far>100</far>
      </clip>
      <noise>
        <type>gaussian</type>
        <mean>0.0</mean>
        <stddev>0.007</stddev>
      </noise>
    </camera>
    <always_on>true</always_on>
    <update_rate>30</update_rate>
    <visualize>true</visualize>
    <plugin name="camera_plugin" filename="libgazebo_ros_camera.so">
      <ros>
        <remapping>~/image_raw:=camera/image_raw</remapping>
        <remapping>~/camera_info:=camera/camera_info</remapping>
      </ros>
      <frame_name>camera_link</frame_name>
      <hack_baseline>0.07</hack_baseline>
    </plugin>
  </sensor>
</link>
```

### Tham số quan trọng

| Tham số | Ý nghĩa | Gợi ý |
|---------|---------|-------|
| `<horizontal_fov>` | Góc nhìn ngang (rad) | 1.047 rad = 60 độ, 1.22 rad = 70 độ |
| `<width>` / `<height>` | Độ phân giải | 640x480 cho nhanh, 1920x1080 cho nét |
| `<format>` | Định dạng pixel | `R8G8B8` cho RGB, `B8G8R8` cho BGR |
| `<noise>` | Nhiễu hình ảnh | `stddev=0.007` mô phỏng ISO cao |
| `<update_rate>` | FPS | 30 Hz là chuẩn webcam |

### Topic và message

- **Topic hình ảnh:** `/camera/image_raw`
- **Topic camera info:** `/camera/camera_info`
- **Message types:** `sensor_msgs/Image`, `sensor_msgs/CameraInfo`

```bash
ros2 topic echo /camera/image_raw
ros2 run rqt_image_view rqt_image_view
```

## 3. IMU

IMU (Inertial Measurement Unit) đo gia tốc và vận tốc góc.

### Cấu hình đầy đủ

```xml
<link name="imu_link">
  <sensor name="imu_sensor" type="imu">
    <always_on>true</always_on>
    <update_rate>100</update_rate>
    <visualize>true</visualize>
    <imu>
      <angular_velocity>
        <x>
          <noise type="gaussian">
            <mean>0.0</mean>
            <stddev>0.02</stddev>
          </noise>
        </x>
        <y>
          <noise type="gaussian">
            <mean>0.0</mean>
            <stddev>0.02</stddev>
          </noise>
        </y>
        <z>
          <noise type="gaussian">
            <mean>0.0</mean>
            <stddev>0.02</stddev>
          </noise>
        </z>
      </angular_velocity>
      <linear_acceleration>
        <x>
          <noise type="gaussian">
            <mean>0.0</mean>
            <stddev>0.1</stddev>
          </noise>
        </x>
        <y>
          <noise type="gaussian">
            <mean>0.0</mean>
            <stddev>0.1</stddev>
          </noise>
        </y>
        <z>
          <noise type="gaussian">
            <mean>0.0</mean>
            <stddev>0.1</stddev>
          </noise>
        </z>
      </linear_acceleration>
    </imu>
    <plugin name="imu_plugin" filename="libgazebo_ros_imu_sensor.so">
      <ros>
        <remapping>~/out:=imu/data</remapping>
      </ros>
      <frame_name>imu_link</frame_name>
      <initial_orientation_as_reference>false</initial_orientation_as_reference>
    </plugin>
  </sensor>
</link>
```

### Topic và message

- **Topic:** `/imu/data`
- **Message type:** `sensor_msgs/Imu`

```bash
ros2 topic echo /imu/data
```

## 4. GPS

GPS trong Gazebo tính toán vị trí tuyệt đối dựa trên tọa độ world.

### Cấu hình đầy đủ

```xml
<link name="gps_link">
  <sensor name="gps_sensor" type="gps">
    <always_on>true</always_on>
    <update_rate>10</update_rate>
    <visualize>true</visualize>
    <plugin name="gps_plugin" filename="libgazebo_ros_gps_sensor.so">
      <ros>
        <remapping>~/out:=gps/fix</remapping>
      </ros>
      <frame_name>gps_link</frame_name>
      <reference_latitude>-30.060224594071936</reference_latitude>
      <reference_longitude>-51.173913575780311</reference_longitude>
      <reference_altitude>0.0</reference_altitude>
      <reference_heading>0.0</reference_heading>
      <velocity_noise>
        <type>gaussian</type>
        <mean>0.0</mean>
        <stddev>0.05</stddev>
      </velocity_noise>
      <position_noise>
        <type>gaussian</type>
        <mean>0.0</mean>
        <stddev>1.0</stddev>
      </position_noise>
    </plugin>
  </sensor>
</link>
```

### Tham số quan trọng

| Tham số | Ý nghĩa |
|---------|---------|
| `<reference_latitude>` | Vĩ độ gốc (độ) |
| `<reference_longitude>` | Kinh độ gốc (độ) |
| `<position_noise>` | Nhiễu vị trí (m) |
| `<velocity_noise>` | Nhiễu vận tốc (m/s) |

### Topic và message

- **Topic:** `/gps/fix`
- **Message type:** `sensor_msgs/NavSatFix`

```bash
ros2 topic echo /gps/fix
```

## 5. Ghi dữ liệu sensor

ROS2 bag là cách đơn giản nhất để ghi lại dữ liệu sensor.

```bash
# Ghi tất cả topic
ros2 bag record -a

# Ghi topic cụ thể
ros2 bag record /scan /camera/image_raw /imu/data /gps/fix

# Ghi với tên file tùy chỉnh
ros2 bag record -o sensor_data /scan /imu/data
```

Phát lại:

```bash
ros2 bag play sensor_data
```

Kiểm tra thông tin bag:

```bash
ros2 bag info sensor_data
```

## 6. Tóm tắt topic và message types

| Sensor | Topic mặc định | Message Type | Tần số gợi ý |
|--------|---------------|--------------|--------------|
| LiDAR | `/scan` | `sensor_msgs/LaserScan` | 10 Hz |
| Camera | `/camera/image_raw` | `sensor_msgs/Image` | 30 Hz |
| IMU | `/imu/data` | `sensor_msgs/Imu` | 100 Hz |
| GPS | `/gps/fix` | `sensor_msgs/NavSatFix` | 10 Hz |

## 7. Tối ưu hiệu năng

Sensor nặng nề nhất là camera (render 3D) và LiDAR (ray casting). Một số mẹo:

- Giảm độ phân giải camera nếu không cần nét.
- Giảm `<samples>` LiDAR nếu chỉ cần tránh vật cản thô.
- Tăng `<max_step_size>` trong physics để Gazebo chạy nhanh hơn.
- Dùng `<visualize>false>` cho sensor không cần hiển thị trong GUI.

---

**Bài tiếp theo:** [Bài 26: ros2_control](../26.ros2-control/ros2-control.md)
