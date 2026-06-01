# Bài 12: TF2 — Coordinate Transforms

Robots have many parts: a base, a LiDAR, a camera, an arm. Each sensor reports data in its own coordinate frame. **TF2** maintains a tree of coordinate frames and lets any node ask: "Where is the LiDAR relative to the base?" This lesson covers broadcasting, listening, and common tools.

---

## 1. What is TF2?

TF2 is a distributed transform library. It stores a **directed tree** of frames where each edge is a 3D transform (translation + rotation). Nodes publish transforms. Nodes query transforms. The library handles interpolation, buffering, and time travel.

Standard frame names you will see everywhere:

| Frame | Meaning |
|-------|---------|
| `map` | Global fixed frame, origin of the world |
| `odom` | Odometry origin, drifts over time |
| `base_link` | Center of the robot body |
| `base_footprint` | Projection of base_link onto the ground plane |
| `laser`, `camera_link`, `imu_link` | Sensor mounting points |

---

## 2. Broadcast a Transform

A node that knows where a sensor is mounted publishes the transform using `TransformBroadcaster`.

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from tf2_ros import TransformBroadcaster
from geometry_msgs.msg import TransformStamped
import tf_transformations

class FramePublisher(Node):
    def __init__(self):
        super().__init__('frame_publisher')
        self.br = TransformBroadcaster(self)
        self.timer = self.create_timer(0.1, self.broadcast_transform)

    def broadcast_transform(self):
        t = TransformStamped()
        t.header.stamp = self.get_clock().now().to_msg()
        t.header.frame_id = 'base_link'
        t.child_frame_id = 'laser'
        t.transform.translation.x = 0.2
        t.transform.translation.y = 0.0
        t.transform.translation.z = 0.1

        q = tf_transformations.quaternion_from_euler(0, 0, 0)
        t.transform.rotation.x = q[0]
        t.transform.rotation.y = q[1]
        t.transform.rotation.z = q[2]
        t.transform.rotation.w = q[3]

        self.br.sendTransform(t)

def main(args=None):
    rclpy.init(args=args)
    node = FramePublisher()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

`header.frame_id` is the parent frame. `child_frame_id` is the frame being defined.

---

## 3. Listen to a Transform

A node that needs to fuse sensor data looks up the transform between frames.

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from tf2_ros import Buffer, TransformListener
import tf2_ros

class TransformChecker(Node):
    def __init__(self):
        super().__init__('transform_checker')
        self.tf_buffer = Buffer()
        self.tf_listener = TransformListener(self.tf_buffer, self)
        self.timer = self.create_timer(1.0, self.check_transform)

    def check_transform(self):
        try:
            trans = self.tf_buffer.lookup_transform(
                'base_link',    # target frame
                'laser',        # source frame
                rclpy.time.Time()  # latest available
            )
            self.get_logger().info(
                f'Laser is at ({trans.transform.translation.x:.2f}, '
                f'{trans.transform.translation.y:.2f}, '
                f'{trans.transform.translation.z:.2f})'
            )
        except tf2_ros.LookupException:
            self.get_logger().warn('Transform not available yet')
        except tf2_ros.ConnectivityException:
            self.get_logger().warn('Transform tree is broken')
        except tf2_ros.ExtrapolationException:
            self.get_logger().warn('Extrapolation error')

def main(args=None):
    rclpy.init(args=args)
    node = TransformChecker()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

`Buffer` stores the transform history. `TransformListener` subscribes to `/tf` and `/tf_static` to fill the buffer.

---

## 4. Quaternion Basics

Rotations in ROS2 are represented as quaternions `(x, y, z, w)`. Most humans think in Euler angles (roll, pitch, yaw). Use `tf_transformations` to convert.

```python
import tf_transformations

# Euler (roll, pitch, yaw) to quaternion
q = tf_transformations.quaternion_from_euler(0.0, 0.0, 1.57)
# Returns [x, y, z, w]

# Quaternion to Euler
roll, pitch, yaw = tf_transformations.euler_from_quaternion([qx, qy, qz, qw])
```

Install the package if it is missing:

```bash
sudo apt install ros-humble-tf-transformations
```

---

## 5. Static Transforms

Some transforms never change: a camera bolted to a chassis. Publish these on `/tf_static` with `StaticTransformBroadcaster`.

```python
from tf2_ros import StaticTransformBroadcaster

class StaticFramePublisher(Node):
    def __init__(self):
        super().__init__('static_frame_publisher')
        self.br = StaticTransformBroadcaster(self)
        t = TransformStamped()
        t.header.stamp = self.get_clock().now().to_msg()
        t.header.frame_id = 'base_link'
        t.child_frame_id = 'camera_link'
        t.transform.translation.x = 0.15
        t.transform.translation.z = 0.05
        q = tf_transformations.quaternion_from_euler(0, -0.1, 0)
        t.transform.rotation.x = q[0]
        t.transform.rotation.y = q[1]
        t.transform.rotation.z = q[2]
        t.transform.rotation.w = q[3]
        self.br.sendTransform(t)
```

You can also use the command line tool:

```bash
ros2 run tf2_ros static_transform_publisher \
  0.2 0.0 0.1 0.0 0.0 0.0 base_link laser
```

Arguments are `x y z yaw pitch roll parent child`.

---

## 6. Robot with Multiple Sensors Example

A typical mobile robot setup:

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from tf2_ros import TransformBroadcaster, StaticTransformBroadcaster
from geometry_msgs.msg import TransformStamped
import tf_transformations

class RobotTFPublisher(Node):
    def __init__(self):
        super().__init__('robot_tf_publisher')
        self.static_br = StaticTransformBroadcaster(self)
        self.dynamic_br = TransformBroadcaster(self)

        # Static: sensors mounted on the robot
        self.publish_static('base_link', 'laser', 0.2, 0.0, 0.1, 0, 0, 0)
        self.publish_static('base_link', 'camera_link', 0.15, 0.0, 0.3, 0, -0.1, 0)
        self.publish_static('base_link', 'imu_link', 0.0, 0.0, 0.05, 0, 0, 0)

        # Dynamic: robot moving in odom frame
        self.timer = self.create_timer(0.05, self.publish_odom)
        self.x = 0.0

    def publish_static(self, parent, child, x, y, z, roll, pitch, yaw):
        t = TransformStamped()
        t.header.stamp = self.get_clock().now().to_msg()
        t.header.frame_id = parent
        t.child_frame_id = child
        t.transform.translation.x = x
        t.transform.translation.y = y
        t.transform.translation.z = z
        q = tf_transformations.quaternion_from_euler(roll, pitch, yaw)
        t.transform.rotation.x = q[0]
        t.transform.rotation.y = q[1]
        t.transform.rotation.z = q[2]
        t.transform.rotation.w = q[3]
        self.static_br.sendTransform(t)

    def publish_odom(self):
        self.x += 0.01
        t = TransformStamped()
        t.header.stamp = self.get_clock().now().to_msg()
        t.header.frame_id = 'odom'
        t.child_frame_id = 'base_link'
        t.transform.translation.x = self.x
        q = tf_transformations.quaternion_from_euler(0, 0, 0)
        t.transform.rotation.w = q[3]
        self.dynamic_br.sendTransform(t)

def main(args=None):
    rclpy.init(args=args)
    node = RobotTFPublisher()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

---

## 7. CLI Tools

```bash
# Print a transform in real time
ros2 run tf2_ros tf2_echo odom base_link

# Generate a PDF/PNG graph of the transform tree
ros2 run tf2_tools view_frames.py
# Produces frames.pdf and frames.gv in the current directory

# Check if a transform is available
ros2 topic echo /tf
ros2 topic echo /tf_static
```

---

## 8. Key Takeaways

- `TransformBroadcaster` for moving frames. `StaticTransformBroadcaster` for fixed mounts.
- Always use `Buffer` + `TransformListener` to query transforms.
- Catch `tf2_ros` exceptions; transforms may not be ready at startup.
- Stick to standard frame names: `map -> odom -> base_link -> sensors`.

---

## Bài tiếp theo

[Bài 13: Workspace & Colcon](../13.workspace-colcon/workspace-colcon.md)
