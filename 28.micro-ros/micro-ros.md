# Bài 28: Micro-ROS — ROS2 trên Vi điều khiển

Micro-ROS mang ROS2 xuống vi điều khiển (MCU). Thay vì chạy một full Linux board như Raspberry Pi, bạn có thể chạy node ROS2 trực tiếp trên STM32, ESP32, hoặc Arduino. Điều này giảm chi phí, giảm tiêu thụ điện, và giảm độ trễ cho các tác vụ real-time.

## 1. Micro-ROS là gì?

ROS2 thông thường chạy trên Linux với DDS middleware. Micro-ROS thay thế DDS bằng một agent nhẹ chạy trên Linux/PC. MCU chỉ chạy client, giao tiếp với agent qua serial, UDP, hoặc CAN.

```
+------------------+         Serial / UDP         +------------------+
|   ROS2 Network   |  <------------------------->  |   Micro-ROS      |
|   (Linux/PC)     |                             |   Client (MCU)   |
|                  |                             |                  |
|  +------------+  |                             |  +------------+  |
|  | Micro-ROS  |  |                             |  | rcl + rclc |  |
|  |   Agent    |  |                             |  | (C API)    |  |
|  +------------+  |                             |  +------------+  |
|        |         |                             |        |         |
|  +------------+  |                             |  +------------+  |
|  |   DDS/RTPS   |  |                             |  | Middleware |  |
|  +------------+  |                             |  | (XRCE-DDS) |  |
+------------------+                             +------------------+
```

## 2. Kiến trúc

- **Client**: Chạy trên MCU. Dùng thư viện `rclc` (C API nhẹ). Hỗ trợ publisher, subscriber, service, client, action.
- **Agent**: Chạy trên Linux. Nhận packet từ MCU, chuyển đổi sang DDS/RTPS, và tham gia mạng ROS2.
- **Transport**: Serial (UART/USB), UDP, TCP, hoặc CAN FD.

## 3. ESP32 với PlatformIO

ESP32 có WiFi và nhiều GPIO, phù hợp cho robot nhỏ.

### Cài đặt PlatformIO

```bash
pip install platformio
```

### platformio.ini

```ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino

lib_deps =
    https://github.com/micro-ROS/micro_ros_platformio

build_flags =
    -L ./.pio/libdeps/esp32dev/micro_ros_platformio/libmicroros
    -I ./.pio/libdeps/esp32dev/micro_ros_platformio/libmicroros/include
```

### Code mẫu: Publisher + Subscriber

```cpp
// src/main.cpp
#include <micro_ros_platformio.h>
#include <rcl/rcl.h>
#include <rclc/rclc.h>
#include <rclc/executor.h>
#include <std_msgs/msg/float32.h>
#include <std_msgs/msg/bool.h>

// Node và executor
rcl_node_t node;
rclc_executor_t executor;
rclc_support_t support;
rcl_allocator_t allocator;

// Publisher: nhiệt độ
rcl_publisher_t temp_publisher;
std_msgs__msg__Float32 temp_msg;

// Subscriber: điều khiển LED
rcl_subscription_t led_subscriber;
std_msgs__msg__Bool led_msg;

// Timer
rcl_timer_t timer;

// GPIO
const int LED_PIN = 2;
const int TEMP_SENSOR_PIN = 34; // ADC pin

// Callback timer: publish nhiệt độ
void timer_callback(rcl_timer_t * timer, int64_t last_call_time)
{
  (void) last_call_time;
  if (timer != NULL) {
    // Đọc nhiệt độ từ sensor (giả lập)
    int raw = analogRead(TEMP_SENSOR_PIN);
    float voltage = raw * 3.3 / 4095.0;
    float temp = (voltage - 0.5) * 100.0; // LM35

    temp_msg.data = temp;
    rcl_publish(&temp_publisher, &temp_msg, NULL);
  }
}

// Callback subscription: nhận lệnh LED
void led_subscription_callback(const void * msgin)
{
  const std_msgs__msg__Bool * msg = (const std_msgs__msg__Bool *)msgin;
  digitalWrite(LED_PIN, msg->data ? HIGH : LOW);
}

void setup()
{
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);
  pinMode(TEMP_SENSOR_PIN, INPUT);

  // Khởi tạo micro-ROS transport (serial)
  set_microros_serial_transports(Serial);

  delay(2000);

  allocator = rcl_get_default_allocator();

  // Khởi tạo support
  rclc_support_init(&support, 0, NULL, &allocator);

  // Tạo node
  rclc_node_init_default(&node, "esp32_node", "", &support);

  // Tạo publisher
  rclc_publisher_init_default(
    &temp_publisher,
    &node,
    ROSIDL_GET_MSG_TYPE_SUPPORT(std_msgs, msg, Float32),
    "temperature");

  // Tạo subscriber
  rclc_subscription_init_default(
    &led_subscriber,
    &node,
    ROSIDL_GET_MSG_TYPE_SUPPORT(std_msgs, msg, Bool),
    "cmd_led");

  // Tạo timer (1 Hz)
  rclc_timer_init_default(
    &timer,
    &support,
    RCL_MS_TO_NS(1000),
    timer_callback);

  // Tạo executor
  rclc_executor_init(&executor, &support.context, 2, &allocator);
  rclc_executor_add_timer(&executor, &timer);
  rclc_executor_add_subscription(&executor, &led_subscriber, &led_msg, &led_subscription_callback, ON_NEW_DATA);
}

void loop()
{
  rclc_executor_spin_some(&executor, RCL_MS_TO_NS(100));
  delay(10);
}
```

Build và upload:

```bash
pio run --target upload
```

## 4. STM32 với UART

STM32 thường không có WiFi, nên dùng UART để nối với Raspberry Pi hoặc PC.

### Kết nối phần cứng

```
STM32F4xx          Raspberry Pi 4
PA9 (USART1_TX) -> GPIO15 (UART_RX)
PA10 (USART1_RX) -> GPIO14 (UART_TX)
GND              -> GND
```

### Code STM32 (tóm tắt)

```c
// main.c với STM32CubeIDE + micro-ROS
#include <rcl/rcl.h>
#include <rclc/rclc.h>
#include <rclc/executor.h>
#include <std_msgs/msg/int32.h>

// UART transport
#include <rmw_microxrcedds_c/config.h>
#include <rmw_microros/rmw_microros.h>

bool cubemx_transport_open(struct uxrCustomTransport * transport);
bool cubemx_transport_close(struct uxrCustomTransport * transport);
size_t cubemx_transport_write(struct uxrCustomTransport * transport, const uint8_t * buf, size_t len, uint8_t * err);
size_t cubemx_transport_read(struct uxrCustomTransport * transport, uint8_t * buf, size_t len, int timeout, uint8_t * err);

rcl_publisher_t publisher;
std_msgs__msg__Int32 msg;

int main(void)
{
  HAL_Init();
  SystemClock_Config();
  MX_USART1_UART_Init();

  // Thiết lập transport
  rmw_uros_set_custom_transport(
    true,
    (void *) &huart1,
    cubemx_transport_open,
    cubemx_transport_close,
    cubemx_transport_write,
    cubemx_transport_read
  );

  rcl_allocator_t allocator = rcl_get_default_allocator();
  rclc_support_t support;
  rclc_support_init(&support, 0, NULL, &allocator);

  rcl_node_t node;
  rclc_node_init_default(&node, "stm32_node", "", &support);

  rclc_publisher_init_default(
    &publisher,
    &node,
    ROSIDL_GET_MSG_TYPE_SUPPORT(std_msgs, msg, Int32),
    "stm32_counter");

  rcl_timer_t timer;
  rclc_timer_init_default(&timer, &support, RCL_MS_TO_NS(1000), timer_callback);

  rclc_executor_t executor;
  rclc_executor_init(&executor, &support.context, 1, &allocator);
  rclc_executor_add_timer(&executor, &timer);

  msg.data = 0;

  while (1) {
    rclc_executor_spin_some(&executor, RCL_MS_TO_NS(100));
    HAL_Delay(10);
  }
}

void timer_callback(rcl_timer_t * timer, int64_t last_call_time)
{
  (void) last_call_time;
  if (timer != NULL) {
    msg.data++;
    rcl_publish(&publisher, &msg, NULL);
  }
}
```

## 5. Chạy Micro-ROS Agent

Agent là cầu nối giữa MCU và mạng ROS2.

### Cài đặt Agent

```bash
# Docker (nhanh nhất)
docker run -it --rm -v /dev:/dev --privileged microros/micro-ros-agent:humble serial --dev /dev/ttyUSB0 -b 115200

# Hoặc build từ source
git clone -b humble https://github.com/micro-ROS/micro-ROS-Agent.git
cd micro-ROS-Agent
colcon build
source install/local_setup.bash
```

### Chạy Agent với Serial

```bash
# Nếu build từ source
ros2 run micro_ros_agent micro_ros_agent serial --dev /dev/ttyUSB0 -b 115200

# Hoặc với Docker
docker run -it --rm -v /dev:/dev --privileged microros/micro-ros-agent:humble serial --dev /dev/ttyUSB0 -b 115200
```

### Chạy Agent với UDP (ESP32 WiFi)

```bash
ros2 run micro_ros_agent micro_ros_agent udp4 --port 8888
```

ESP32 sẽ kết nối tới IP của PC, port 8888.

## 6. Kiểm tra bằng CLI

Sau khi Agent chạy và MCU đã kết nối:

```bash
# Liệt kê node
ros2 node list
# /esp32_node
# /stm32_node

# Liệt kê topic
ros2 topic list
# /temperature
# /cmd_led
# /stm32_counter

# Đọc nhiệt độ
ros2 topic echo /temperature

# Gửi lệnh LED
ros2 topic pub /cmd_led std_msgs/msg/Bool "data: true"

# Kiểm tra graph
ros2 run rqt_graph rqt_graph
```

## 7. Hạn chế của Micro-ROS

Micro-ROS rất mạnh nhưng có một số giới hạn cần biết:

| Hạn chế | Mô tả | Giải pháp |
|---------|-------|-----------|
| Không có `ros2 launch` | MCU không chạy Python | Dùng launch file trên PC cho Agent |
| Không có `ros2 bag` | MCU không lưu file | Ghi bag trên PC |
| Bộ nhớ hạn chế | ESP32 ~300KB RAM, STM32F4 ~128KB | Giảm số node, topic, message size |
| Không hỗ trợ đầy đủ QoS | Một số QoS policy không khả dụng | Dùng `RMW_QOS_POLICY_RELIABILITY_BEST_EFFORT` |
| Không có `ros2 param` | Tham số không linh hoạt như Linux | Hardcode hoặc dùng custom service |
| Debug khó khăn | Không có `ros2 topic hz` trên MCU | Dùng Serial print hoặc logic analyzer |
| Chỉ hỗ trợ C | Không có C++ client API trên MCU | Dùng `rclc` (C API) |

## 8. So sánh Micro-ROS và Hardware Interface truyền thống

| Tiêu chí | Micro-ROS | Serial + Hardware Interface |
|----------|-----------|----------------------------|
| MCU chạy | Full ROS2 node | Custom firmware đơn giản |
| PC chạy | Chỉ Agent | Controller Manager + Hardware Interface |
| Độ phức tạp MCU | Cao (cần micro-ROS lib) | Thấp (chỉ parse serial) |
| Độ phức tạp PC | Thấp | Cao (viết Hardware Interface C++) |
| Linh hoạt | Cao, dùng được nhiều node | Thấp, chỉ điều khiển + sensor |
| Bộ nhớ MCU | Yêu cầu lớn | Nhỏ |
| Phù hợp | Robot phức tạp, nhiều sensor | Robot đơn giản, MCU yếu |

Nếu MCU của bạn đủ mạnh (ESP32, STM32F4/F7/H7), Micro-ROS là lựa chọn tốt. Nếu dùng Arduino Uno hoặc STM32F1, nên dùng Serial + Hardware Interface truyền thống.

## 9. Tóm tắt

- Micro-ROS mang ROS2 xuống MCU qua client-agent architecture.
- Client chạy trên MCU (ESP32, STM32), agent chạy trên Linux.
- ESP32 dùng PlatformIO, STM32 dùng STM32CubeIDE + UART transport.
- Agent có thể chạy bằng Docker hoặc build từ source.
- Kiểm tra hoạt động bằng `ros2 node list` và `ros2 topic echo`.
- Micro-ROS có hạn chế về bộ nhớ, QoS, và debug. Đánh đổi với sự linh hoạt.

---

**Bài tiếp theo:** [Bài 29: SLAM](../29.slam/slam.md)
