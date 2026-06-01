# Bài 42: ROS2 + AI

## 1. OpenCV với ROS2

Robot cần "nhìn" để hiểu môi trường. OpenCV là thư viện xử lý ảnh phổ biến nhất, và ROS2 cung cấp `cv_bridge` để chuyển đổi giữa `sensor_msgs/Image` và `cv::Mat` (C++) / `numpy.ndarray` (Python).

### Cài đặt

```bash
sudo apt install ros-humble-cv-bridge
sudo apt install ros-humble-usb-cam
pip install opencv-python
```

### Python: Canny Edge Detection

```python
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import Image
from cv_bridge import CvBridge
import cv2

class CannyEdgeNode(Node):
    def __init__(self):
        super().__init__('canny_edge_node')
        self.bridge = CvBridge()
        self.sub = self.create_subscription(
            Image, '/camera/image_raw', self.image_cb, 10)
        self.pub = self.create_publisher(Image, '/camera/edges', 10)
        self.get_logger().info("Canny Edge Node started")

    def image_cb(self, msg):
        # Chuyển ROS Image → OpenCV Mat
        cv_image = self.bridge.imgmsg_to_cv2(msg, desired_encoding='bgr8')

        # Chuyển sang grayscale
        gray = cv2.cvtColor(cv_image, cv2.COLOR_BGR2GRAY)

        # Canny edge detection
        edges = cv2.Canny(gray, threshold1=100, threshold2=200)

        # Chuyển lại ROS Image
        edge_msg = self.bridge.cv2_to_imgmsg(edges, encoding='mono8')
        edge_msg.header = msg.header  # Giữ timestamp và frame_id

        self.pub.publish(edge_msg)
        self.get_logger().info("Published edge image", throttle_duration_sec=1.0)

def main():
    rclpy.init()
    node = CannyEdgeNode()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

### C++: Canny Edge Detection

```cpp
#include <rclcpp/rclcpp.hpp>
#include <sensor_msgs/msg/image.hpp>
#include <cv_bridge/cv_bridge.hpp>
#include <opencv2/opencv.hpp>

class CannyEdgeNode : public rclcpp::Node
{
public:
  CannyEdgeNode() : Node("canny_edge_node"), bridge_(std::make_shared<CvBridge>())
  {
    sub_ = this->create_subscription<sensor_msgs::msg::Image>(
      "/camera/image_raw", 10,
      std::bind(&CannyEdgeNode::imageCb, this, std::placeholders::_1));

    pub_ = this->create_publisher<sensor_msgs::msg::Image>("/camera/edges", 10);
    RCLCPP_INFO(this->get_logger(), "Canny Edge Node started");
  }

private:
  void imageCb(const sensor_msgs::msg::Image::SharedPtr msg)
  {
    try {
      // ROS Image → cv::Mat
      cv::Mat cv_image = cv_bridge::toCvCopy(msg, "bgr8")->image;

      // Grayscale + Canny
      cv::Mat gray, edges;
      cv::cvtColor(cv_image, gray, cv::COLOR_BGR2GRAY);
      cv::Canny(gray, edges, 100, 200);

      // cv::Mat → ROS Image
      auto edge_msg = cv_bridge::CvImage(msg->header, "mono8", edges).toImageMsg();
      pub_->publish(*edge_msg);

    } catch (const cv_bridge::Exception & e) {
      RCLCPP_ERROR(this->get_logger(), "cv_bridge exception: %s", e.what());
    }
  }

  std::shared_ptr<CvBridge> bridge_;
  rclcpp::Subscription<sensor_msgs::msg::Image>::SharedPtr sub_;
  rclcpp::Publisher<sensor_msgs::msg::Image>::SharedPtr pub_;
};

int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<CannyEdgeNode>());
  rclcpp::shutdown();
  return 0;
}
```

### Chạy

```bash
# Terminal 1: Camera
ros2 run usb_cam usb_cam_node_exe

# Terminal 2: Edge detection
ros2 run my_pkg canny_edge_node

# Terminal 3: Xem kết quả
ros2 run rqt_image_view rqt_image_view
```

## 2. YOLO Object Detection

YOLO (You Only Look Once) là mô hình object detection nhanh và chính xác. Bạn có thể tích hợp YOLOv8 (Ultralytics) với ROS2.

### Cài đặt

```bash
pip install ultralytics
```

### Python: YOLO + ROS2

```python
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import Image
from cv_bridge import CvBridge
from ultralytics import YOLO
import cv2

class YoloDetectorNode(Node):
    def __init__(self):
        super().__init__('yolo_detector')
        self.bridge = CvBridge()

        # Load YOLOv8n (nano, nhanh, nhẹ)
        self.get_logger().info("Loading YOLOv8n...")
        self.model = YOLO('yolov8n.pt')
        self.get_logger().info("YOLO loaded")

        self.sub = self.create_subscription(
            Image, '/camera/image_raw', self.image_cb, 10)
        self.pub = self.create_publisher(Image, '/camera/detections', 10)

    def image_cb(self, msg):
        # ROS → OpenCV
        cv_image = self.bridge.imgmsg_to_cv2(msg, desired_encoding='bgr8')

        # Inference
        results = self.model(cv_image, verbose=False)

        # Vẽ bounding boxes
        annotated = results[0].plot()

        # Đếm object
        num_objects = len(results[0].boxes)
        self.get_logger().info(f"Detected {num_objects} objects", throttle_duration_sec=1.0)

        # OpenCV → ROS
        out_msg = self.bridge.cv2_to_imgmsg(annotated, encoding='bgr8')
        out_msg.header = msg.header
        self.pub.publish(out_msg)

def main():
    rclpy.init()
    node = YoloDetectorNode()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

### Chạy với GPU (nếu có CUDA)

```python
# YOLO tự động dùng GPU nếu torch có CUDA
self.model = YOLO('yolov8n.pt')
# Kiểm tra: results[0].boxes.is_cuda
```

### Custom model (train riêng)

```python
# Train trên dataset riêng
from ultralytics import YOLO
model = YOLO('yolov8n.yaml')
model.train(data='custom_data.yaml', epochs=100, imgsz=640)

# Sau đó dùng model best.pt trong ROS2
self.model = YOLO('path/to/best.pt')
```

## 3. Reinforcement Learning với ROS2

Reinforcement Learning (RL) giúp robot học cách hành động thông qua thử và sai.

### Gym environment tích hợp ROS2

```python
import gymnasium as gym
from gymnasium import spaces
import numpy as np
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist
from sensor_msgs.msg import LaserScan

class ROS2RobotEnv(gym.Env):
    """Gym environment cho robot ROS2."""

    def __init__(self):
        super().__init__()

        # Khởi tạo ROS2 (nếu chưa init)
        if not rclpy.ok():
            rclpy.init()

        self.node = Node('rl_env_node')
        self.cmd_pub = self.node.create_publisher(Twist, 'cmd_vel', 10)
        self.scan_sub = self.node.create_subscription(
            LaserScan, 'scan', self.scan_cb, 10)

        # Không gian hành động: [linear_x, angular_z]
        self.action_space = spaces.Box(
            low=np.array([0.0, -1.0]),
            high=np.array([0.5, 1.0]),
            dtype=np.float32)

        # Không gian quan sát: 10 giá trị lidar
        self.observation_space = spaces.Box(
            low=0.0, high=10.0, shape=(10,), dtype=np.float32)

        self.current_scan = np.zeros(10)
        self.collision_threshold = 0.2

    def scan_cb(self, msg):
        # Lấy 10 giá trị đều từ 360 điểm lidar
        step = len(msg.ranges) // 10
        self.current_scan = np.array([
            min(msg.ranges[i * step], 10.0)  # Clamp max 10m
            for i in range(10)
        ])

    def reset(self, seed=None, options=None):
        super().reset(seed=seed)

        # Dừng robot
        self._send_cmd(0.0, 0.0)
        self.current_scan = np.zeros(10)

        return self.current_scan.astype(np.float32), {}

    def step(self, action):
        linear_x, angular_z = action
        self._send_cmd(linear_x, angular_z)

        # Đợi một chút để sensor cập nhật
        rclpy.spin_once(self.node, timeout_sec=0.1)

        # Tính reward
        min_distance = np.min(self.current_scan)
        reward = min_distance  # Càng xa vật cản càng tốt

        terminated = False
        truncated = False

        # Va chạm
        if min_distance < self.collision_threshold:
            reward = -10.0
            terminated = True

        # Reward tiến về phía trước
        reward += linear_x * 2.0

        return self.current_scan.astype(np.float32), reward, terminated, truncated, {}

    def _send_cmd(self, linear_x, angular_z):
        cmd = Twist()
        cmd.linear.x = float(linear_x)
        cmd.angular.z = float(angular_z)
        self.cmd_pub.publish(cmd)

    def close(self):
        self.node.destroy_node()
```

### Train với Stable Baselines3

```python
from stable_baselines3 import PPO
from ros2_robot_env import ROS2RobotEnv

env = ROS2RobotEnv()

# Tạo agent PPO
model = PPO("MlpPolicy", env, verbose=1)

# Train
model.learn(total_timesteps=100000)

# Lưu model
model.save("ros2_robot_ppo")

# Test
obs, _ = env.reset()
for _ in range(1000):
    action, _ = model.predict(obs)
    obs, reward, terminated, truncated, _ = env.step(action)
    if terminated or truncated:
        obs, _ = env.reset()

env.close()
```

## 4. TensorRT Optimization

TensorRT là SDK của NVIDIA để tối ưu inference trên GPU. Nó biên dịch model thành engine chuyên dụng, giảm latency 5-10x.

### Export YOLO sang TensorRT

```bash
# Ultralytics hỗ trợ export sang TensorRT tự động
yolo export model=yolov8n.pt format=engine device=0

# Kết quả: yolov8n.engine
```

### Python: TensorRT + ROS2

```python
from ultralytics import YOLO

class TensorRTYoloNode(Node):
    def __init__(self):
        super().__init__('tensorrt_yolo')
        self.bridge = CvBridge()

        # Load TensorRT engine
        self.get_logger().info("Loading TensorRT engine...")
        self.model = YOLO('yolov8n.engine')
        self.get_logger().info("TensorRT loaded")

        self.sub = self.create_subscription(
            Image, '/camera/image_raw', self.image_cb, 10)
        self.pub = self.create_publisher(Image, '/camera/detections', 10)

    def image_cb(self, msg):
        cv_image = self.bridge.imgmsg_to_cv2(msg, 'bgr8')
        results = self.model(cv_image, verbose=False)
        annotated = results[0].plot()
        out_msg = self.bridge.cv2_to_imgmsg(annotated, 'bgr8')
        out_msg.header = msg.header
        self.pub.publish(out_msg)
```

### So sánh performance

| Platform | Model | FPS (640x480) |
|----------|-------|---------------|
| CPU (i7) | YOLOv8n PyTorch | 5-10 |
| GPU (RTX 3060) | YOLOv8n PyTorch | 60-80 |
| GPU (RTX 3060) | YOLOv8n TensorRT | 200-300 |
| Jetson Nano | YOLOv8n TensorRT | 15-25 |
| Jetson Orin | YOLOv8n TensorRT | 100+ |

## 5. Tích hợp AI Pipeline trong ROS2

Một pipeline hoàn chỉnh cho robot tự hành:

```
Camera → YOLO (TensorRT) → Object positions
    ↓
Lidar → PointCloud → Obstacle map
    ↓
Fusion Node → Decision (Behavior Tree)
    ↓
Controller → cmd_vel → Motors
```

### Launch file cho AI pipeline

```python
from launch import LaunchDescription
from launch_ros.actions import Node

def generate_launch_description():
    return LaunchDescription([
        # Camera
        Node(package='usb_cam', executable='usb_cam_node_exe', name='camera'),

        # YOLO detection (TensorRT)
        Node(
            package='my_ai_pkg',
            executable='tensorrt_yolo',
            name='yolo_detector',
            parameters=[{'model_path': '/opt/models/yolov8n.engine'}]
        ),

        # Depth estimation (nếu có stereo camera)
        Node(
            package='my_ai_pkg',
            executable='depth_estimator',
            name='depth'
        ),

        # Fusion
        Node(
            package='my_ai_pkg',
            executable='sensor_fusion',
            name='fusion'
        ),

        # Navigation (Nav2)
        Node(
            package='nav2_bringup',
            executable='navigation_launch.py',
            name='nav2'
        ),
    ])
```

## 6. Lời chúc mừng và lời khuyên

Bạn đã hoàn thành khóa học **ROS2 Zero to Hire**! Từ bài 1 đến bài 42, bạn đã học:

- Cơ bản ROS2: node, topic, service, action, parameter.
- Công cụ: RViz, Gazebo, rqt, ros2bag.
- Lập trình: C++, Python, launch files, custom interfaces.
- Navigation: SLAM, Nav2, localization, path planning.
- Nâng cao: Lifecycle, Component, DDS tuning, Multi-robot.
- Robot arm: MoveIt2, kinematics, motion planning.
- AI: OpenCV, YOLO, Reinforcement Learning, TensorRT.

### Bước tiếp theo: Xây dựng 10 dự án

1. **Robot dò line**: Dùng camera + OpenCV để robot đi theo vạch.
2. **Robot tránh vật cản**: Lidar + simple algorithm (Bug0/Bug2).
3. **Robot follow person**: YOLO detect person → điều khiển theo.
4. **Robot pick-and-place**: Camera + MoveIt2 + gripper.
5. **Multi-robot warehouse**: 3 robot phân phối hàng trong kho.
6. **Robot đua tự hành**: RL train trên Gazebo, chạy trên real.
7. **Drone delivery**: PX4 + ROS2 + GPS waypoint.
8. **Robot inspection**: Đi tuần tra, chụp ảnh lỗi, report.
9. **Social robot**: Nhận diện khuôn mặt, giao tiếp giọng nói.
10. **Autonomous tractor**: GPS RTK + ROS2 + nông nghiệp chính xác.

### Tài nguyên tiếp tục học

- **ROS2 Documentation**: docs.ros.org/en/humble
- **Nav2 Tutorials**: navigation.ros.org
- **MoveIt2 Tutorials**: moveit.picknik.ai
- **OpenCV Docs**: docs.opencv.org
- **Ultralytics**: docs.ultralytics.com
- **Robotics Stack Exchange**: robotics.stackexchange.com

Chúc bạn thành công trên con đường Robotics Engineer!

---

**Kết thúc khóa học ROS2 Zero to Hire.**
