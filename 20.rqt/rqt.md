# Bài 20: RQT — ROS GUI Tools

RQT là framework GUI cho ROS2. Nó cung cấp nhiều plugin để introspect và tương tác với hệ thống ROS2 theo cách trực quan. Khác với RViz2 (chuyên về 3D), RQT tập trung vào dữ liệu và điều khiển.

## 1. Khởi động RQT

```bash
# Mở RQT
rqt

# Mở plugin cụ thể trực tiếp
rqt --standalone rqt_graph
rqt --standalone rqt_topic
```

## 2. Các plugin chính

### 2.1. Node Graph

**Mở**: Plugins > Introspection > Node Graph

Hiển thị toàn bộ node và topic dưới dạng sơ đồ có hướng.

- **Dùng khi nào**: Bạn muốn hiểu kiến trúc hệ thống, xem node nào publish/subscribe topic nào
- **Tính năng**: Có thể lọc theo node hoặc topic, zoom, lưu ảnh
- **Mẹo**: Nếu graph quá rối, bỏ tick "Hide > Dead sinks" và "Hide > Leaf topics"

### 2.2. Topic Monitor

**Mở**: Plugins > Topics > Topic Monitor

Xem real-time message từ mọi topic đang chạy.

- **Dùng khi nào**: Kiểm tra nhanh dữ liệu đang chảy trên topic
- **Tính năng**: Chọn topic muốn xem, hiển thị dạng tree hoặc raw
- **Mẹo**: Double-click vào topic để mở rộng và xem cấu trúc message

### 2.3. Plot

**Mở**: Plugins > Visualization > Plot

Vẽ đồ thị các trường số từ topic.

- **Dùng khi nào**: Theo dõi vận tốc, vị trí, nhiệt độ, hoặc bất kỳ giá trị số nào theo thời gian
- **Cách dùng**: Kéo thả field từ Topic Monitor vào Plot, hoặc nhập trực tiếp `/topic/field`
- **Ví dụ**: `/odom/pose/pose/position/x` để vẽ tọa độ x theo thời gian

### 2.4. Console

**Mở**: Plugins > Logging > Console

Xem log từ `/rosout` trong giao diện GUI.

- **Dùng khi nào**: Theo dõi log từ nhiều node mà không cần mở nhiều terminal
- **Tính năng**: Lọc theo mức log (DEBUG, INFO, WARN, ERROR), lọc theo node, highlight màu
- **Mẹo**: Bật "Autoscroll" để luôn thấy log mới nhất

### 2.5. Service Caller

**Mở**: Plugins > Services > Service Caller

Gọi service trực tiếp từ GUI.

- **Dùng khi nào**: Test service mà không cần viết code
- **Cách dùng**: Chọn service, điền giá trị argument, bấm Call
- **Ví dụ**: Gọi `/spawn` trong turtlesim để tạo rùa mới

## 3. Khi nào dùng plugin nào?

| Bạn muốn... | Dùng plugin |
|-------------|-------------|
| Hiểu kiến trúc hệ thống | Node Graph |
| Xem dữ liệu đang chảy | Topic Monitor |
| Vẽ đồ thị giá trị số | Plot |
| Theo dõi log nhiều node | Console |
| Test service nhanh | Service Caller |
| Xem ảnh từ camera | Image View |
| Gửi message lên topic | Topic Publisher |

## 4. Mẹo sử dụng

### Lưu bố cục

RQT cho phép lưu bố cục các plugin (Perspective).

- **Save**: Perspectives > Save Perspective As...
- **Load**: Perspectives > Load Perspective...
- **Mặc định**: File `~/.config/ros.org/rqt_gui.ini`

### Chạy nhiều plugin cùng lúc

Bạn có thể chia màn hình thành nhiềng tab hoặc dock các plugin cạnh nhau.

```bash
# Ví dụ: mở với 2 plugin
rqt --standalone rqt_graph --baggins "rqt_plot;rqt_console"
```

### Phím tắt

- `Ctrl+T`: New tab
- `Ctrl+W`: Close tab
- `Ctrl+Shift+F`: Find plugin

## Tóm tắt

- RQT là bộ công cụ GUI đa năng cho ROS2
- Node Graph giúp hiểu kiến trúc
- Topic Monitor xem dữ liệu real-time
- Plot vẽ đồ thị các giá trị số
- Console tập trung log từ `/rosout`
- Service Caller test service không cần code

---

**Bài tiếp theo:** [Bài 21: URDF — Robot Description](../21.urdf/urdf.md)
