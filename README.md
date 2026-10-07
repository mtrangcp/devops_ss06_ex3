## 1. Mục tiêu

Cấu hình máy chủ chỉ cho phép kết nối đi vào cổng SSH `22/tcp` và cổng ứng dụng Web `8080/tcp`; các kết nối đi vào khác bị chặn. Kết nối đi ra được cho phép.

## 2. Cấu hình UFW

Thực hiện trên Ubuntu/Debian VPS qua SSH. Mở cổng SSH **trước khi bật UFW** để tránh mất kết nối quản trị:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 8080/tcp
sudo ufw enable
```

Nhấn `y` khi UFW yêu cầu xác nhận. Có thể dùng script đi kèm để thực hiện toàn bộ cấu hình:

```bash
chmod +x configure-ufw.sh
sudo ./configure-ufw.sh
```

Script có thể chạy lại mà không làm mất các rule đã thiết lập.

> Nếu VPS dùng cổng SSH khác `22`, phải mở cổng SSH thực tế trước khi bật UFW. Ngoài UFW, cần kiểm tra Security Group/Cloud Firewall của nhà cung cấp và cho phép `22/tcp`, `8080/tcp` nếu dịch vụ này đang được bật.

## 3. Kiểm tra trạng thái tường lửa

```bash
sudo ufw status verbose
```

Kết quả mong đợi (địa chỉ IPv6 chỉ xuất hiện khi IPv6 được bật):

```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
8080/tcp                   ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
8080/tcp (v6)              ALLOW IN    Anywhere (v6)
```

### Kết quả ghi nhận

Đầu ra minh họa theo cấu hình của bài thực hành:

```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
8080/tcp                   ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
8080/tcp (v6)              ALLOW IN    Anywhere (v6)
```

> Đây là kết quả minh họa vì báo cáo được chuẩn bị trên Windows. Khi thực hành trên VPS Linux, chạy `sudo ufw status verbose` và thay khối trên bằng đầu ra thực tế của máy chủ.

## 4. Kiểm tra cổng đang lắng nghe

```bash
sudo ss -tlnp
```

Có thể lọc riêng hai cổng cần kiểm tra:

```bash
sudo ss -tlnp | grep -E ':(22|8080)\b'
```

Kết quả phải có cổng `22` do dịch vụ SSH lắng nghe. Cổng `8080` chỉ xuất hiện khi ứng dụng Web đã khởi động và thực sự lắng nghe trên cổng đó. Rule UFW `ALLOW` không tự tạo tiến trình lắng nghe.

Ví dụ dòng mong đợi khi ứng dụng nhận kết nối từ mọi giao diện mạng:

```text
LISTEN 0 128 0.0.0.0:22   0.0.0.0:* users:(("sshd",pid=...,fd=...))
LISTEN 0 128 0.0.0.0:8080 0.0.0.0:* users:(("java",pid=...,fd=...))
```

Nếu ứng dụng chỉ hiện `127.0.0.1:8080`, máy bên ngoài sẽ không truy cập trực tiếp được dù UFW đã mở cổng; cần cấu hình ứng dụng lắng nghe trên `0.0.0.0:8080` hoặc địa chỉ mạng phù hợp.

## 5. Kiểm tra kết nối Web

Kiểm tra ngay trên máy chủ:

```bash
curl -v --connect-timeout 5 http://127.0.0.1:8080/
```

Kiểm tra từ một máy khác, thay `<PUBLIC_IP>` bằng IP công khai của VPS:

```bash
curl -v --connect-timeout 5 http://<PUBLIC_IP>:8080/
```

- `Connection refused`: thường là chưa có ứng dụng lắng nghe hoặc ứng dụng chỉ bind vào địa chỉ khác.
- `Connection timed out`: thường do UFW, Cloud Firewall/Security Group, NAT hoặc định tuyến đang chặn.
- Nhận được HTTP response: cổng và ứng dụng có thể được truy cập.
