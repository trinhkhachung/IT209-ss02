# Bài 5: Quản lý quyền sở hữu thư mục Web (Directory Permission Management)

## 1. Mục tiêu

- Hiểu cơ chế phân quyền tệp tin/thư mục trên Linux (`chown` & `chmod`).
- Thiết lập phân quyền tối ưu cho thư mục web tĩnh `/var/www/ptit-web/` để:
  - user **devops** có thể chỉnh sửa mã nguồn mà **không cần** `root`/`sudo`;
  - dịch vụ **Nginx** (chạy dưới user **www-data**) vẫn đọc được và hiển thị website (không lỗi `403 Forbidden`).

## 2. Môi trường thực hiện

| Thành phần | Giá trị |
|---|---|
| Hệ điều hành | Ubuntu (Linux) |
| Web server | Nginx |
| User chạy Nginx | `www-data` (uid=33, gid=33) |
| User chỉnh sửa code | `devops` (uid=1001) |
| Thư mục web | `/var/www/ptit-web/` |

## 3. Mô phỏng trạng thái ban đầu (thư mục do `root` sở hữu)

Trạng thái "có vấn đề" của đề bài được tái hiện: `/var/www/ptit-web/` thuộc sở hữu `root:root`.

```console
$ ls -la /var/www/ptit-web/
total 12
drwxr-xr-x 3 root root 4096 Oct  1 06:57 .
drwxr-xr-x 5 root root 4096 Oct  1 06:57 ..
drwxr-xr-x 2 root root 4096 Oct  1 06:57 html
```

Kiểm chứng vấn đề — user `devops` **không** ghi được file khi chưa phân quyền:

```console
$ su - devops -c 'echo "Update" >> /var/www/ptit-web/html/index.html'
-bash: line 1: /var/www/ptit-web/html/index.html: Permission denied
```

## 4. Thực hiện phân quyền

### 4.1. Đổi chủ sở hữu và nhóm sở hữu (đệ quy)

```bash
# owner = devops, group = www-data (nhóm chạy của Nginx)
sudo chown -R devops:www-data /var/www/ptit-web
```

### 4.2. Thiết lập quyền truy cập (đệ quy)

```bash
# Thư mục: 750 -> rwxr-x---  (chủ: đọc/ghi/đi qua; nhóm: đọc/đi qua; khác: không)
sudo find /var/www/ptit-web -type d -exec chmod 750 {} \;

# Tệp tin: 640 -> rw-r-----  (chủ: đọc/ghi; nhóm: chỉ đọc; khác: không)
sudo find /var/www/ptit-web -type f -exec chmod 640 {} \;
```

> Vì sao dùng `find` thay vì `chmod -R 750`?
> `chmod -R 750` sẽ gán luôn bit thực thi (`x`) cho mọi **file**, khiến file HTML/JS/CSS
> trở thành file thực thi — vừa sai về ngữ nghĩa vừa kém an toàn. Tách riêng `-type d` (750)
> và `-type f` (640) cho đúng quyền của từng loại.
>
> Cách viết tương đương bằng một lệnh duy nhất (dùng `X` hoa = chỉ thêm `x` cho thư mục):
> `sudo chmod -R u=rwX,g=rX,o= /var/www/ptit-web`

## 5. Kiểm tra kết quả

### 5.1. Kiểm tra phân quyền bằng `ls -la`

```console
$ ls -la /var/www/ptit-web/
total 12
drwxr-x--- 3 devops www-data 4096 Oct  1 06:57 .
drwxr-xr-x 5 root   root     4096 Oct  1 06:57 ..
drwxr-x--- 2 devops www-data 4096 Oct  1 06:57 html

$ ls -la /var/www/ptit-web/html/
total 12
drwxr-x--- 2 devops www-data 4096 Oct  1 06:57 .
drwxr-x--- 3 devops www-data 4096 Oct  1 06:57 ..
-rw-r----- 1 devops www-data  237 Oct  1 06:57 index.html
```

- Owner `devops`, group `www-data` cho **cả thư mục gốc và nội dung bên trong** (đệ quy).
- Thư mục `drwxr-x---` (750): chủ `rwx`, nhóm `r-x`, others `---`.
- Tệp `-rw-r-----` (640): chủ `rw-`, nhóm `r--`, others `---`.

### 5.2. Ghi file bằng user `devops` (không dùng `sudo`) — phải thành công

```console
$ su - devops -c 'whoami'
devops

$ su - devops -c 'echo "Update" >> /var/www/ptit-web/html/index.html'
-> WRITE OK (exit 0)

$ tail -3 /var/www/ptit-web/html/index.html
</body>
</html>
Update by devops user - Thu Oct  1 06:58:17 UTC 2026
```

→ `devops` đã ghi trực tiếp vào file **không cần `sudo`** (nhờ quyền `rw` của owner).

### 5.3. Nginx đọc và hiển thị website — không lỗi `403 Forbidden`

```console
$ curl -s -o /dev/null -w "HTTP %{http_code}\n" -H "Host: ptit-web.local" http://127.0.0.1/
HTTP 200

$ curl -s -H "Host: ptit-web.local" http://127.0.0.1/ | tail -3
</body>
</html>
Update by devops user - Thu Oct  1 06:58:17 UTC 2026
```

→ Nginx (user `www-data`, thuộc **group `www-data`**) đọc được nhờ quyền `r--` của nhóm và bit `x` trên thư mục (đi qua), trả về **HTTP 200** kèm nội dung vừa cập nhật. Không có log lỗi `403`/`Permission denied` trong `/var/log/nginx/error.log`.

### 5.4. Người dùng khác (others) — không có quyền truy cập

```console
$ id dun
uid=1000(dun) gid=1000(dun) groups=1000(dun),4(adm),...   # không thuộc www-data

$ su - dun -c 'cat /var/www/ptit-web/html/index.html'
cat: /var/www/ptit-web/html/index.html: Permission denied
```

→ Đúng thiết kế: `others` (`---`) không đọc được nội dung web.

## 6. Giải thích cơ chế phân quyền

Mỗi file/thư mục có **owner** và **group**, cùng 3 bộ quyền: `user (u)`, `group (g)`, `others (o)`.

| Đối tượng | Vai trò | Quyền cần | Ý nghĩa |
|---|---|---|---|
| `devops` (owner) | Lập trình viên | `rw` trên file, `rwx` trên thư mục | Đọc + sửa + tạo/xóa file, không cần `sudo` |
| `www-data` (group) | Tiến trình Nginx | `r` trên file, `r-x` trên thư mục | Nginx đọc file; `x` để "đi qua" thư mục khi phục vụ |
| others | Khách khác | không (`---`) | Không truy cập trực tiếp vào mã nguồn |

Điểm mấu chốt:
- **Chỉ cần đổi owner → `devops`** là giải quyết được yêu cầu "sửa file không cần root".
- **Group `www-data` + bit `r`** cho phép Nginx phục vụ web mà **không** phải cấp quyền cho `others` (an toàn hơn `755/644`).
- **Bit `x` trên thư mục** là điều kiện bắt buộc để Nginx truy cập được vào trong thư mục; thiếu `x` sẽ gây `403 Forbidden` dù file có `r`.

## 7. Kết luận

- Đã chuyển sở hữu `/var/www/ptit-web/` (đệ quy) sang **`devops:www-data`**.
- Đã thiết lập quyền **750 cho thư mục** và **640 cho tệp tin**: owner đọc/ghi, group `www-data` chỉ đọc, others bị chặn.
- **User `devops` ghi được file không cần `sudo`** (kiểm tra thành công).
- **Nginx trả về HTTP 200, không còn `403 Forbidden`**, hiển thị đúng nội dung mới.

## Phụ lục — Cấu hình Nginx cho site

`/etc/nginx/sites-available/ptit-web` (đã `ln -s` sang `sites-enabled/` và `nginx -t && systemctl reload nginx`):

```nginx
server {
    listen 80;
    listen [::]:80;

    root /var/www/ptit-web/html;
    index index.html index.htm;

    server_name ptit-web.local;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Tóm tắt lệnh đã dùng

```bash
sudo chown -R devops:www-data /var/www/ptit-web
sudo find /var/www/ptit-web -type d -exec chmod 750 {} \;
sudo find /var/www/ptit-web -type f -exec chmod 640 {} \;
ls -la /var/www/ptit-web/
su - devops -c 'echo "Update" >> /var/www/ptit-web/html/index.html'
curl -s -o /dev/null -w "HTTP %{http_code}\n" -H "Host: ptit-web.local" http://127.0.0.1/
```
