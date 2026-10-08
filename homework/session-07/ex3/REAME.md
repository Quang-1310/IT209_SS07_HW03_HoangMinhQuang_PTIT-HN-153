Thư mục nộp bài gồm:

- `spring-app.service`: dịch vụ Systemd chạy ứng dụng bằng user giới hạn `spring-runner`.
- `README.md`: các bước triển khai và kết quả kiểm tra.

## 1. Khởi tạo MySQL

Đăng nhập MySQL bằng tài khoản quản trị:

```bash
sudo mysql
```

Chạy các câu lệnh sau:

```sql
CREATE DATABASE springboot_db
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE USER 'spring-admin'@'localhost'
  IDENTIFIED BY 'SpringSecure@123';

GRANT ALL PRIVILEGES ON springboot_db.*
  TO 'spring-admin'@'localhost';

FLUSH PRIVILEGES;
```

Kiểm tra quyền đã cấp:

```sql
SHOW GRANTS FOR 'spring-admin'@'localhost';
```

Có thể kiểm tra đăng nhập bằng user của ứng dụng:

```bash
mysql -u spring-admin -p springboot_db
```

## 2. Tạo user hệ thống và cài đặt ứng dụng

Tạo tài khoản hệ thống không có khả năng đăng nhập shell:

```bash
sudo useradd --system --user-group --shell /usr/sbin/nologin spring-runner
```

Tạo thư mục ứng dụng, sao chép file JAR và phân quyền:

```bash
sudo install -d -o spring-runner -g spring-runner -m 0750 /opt/spring-app
sudo install -o spring-runner -g spring-runner -m 0550 app.jar /opt/spring-app/app.jar
```

Tạo file chứa mật khẩu riêng, chỉ `root` được phép đọc:

```bash
sudo install -d -o root -g root -m 0755 /etc/spring-app
sudo install -o root -g root -m 0600 /dev/null /etc/spring-app/spring-app.env
sudoedit /etc/spring-app/spring-app.env
```

Thêm đúng một dòng sau vào file rồi lưu lại:

```dotenv
SPRING_DATASOURCE_PASSWORD=SpringSecure@123
```

Mật khẩu được tách khỏi unit file vì `/etc/systemd/system/spring-app.service`
thường có quyền `0644`. Dịch vụ đọc mật khẩu qua khai báo `EnvironmentFile`.

## 3. Cài đặt và khởi động dịch vụ

Từ thư mục chứa bài nộp, sao chép unit file vào Systemd:

```bash
sudo cp spring-app.service /etc/systemd/system/spring-app.service
sudo chmod 0644 /etc/systemd/system/spring-app.service
sudo systemctl daemon-reload
sudo systemctl enable --now spring-app.service
```

Nếu MySQL trên máy có unit tên `mysqld.service` thay vì `mysql.service`, thay
`mysql.service` tại dòng `After` trong `spring-app.service` bằng tên unit thực tế,
sau đó chạy lại `sudo systemctl daemon-reload` và khởi động lại dịch vụ.

## 4. Kiểm tra kết quả

Kiểm tra trạng thái dịch vụ:

```bash
sudo systemctl status spring-app.service --no-pager
```

Kết quả dự kiến sau khi triển khai thành công:

```text
● spring-app.service - Spring Boot Application
     Loaded: loaded (/etc/systemd/system/spring-app.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-10-07 22:15:03 +07; 2min 18s ago
   Main PID: 2146 (java)
      Tasks: 28 (limit: 4557)
     Memory: 186.4M
        CPU: 8.731s
     CGroup: /system.slice/spring-app.service
             └─2146 /usr/bin/java -jar /opt/spring-app/app.jar

Oct 07 22:15:03 spring-server systemd[1]: Started spring-app.service - Spring Boot Application.
Oct 07 22:15:07 spring-server java[2146]: Started Application in 3.842 seconds
```

Kiểm tra cổng `8082`:

```bash
sudo ss -tlnp | grep 8082
```

Kết quả dự kiến sau khi ứng dụng đã khởi động:

```text
LISTEN 0      100                *:8082             *:*    users:(("java",pid=2146,fd=12))
```

Kiểm tra tiến trình Java chạy đúng user:

```bash
ps -o user,pid,cmd -C java
```

Kết quả dự kiến:

```text
USER             PID CMD
spring-runner    2146 /usr/bin/java -jar /opt/spring-app/app.jar
```

Kết quả mong đợi là service có trạng thái `active (running)`, cổng `8082` ở
trạng thái `LISTEN`, và tiến trình `/usr/bin/java -jar /opt/spring-app/app.jar`
thuộc user `spring-runner`.

Nếu ứng dụng không khởi động, xem log bằng:

```bash
sudo journalctl -u spring-app.service -n 100 --no-pager
```