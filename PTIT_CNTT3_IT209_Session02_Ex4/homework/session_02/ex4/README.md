# Bài 4: Cấu hình tường lửa bảo vệ máy chủ (UFW & Cloud Firewall Integration)

## 1. Cấu hình UFW trên Ubuntu (Lớp 1)

Thực hiện các lệnh sau trên Droplet để cấu hình tường lửa UFW:

```bash
# Thiết lập chính sách mặc định
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Cho phép kết nối SSH (cổng 22)
sudo ufw allow 22/tcp

# Cho phép lưu lượng HTTP (cổng 80)
sudo ufw allow 80/tcp

# Kích hoạt UFW
sudo ufw enable
```

Kiểm tra trạng thái của UFW:
```bash
sudo ufw status verbose
```
*(Học viên chèn ảnh chụp màn hình đầu ra của lệnh `sudo ufw status verbose` tại đây)*

## 2. Cấu hình DigitalOcean Cloud Firewall (Lớp 2)

1. Truy cập vào **DigitalOcean Console**.
2. Điều hướng tới phần **Networking** > **Firewalls** và nhấn **Create Firewall**.
3. Thiết lập **Inbound Rules**:
   - **Type:** SSH, **Port:** 22, **Sources:** All IPv4, All IPv6 (hoặc IP cá nhân của bạn).
   - **Type:** HTTP, **Port:** 80, **Sources:** All IPv4, All IPv6.
4. (Tuỳ chọn) Thiết lập **Outbound Rules**: Cho phép tất cả lưu lượng đi ra.
5. Ở phần **Apply to Droplets**, gán Droplet của bạn vào Cloud Firewall này.
6. Nhấn **Create Firewall**.

*(Học viên chèn ảnh chụp màn hình thiết lập Cloud Firewall trên DigitalOcean Console tại đây)*

## 3. Kiểm tra kết quả
- Từ máy cá nhân, thử kết nối SSH vào Droplet: Kết nối thành công.
- Truy cập vào IP Droplet qua trình duyệt (HTTP - cổng 80): Truy cập thành công.
- Thử ping hoặc quét (quét bằng nmap) thử một cổng khác không mở (ví dụ: 8080): Phải bị chặn/timeout.
