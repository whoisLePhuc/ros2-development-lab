# Bài 18: Docker cho ROS2

Docker giúp đóng gói toàn bộ môi trường ROS2 vào một container. Bạn có thể chạy cùng một setup trên máy tính khác mà không cần cài đặt từng package thủ công.

## 1. Tại sao dùng Docker với ROS2?

- **Tái lập kết quả**: Cùng một image chạy giống hệt nhau ở mọi nơi
- **Cô lập**: Không xung đột với package hệ thống
- **Dễ chia sẻ**: Push image lên Docker Hub, người khác pull về chạy
- **Đa phiên bản**: Chạy ROS2 Humble và Foxy trên cùng một máy

## 2. Dockerfile cơ bản cho ROS2

```dockerfile
# Sử dụng image ROS2 Humble chính thức
FROM ros:humble-ros-base

# Cài thêm công cụ cần thiết
RUN apt-get update && apt-get install -y \
    ros-humble-rviz2 \
    ros-humble-turtlebot3-* \
    python3-pip \
    nano \
    && rm -rf /var/lib/apt/lists/*

# Tạo workspace
RUN mkdir -p /ros2_ws/src
WORKDIR /ros2_ws

# Copy code vào container
COPY ./src /ros2_ws/src

# Build workspace
RUN /bin/bash -c "source /opt/ros/humble/setup.bash && colcon build"

# Tự động source khi vào container
RUN echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
RUN echo "source /ros2_ws/install/setup.bash" >> ~/.bashrc

# Entrypoint mặc định
CMD ["/bin/bash"]
```

Build image:

```bash
docker build -t my_ros2_app .
```

## 3. Chạy container với GUI

ROS2 cần hiển thị GUI (RViz2, Gazebo). Bạn phải chia sẻ X11 socket.

### 3.1. Linux

```bash
# Cho phép Docker truy cập X11
xhost +local:docker

# Chạy container
docker run -it --rm \
    -e DISPLAY=$DISPLAY \
    -v /tmp/.X11-unix:/tmp/.X11-unix \
    --network host \
    my_ros2_app
```

### 3.2. Script chạy tiện lợi

Tạo file `run_docker.sh`:

```bash
#!/bin/bash

# Cho phép X11 access
xhost +local:docker 2>/dev/null || true

# Chạy container
docker run -it --rm \
    --name ros2_dev \
    -e DISPLAY=$DISPLAY \
    -e QT_X11_NO_MITSHM=1 \
    -v /tmp/.X11-unix:/tmp/.X11-unix \
    -v $(pwd)/ros2_ws:/ros2_ws \
    --network host \
    --privileged \
    my_ros2_app \
    bash
```

```bash
chmod +x run_docker.sh
./run_docker.sh
```

Lưu ý các flag:

- `-e DISPLAY=$DISPLAY`: Chia sẻ biến môi trường display
- `-v /tmp/.X11-unix`: Chia sẻ X11 socket
- `--network host`: Container dùng network của host (cần cho ROS2 DDS)
- `--privileged`: Quyền truy cập phần cứng (USB, serial)
- `-v $(pwd)/ros2_ws:/ros2_ws`: Mount workspace từ host vào container

## 4. Docker Compose cho multi-container

Khi hệ thống có nhiều thành phần, `docker-compose.yaml` giúp quản lý.

```yaml
version: '3.8'

services:
  ros2_base:
    build: .
    image: my_ros2_app
    container_name: ros2_base
    network_mode: host
    environment:
      - DISPLAY=${DISPLAY}
      - QT_X11_NO_MITSHM=1
    volumes:
      - /tmp/.X11-unix:/tmp/.X11-unix
      - ./ros2_ws:/ros2_ws
    privileged: true
    stdin_open: true
    tty: true
    command: bash

  # Service riêng cho simulation
  gazebo:
    image: my_ros2_app
    network_mode: host
    environment:
      - DISPLAY=${DISPLAY}
    volumes:
      - /tmp/.X11-unix:/tmp/.X11-unix
    privileged: true
    command: ros2 launch my_pkg simulation.launch.py

  # Service riêng cho navigation
  nav2:
    image: my_ros2_app
    network_mode: host
    command: ros2 launch my_pkg navigation.launch.py
```

Chạy:

```bash
# Khởi động tất cả service
docker-compose up

# Hoặc chỉ chạy một service
docker-compose up ros2_base

# Dừng tất cả
docker-compose down
```

## 5. Multi-stage build

Multi-stage giúp giảm kích thước image cuối cùng bằng cách loại bỏ build dependencies.

```dockerfile
# ===== Stage 1: Build =====
FROM ros:humble-ros-base AS builder

RUN apt-get update && apt-get install -y \
    build-essential \
    cmake \
    python3-colcon-common-extensions

WORKDIR /ros2_ws
COPY ./src ./src
RUN /bin/bash -c "source /opt/ros/humble/setup.bash && colcon build"

# ===== Stage 2: Runtime =====
FROM ros:humble-ros-base

RUN apt-get update && apt-get install -y \
    ros-humble-rviz2 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /ros2_ws
COPY --from=builder /ros2_ws/install ./install
RUN echo "source /ros2_ws/install/setup.bash" >> ~/.bashrc

CMD ["/bin/bash"]
```

Image cuối chỉ chứa runtime dependencies, không có compiler.

## 6. Chia sẻ image qua Docker Hub

```bash
# Đăng nhập
docker login

# Tag image
docker tag my_ros2_app username/my_ros2_app:humble

# Push
docker push username/my_ros2_app:humble

# Người khác pull về
docker pull username/my_ros2_app:humble
```

## 7. Mẹo thực tế

### Mount workspace thay vì copy

Dùng `-v` để mount workspace từ host. Thay đổi code trên host sẽ thấy ngay trong container.

```bash
docker run -it --rm \
    -v $(pwd)/ros2_ws:/ros2_ws \
    my_ros2_app \
    bash
```

### Giữ container chạy ngầm

```bash
# Chạy ngầm
docker run -d --name ros2_bg my_ros2_app tail -f /dev/null

# Vào container
docker exec -it ros2_bg bash

# Dừng
docker stop ros2_bg
```

### Dọn dẹp

```bash
# Xóa container đã dừng
docker container prune

# Xóa image không dùng
docker image prune -a
```

## Tóm tắt

- Dockerfile định nghĩa môi trường ROS2
- Chạy GUI cần mount X11 socket và biến `DISPLAY`
- `docker-compose` quản lý multi-container
- Multi-stage build giảm kích thước image
- Docker Hub giúp chia sẻ môi trường với team

---

**Bài tiếp theo:** [Bài 19: RViz2 — 3D Visualization](../19.rviz2/rviz2.md)
