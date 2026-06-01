# Bài 0.1: Linux cơ bản cho ROS2

Trước khi học ROS2, bạn cần làm quen với terminal Linux. ROS2 chạy chủ yếu trên Ubuntu, nên các lệnh dưới đây là công cụ hàng ngày của bạn.

---

## 1. Di chuyển trong hệ thống file

| Lệnh | Mô tả | Ví dụ |
|------|-------|-------|
| `pwd` | In đường dẫn thư mục hiện tại | `pwd` |
| `ls` | Liệt kê file và thư mục | `ls -la` |
| `cd` | Chuyển thư mục | `cd /home/username` |
| `mkdir` | Tạo thư mục mới | `mkdir my_folder` |
| `cp` | Sao chép file/thư mục | `cp file.txt backup.txt` |
| `mv` | Di chuyển hoặc đổi tên | `mv old.txt new.txt` |
| `rm` | Xóa file hoặc thư mục | `rm file.txt`, `rm -r folder/` |

### Ví dụ thực tế

```bash
# Xem mình đang ở đâu
pwd
# Output: /home/phuc

# Liệt kê file chi tiết
ls -la
# -l: dạng list, -a: hiện file ẩn

# Tạo thư mục workspace cho ROS2
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src

# Sao chép một gói ROS2
mkdir -p ~/backup
cp -r ~/ros2_ws/src/my_package ~/backup/
```

---

## 2. Biến môi trường: `source` và `export`

ROS2 dùng biến môi trường để tìm gói, thư viện, và công cụ.

### `export` - Đặt biến môi trường

```bash
# Đặt biến tạm thời trong terminal hiện tại
export MY_VAR="hello"
echo $MY_VAR   # In ra: hello
```

### `source` - Chạy script trong shell hiện tại

```bash
# Kích hoạt môi trường ROS2
source /opt/ros/humble/setup.bash

# Kích hoạt workspace của bạn
source ~/ros2_ws/install/setup.bash
```

> **Lưu ý quan trọng:** Dùng `source` thay vì chạy trực tiếp script. Nếu bạn chỉ gõ `setup.bash`, biến môi trường sẽ không áp dụng cho terminal hiện tại.

---

## 3. `sudo` và `apt`

Ubuntu dùng `apt` để quản lý phần mềm. Các lệnh cài đặt cần quyền root qua `sudo`.

```bash
# Cập nhật danh sách gói
sudo apt update

# Cài đặt một gói
sudo apt install python3-pip

# Cài đặt nhiều gói cùng lúc
sudo apt install git build-essential cmake

# Gỡ bỏ gói
sudo apt remove package_name
```

> **Mẹo:** Sau khi cài ROS2, bạn sẽ chạy `sudo apt install ros-humble-desktop` rất nhiều lần trong các bài sau.

---

## 4. Quyền file với `chmod`

Linux phân quyền rõ ràng: đọc (r), ghi (w), thực thi (x).

```bash
# Xem quyền file
ls -l script.py
# Output: -rw-r--r-- 1 user user 123 Jun  2 10:00 script.py

# Cấp quyền thực thi cho script
chmod +x script.py
# Giờ file có thể chạy: ./script.py

# Cấp quyền đầy đủ (thường dùng cho thư mục)
chmod 755 my_folder
# 7=rwx, 5=r-x, 5=r-x
```

| Số | Quyền |
|----|-------|
| 7 | rwx (đọc, ghi, thực thi) |
| 6 | rw- (đọc, ghi) |
| 5 | r-x (đọc, thực thi) |
| 4 | r-- (chỉ đọc) |

---

## 5. Shell scripting cơ bản

Script giúp tự động hóa lệnh lặp đi lặp lại.

### Script đơn giản

```bash
#!/bin/bash
# File: build_ros2.sh

echo "Bắt đầu build ROS2 workspace..."
cd ~/ros2_ws
colcon build --symlink-install

echo "Build xong! Kích hoạt workspace."
source install/setup.bash
```

Chạy script:
```bash
chmod +x build_ros2.sh
./build_ros2.sh
```

### Biến và vòng lặp

```bash
#!/bin/bash

# Biến
WORKSPACE="$HOME/ros2_ws"
echo "Workspace: $WORKSPACE"

# Vòng lặp for
for pkg in pkg1 pkg2 pkg3; do
    echo "Đang xử lý: $pkg"
done

# Kiểm tra điều kiện
if [ -d "$WORKSPACE" ]; then
    echo "Thư mục đã tồn tại"
else
    mkdir -p "$WORKSPACE"
    echo "Đã tạo thư mục"
fi
```

---

## 6. `~/.bashrc` - Cấu hình terminal

File `~/.bashrc` chạy mỗi khi bạn mở terminal mới. Đây là nơi bạn thêm các lệnh `source` và alias ROS2.

```bash
# Mở file để chỉnh sửa
nano ~/.bashrc
```

Thêm vào cuối file:

```bash
# Tự động source ROS2 Humble
source /opt/ros/humble/setup.bash

# Alias tiện lợi
alias rosbuild='cd ~/ros2_ws && colcon build --symlink-install'
alias rossource='source ~/ros2_ws/install/setup.bash'
alias rosdep='rosdep install --from-paths src --ignore-src -y'
```

Áp dụng thay đổi:
```bash
source ~/.bashrc
```

---

## 7. Lệnh hữu ích khác

```bash
# Tìm kiếm file
grep -r "class Node" ~/ros2_ws/src/

# Xem nội dung file
cat package.xml

# Xem phần đầu file
head -n 20 README.md

# Xem phần cuối file
tail -n 10 log.txt

# Giám sát tiến trình
top
htop

# Tìm tiến trình theo tên
ps aux | grep ros
```

---

## Bài tập

1. Tạo thư mục `~/ros2_ws/src` bằng lệnh `mkdir -p`.
2. Tạo file `hello_ros2.sh` với nội dung in ra "Hello ROS2!", cấp quyền thực thi, và chạy nó.
3. Thêm alias `ros2cd='cd ~/ros2_ws'` vào `~/.bashrc`, sau đó `source` lại và thử dùng.
4. Dùng `ls -la` để xem quyền của file `hello_ros2.sh` trước và sau khi `chmod +x`.
5. Viết script tự động tạo 3 thư mục: `~/ros2_ws/src`, `~/ros2_ws/build`, `~/ros2_ws/install`.

---

## Tóm tắt

- `cd/ls/pwd/mkdir/cp/mv/rm`: di chuyển và quản lý file.
- `source/export`: quản lý biến môi trường, cực kỳ quan trọng cho ROS2.
- `sudo/apt`: cài đặt phần mềm.
- `chmod`: cấp quyền thực thi cho script.
- `~/.bashrc`: tự động hóa môi trường ROS2 mỗi khi mở terminal.

---

**Bài tiếp theo:** [Bài 0.2: Python cơ bản cho ROS2](../01.python-basics/python-basics.md)
