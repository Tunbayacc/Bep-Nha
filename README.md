# Bếp Nhà — Website công thức nấu ăn

Đề tài 29 — Môn Triển khai và Quản trị Hệ thống Phần mềm.

- Sinh viên: La Văn Tuấn
- MSSV: DTC245180118
- Lớp: KTPM-K23A

## 1. Chức năng

- Xem danh sách và chi tiết công thức.
- Phân loại công thức theo danh mục.
- Tìm kiếm theo từ khóa trong tiêu đề và nội dung.
- Quản trị thêm, sửa, xóa công thức bằng WordPress.
- Mỗi công thức có nguyên liệu, định lượng và cách chế biến.

## 2. Kiến trúc

Trình duyệt → Nginx → WordPress → MySQL 8.4.

- phpMyAdmin: quản lý database qua SSH tunnel.
- cAdvisor: metrics tài nguyên container.
- Nginx exporter: metrics Nginx.
- MySQL exporter: metrics database.
- Prometheus: thu thập metrics.
- Grafana: hiển thị dashboard và truy vấn log.
- Promtail → Loki: thu thập và lưu log container.
- Alertmanager: dịch vụ có sẵn trong môi trường thực hành.
- Service `mysql` dùng MariaDB là hệ thống cũ được giữ lại;
  WordPress hiện kết nối service `mysql8`.

Tất cả dịch vụ được triển khai bằng Docker Compose.

## 3. Cấu trúc repository

- docker-compose.yml: cấu hình dịch vụ, mạng và volume.
- nginx/: reverse proxy, security headers và stub_status.
- prometheus/: scrape jobs và alert rules.
- grafana/dashboards/: dashboard JSON.
- loki/: cấu hình lưu trữ log.
- promtail/: khám phá container và gửi log.
- alertmanager/: cấu hình Alertmanager.
- wordpress-export/: nội dung XML, theme ZIP và ảnh uploads.
- docs/: hướng dẫn và ảnh minh chứng.

## 4. Chuẩn bị cấu hình

Cần Ubuntu Server, Docker Engine và Docker Compose v2.

Tạo các file không đưa lên Git:

- .env: WORDPRESS_VOLUME_NAME.
- .env.mysql8: MYSQL_DATABASE, MYSQL_USER, MYSQL_PASSWORD,
  MYSQL_ROOT_PASSWORD.
- .env.wordpress: WORDPRESS_DB_HOST, WORDPRESS_DB_USER,
  WORDPRESS_DB_PASSWORD, WORDPRESS_DB_NAME.
- .env.mariadb: các biến MYSQL_* của service MariaDB cũ.
- secrets/mysqld-exporter.cnf: thông tin tài khoản exporter.

Tham khảo tên biến trong .env.example.
Mật khẩu WordPress phải trùng tài khoản ứng dụng trong MySQL.

Trên máy mới:

```bash
docker volume create recipe_wordpress_data
docker compose config --quiet
docker compose up -d mysql8 wordpress nginx
```

Kiểm tra MySQL healthy trước khi cài đặt WordPress.

Các biến khởi tạo MySQL chỉ có tác dụng khi volume dữ liệu
chưa được khởi tạo. Sửa file môi trường không tự đổi mật khẩu
của tài khoản đã tồn tại trong database.

## 5. Khôi phục nội dung trên máy mới

1. Cài đặt WordPress và tạo tài khoản quản trị mới.
2. Cài theme từ wordpress-export/bep-nha-theme.zip.
3. Nếu WordPress báo theme đã tồn tại, kiểm tra đúng theme
   trước khi chọn thay thế, rồi kích hoạt.
4. Dùng Tools → Import → WordPress để nhập bep-nha.xml.
   Có thể cần cài WordPress Importer.
5. Sao chép uploads vào wp-content/uploads của container.
6. Kiểm tra ảnh đại diện, menu và các liên kết sau khi nhập.
7. Đổi URL cũ sang địa chỉ máy mới bằng công cụ hỗ trợ
   dữ liệu serialized; không thay chuỗi trực tiếp toàn bộ SQL.
8. Kiểm tra trang chủ, công thức, danh mục và tìm kiếm.

XML, theme và uploads không thay thế cho bản backup toàn bộ
database. Các liên kết và tham chiếu ảnh cần được kiểm tra
sau khi nhập.

Chưa kiểm thử quy trình khôi phục trên một máy mới.

## 6. Giám sát

Tạo tài khoản MySQL exporter với giới hạn 3 kết nối:

```sql
CREATE USER 'exporter'@'%'
IDENTIFIED BY 'REPLACE_WITH_MONITORING_PASSWORD'
WITH MAX_USER_CONNECTIONS 3;

GRANT PROCESS, REPLICATION CLIENT, SELECT
ON *.* TO 'exporter'@'%';
```

Tạo secrets/mysqld-exporter.cnf:

```ini
[client]
user=exporter
password=REPLACE_WITH_MONITORING_PASSWORD
```

File phải đọc được bởi UID 65534 của container exporter.
Trong môi trường thực hành:

```bash
sudo chown 65534:65534 secrets/mysqld-exporter.cnf
sudo chmod 400 secrets/mysqld-exporter.cnf
```

Sau khi chuẩn bị đầy đủ cấu hình:

```bash
docker compose up -d
docker compose ps
```

Trong Grafana:
- Thêm nguồn Prometheus: http://prometheus:9090.
- Import JSON trong grafana/dashboards/.
- Chọn đúng nguồn Prometheus khi import.

Prometheus cần có các target UP:
prometheus, nginx, cadvisor và mysql.

Truy vấn xác nhận:

```promql
nginx_up
mysql_up
```

Cả hai cần trả về 1.

## 7. Log tập trung

Trong Grafana, thêm nguồn Loki: http://loki:3100.

Ví dụ LogQL:

```logql
{service="nginx"}
```

```logql
{service="nginx"} |= "bepnha-test-01"
```

```logql
sum(count_over_time({service="nginx"} |= "bepnha-test-01" [5m]))
```

Tạo log kiểm thử, thay SERVER_IP bằng IP Ubuntu:

```bash
curl "http://SERVER_IP/?demo_log=bepnha-test-01"
```

Chọn khoảng thời gian chứa lượt truy cập khi truy vấn.
Promtail được sử dụng theo yêu cầu đề học phần; công cụ này
đã hết vòng đời hỗ trợ.

## 8. Địa chỉ truy cập

Thay SERVER_IP bằng IP Ubuntu Server:

- Website: http://SERVER_IP
- WordPress Admin: http://SERVER_IP/wp-admin/
- Grafana: http://SERVER_IP:3000
- Prometheus: http://SERVER_IP:9090

Tên service như prometheus, mysql8 và loki dùng trong
mạng Docker; trình duyệt Windows dùng IP Ubuntu.

phpMyAdmin được bind tại 127.0.0.1:8081 trên Ubuntu.
Từ PowerShell Windows:

```powershell
ssh -N -L 127.0.0.1:8081:127.0.0.1:8081 USER@SERVER_IP
```

Sau đó mở http://127.0.0.1:8081 và đăng nhập bằng tài khoản
MySQL phù hợp.

## 9. Hardening và giới hạn

Đã áp dụng:
- MySQL dùng mạng database riêng, internal.
- MySQL không publish cổng ra host.
- WordPress không publish cổng 8080.
- phpMyAdmin truy cập qua SSH tunnel.
- MySQL exporter chạy bằng UID không phải root.
- Tài khoản giám sát riêng và giới hạn kết nối.
- Nginx có security headers.
- File bí mật và backup được bỏ qua bởi Git.

Giới hạn:
- Website hiện sử dụng HTTP.
- Một số dịch vụ giám sát vẫn mở cổng trên mạng LAN.
- cAdvisor và Promtail có quyền truy cập thông tin host/Docker.
- File môi trường không phải kho bí mật được mã hóa.
- Promtail positions hiện lưu tại /tmp trong container.
- Một số image còn dùng tag latest.
- Môi trường thực hành hiện dùng Ubuntu 18.04.

## 10. Vận hành

```bash
docker compose ps
docker compose logs --tail=50 SERVICE_NAME
docker compose stop
docker compose start
```

Không dùng `docker compose down -v` khi cần giữ dữ liệu.
Bản sao lưu chứa dữ liệu và thông tin bí mật phải được bảo vệ,
không đưa lên GitHub.
