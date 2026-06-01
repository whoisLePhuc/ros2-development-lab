# Bài 2: Cài đặt ROS2

Bài này hướng dẫn cài đặt ROS2 Humble Hawksbill trên Ubuntu 22.04 (Jammy Jellyfish). Humble là bản LTS (Long Term Support) được hỗ trợ đến năm 2027.

---

## 1. Chuẩn bị hệ thống

### Kiểm tra phiên bản Ubuntu

```bash
lsb_release -a
```

Bạn cần thấy `Ubuntu 22.04`. Nếu đang dùng phiên bản khác, hãy cài đặt đúng bản ROS2 tương thích.

### Cài đặt locale

ROS2 yêu cầu locale hỗ trợ UTF-8.

```bash
locale  # Kiểm tra locale hiện tại

# Nếu chưa có UTF-8, chạy:
sudo apt update && sudo apt install locales
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8

# Kiểm tra lại
locale
```

---

## 2. Thêm kho phần mềm ROS2

### Cài đặt các công cụ cần thiết

```bash
sudo apt install software-properties-common
sudo add-apt-repository universe
```

### Thêm GPG key

```bash
sudo apt update && sudo apt install curl -y
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
```

### Thêm repository

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
```

### Cập nhật danh sách gói

```bash
sudo apt update
sudo apt upgrade
```

---

## 3. Cài đặt ROS2 Humble

### Cài đặt bản Desktop (khuyên dùng)

Bản Desktop bao gồm ROS2 core, công cụ build, thư viện robot, và các demo.

```bash
sudo apt install ros-humble-desktop
```

### Cài đặt bản Base (tối thiểu)

Nếu bạn chỉ cần core và không cần GUI:

```bash
sudo apt install ros-humble-ros-base
```

Quá trình cài đặt có thể mất 10-30 phút tùy theo tốc độ mạng.

---

## 4. Cài đặt công cụ phát triển

### `colcon` - Công cụ build workspace

```bash
sudo apt install python3-colcon-common-extensions
```

### `rosdep` - Quản lý phụ thuộc

```bash
sudo apt install python3-rosdep
sudo rosdep init
rosdep update
```

> **Lưu ý:** Nếu `rosdep init` báo lỗi "already initialized", bạn có thể bỏ qua.

---

## 5. Thiết lập môi trường

### Source ROS2 trong terminal hiện tại

```bash
source /opt/ros/humble/setup.bash
```

### Tự động source mỗi khi mở terminal

Thêm vào `~/.bashrc`:

```bash
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

### Kiểm tra môi trường

```bash
printenv | grep ROS
```

Bạn nên thấy các biến như:
```
ROS_VERSION=2
ROS_DISTRO=humble
```

---

## 6. Kiểm tra cài đặt với demo_nodes

ROS2 cung cấp các node demo để kiểm tra.

### Terminal 1: Chạy talker

```bash
ros2 run demo_nodes_cpp talker
```

Bạn sẽ thấy output:
```
[INFO] [1234567890.123456789] [talker]: Publishing: 'Hello World: 1'
[INFO] [1234567890.623456789] [talker]: Publishing: 'Hello World: 2'
```

### Terminal 2: Chạy listener

Mở terminal mới và chạy:

```bash
ros2 run demo_nodes_py listener
```

Bạn sẽ thấy:
```
[INFO] [1234567890.234567890] [listener]: I heard: [Hello World: 1]
[INFO] [1234567890.734567890] [listener]: I heard: [Hello World: 2]
```

Nếu talker và listener giao tiếp được, ROS2 đã cài đặt thành công.

---

## 7. Lỗi thường gặp và cách khắc phục

### Lỗi: `ros2: command not found`

**Nguyên nhân:** Chưa source môi trường ROS2.

**Khắc phục:**
```bash
source /opt/ros/humble/setup.bash
```

Hoặc thêm vào `~/.bashrc` như hướng dẫn ở mục 5.

### Lỗi: `Package 'demo_nodes_cpp' not found`

**Nguyên nhân:** Cài bản `ros-base` thay vì `desktop`, hoặc cài đặt bị lỗi.

**Khắc phục:**
```bash
sudo apt install ros-humble-demo-nodes-cpp ros-humble-demo-nodes-py
```

### Lỗi: `Unable to contact my own daemon`

**Nguyên nhân:** ROS2 daemon gặp vấn đề.

**Khắc phục:**
```bash
ros2 daemon stop
ros2 daemon start
```

Hoặc:
```bash
ros2 daemon status
# Nếu không chạy:
ros2 daemon start
```

### Lỗi: `rosdep update` bị lỗi SSL

**Khắc phục:**
```bash
rosdep update --include-eol-distros
```

Hoặc thử lại sau vài phút nếu là lỗi mạng tạm thời.

### Lỗi: Không tìm thấy gói sau khi `apt install`

**Nguyên nhân:** Chưa `sudo apt update` sau khi thêm repository.

**Khắc phục:**
```bash
sudo apt update
sudo apt install ros-humble-desktop
```

---

## Tóm tắt

1. Kiểm tra Ubuntu 22.04 và cài locale UTF-8.
2. Thêm GPG key và repository ROS2.
3. Cài `ros-humble-desktop` qua `apt`.
4. Cài `colcon` và `rosdep`.
5. Source `/opt/ros/humble/setup.bash` (thêm vào `~/.bashrc` để tự động).
6. Kiểm tra bằng `demo_nodes_cpp talker` và `demo_nodes_py listener`.

---

**Bài tiếp theo:** [Bài 3: Làm quen ROS2 với Turtlesim](../04.turtlesim/turtlesim.md)
