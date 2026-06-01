# Triển khai Hệ thống Giám sát pfSense với Grafana, InfluxDB, Loki và Promtail

---

## Mục lục

- [Phần 1: Tổng quan kiến trúc hệ thống giám sát](#phần-1-tổng-quan-kiến-trúc-hệ-thống-giám-sát)
  - [1.1. Đặt vấn đề](#11-đặt-vấn-đề)
  - [1.2. Kiến trúc tổng thể](#12-kiến-trúc-tổng-thể)
  - [1.3. Vai trò của từng thành phần](#13-vai-trò-của-từng-thành-phần)
  - [1.4. Kiến trúc luồng thu thập Log — Giải pháp Syslog-ng Relay](#14-kiến-trúc-luồng-thu-thập-log--giải-pháp-syslog-ng-relay)
- [Phần 2: Cài đặt môi trường Docker trên Ubuntu](#phần-2-cài-đặt-môi-trường-docker-trên-ubuntu)
  - [2.1. Cài đặt Docker và Docker Compose](#21-cài-đặt-docker-và-docker-compose)
  - [2.2. Tạo thư mục làm việc](#22-tạo-thư-mục-làm-việc)
- [Phần 3: Xây dựng file cấu hình hệ thống](#phần-3-xây-dựng-file-cấu-hình-hệ-thống)
  - [3.1. Cấu hình Syslog-ng — Trạm trung chuyển Log](#31-cấu-hình-syslog-ng--trạm-trung-chuyển-log)
  - [3.2. Cấu hình Promtail — Thu thập và gán nhãn Log](#32-cấu-hình-promtail--thu-thập-và-gán-nhãn-log)
  - [3.3. Cấu hình Docker Compose — Điều phối toàn bộ cụm](#33-cấu-hình-docker-compose--điều-phối-toàn-bộ-cụm)
- [Phần 4: Khởi chạy và kiểm tra cụm hệ thống](#phần-4-khởi-chạy-và-kiểm-tra-cụm-hệ-thống)
  - [4.1. Khởi động cụm Docker](#41-khởi-động-cụm-docker)
  - [4.2. Kiểm tra trạng thái các container](#42-kiểm-tra-trạng-thái-các-container)
- [Phần 5: Cấu hình pfSense đẩy dữ liệu về hệ thống giám sát](#phần-5-cấu-hình-pfsense-đẩy-dữ-liệu-về-hệ-thống-giám-sát)
  - [5.1. Cài đặt và cấu hình Telegraf — Đẩy chỉ số hiệu năng](#51-cài-đặt-và-cấu-hình-telegraf--đẩy-chỉ-số-hiệu-năng)
  - [5.2. Cấu hình Remote Syslog — Đẩy nhật ký tường lửa](#52-cấu-hình-remote-syslog--đẩy-nhật-ký-tường-lửa)
- [Phần 6: Cấu hình Grafana và triển khai Dashboard](#phần-6-cấu-hình-grafana-và-triển-khai-dashboard)
  - [6.1. Khai báo nguồn dữ liệu InfluxDB](#61-khai-báo-nguồn-dữ-liệu-influxdb)
  - [6.2. Khai báo nguồn dữ liệu Loki](#62-khai-báo-nguồn-dữ-liệu-loki)
  - [6.3. Import Dashboard hiển thị băng thông và hiệu năng](#63-import-dashboard-hiển-thị-băng-thông-và-hiệu-năng)
  - [6.4. Import Dashboard hiển thị nhật ký tường lửa](#64-import-dashboard-hiển-thị-nhật-ký-tường-lửa)
- [Phần 7: Xử lý sự cố — Chuẩn hóa Dashboard JSON](#phần-7-xử-lý-sự-cố--chuẩn-hóa-dashboard-json)
  - [7.1. Phân tích nguyên nhân lỗi No Data](#71-phân-tích-nguyên-nhân-lỗi-no-data)
  - [7.2. Quy trình xử lý và chuẩn hóa file JSON](#72-quy-trình-xử-lý-và-chuẩn-hóa-file-json)
  - [7.3. Nghiệm thu kết quả](#73-nghiệm-thu-kết-quả)

---

## Phần 1: Tổng quan kiến trúc hệ thống giám sát

### 1.1. Đặt vấn đề

Hệ thống tường lửa pfSense vận hành song song trên hai nút (Master và Backup) tạo ra lượng lớn dữ liệu vận hành theo hai luồng riêng biệt:

**Luồng chỉ số hiệu năng (Metrics):** Các thông số như tải CPU, dung lượng RAM sử dụng và lưu lượng băng thông qua từng giao diện mạng. Dữ liệu này có cấu trúc số, thay đổi liên tục theo thời gian và cần được lưu trữ trong cơ sở dữ liệu chuỗi thời gian (Time-Series Database) để truy vấn nhanh.

**Luồng nhật ký sự kiện (Logs):** Các bản ghi về hành động của tường lửa — kết nối nào bị chặn, từ IP nào, đến cổng nào, và tại thời điểm nào. Dữ liệu này là văn bản phi cấu trúc, cần được tổng hợp và lập chỉ mục để tìm kiếm và phân tích.

Việc theo dõi thủ công qua giao diện WebGUI của pfSense chỉ cho phép xem thời gian thực trên từng nút riêng lẻ, không có khả năng tổng hợp dài hạn, so sánh xu hướng hoặc cảnh báo tự động. Giải pháp đặt ra là xây dựng một hệ thống giám sát tập trung, tự động hóa thu thập và trực quan hóa cả hai luồng dữ liệu trên một giao diện duy nhất.

---

### 1.2. Kiến trúc tổng thể

Hệ thống giám sát được triển khai trên một máy chủ Ubuntu riêng biệt (`172.16.1.241`), toàn bộ các dịch vụ chạy dưới dạng container Docker độc lập, được điều phối bởi Docker Compose.

```
[pfSense Master (.10)]  ──Telegraf──▶
                                      InfluxDB :8086 ──▶ Grafana :3000
[pfSense Backup (.11)]  ──Telegraf──▶

[pfSense Master (.10)]  ──UDP:1514──▶
                                      Syslog-ng :1514/UDP ──TCP──▶ Promtail ──▶ Loki :3100 ──▶ Grafana :3000
[pfSense Backup (.11)]  ──UDP:1514──▶
```

Hai luồng dữ liệu hoàn toàn độc lập nhau về đường truyền và cơ sở dữ liệu, nhưng được tích hợp và hiển thị thống nhất tại một điểm duy nhất là Grafana.

---

### 1.3. Vai trò của từng thành phần

**Telegraf** là agent thu thập chỉ số hiệu năng, được cài đặt trực tiếp trên pfSense như một package. Telegraf định kỳ đọc các thông số hệ thống (CPU, RAM, network interface) và đẩy về InfluxDB theo giao thức HTTP.

**InfluxDB** là cơ sở dữ liệu chuỗi thời gian (Time-Series Database), được tối ưu hóa để lưu trữ và truy vấn dữ liệu số có dấu thời gian. Phiên bản 1.8 được sử dụng vì tương thích tốt với các Dashboard mẫu từ cộng đồng.

**Syslog-ng** đóng vai trò trạm trung chuyển (relay) log, tiếp nhận luồng UDP Syslog thô từ pfSense, chuẩn hóa định dạng và chuyển tiếp qua TCP nội bộ cho Promtail. Lý do cần trạm trung chuyển này được giải thích chi tiết tại mục 1.4.

**Promtail** là agent thu thập log phía máy chủ, lắng nghe luồng TCP từ Syslog-ng, gán nhãn (label) cho từng bản ghi log và đẩy vào Loki.

**Loki** là hệ thống tổng hợp log (Log Aggregation System) của Grafana Labs, được thiết kế để lưu trữ và lập chỉ mục log theo nhãn thay vì toàn bộ nội dung, giúp tiết kiệm dung lượng đáng kể so với Elasticsearch.

**Grafana** là nền tảng trực quan hóa dữ liệu, kết nối đồng thời với cả InfluxDB (để vẽ biểu đồ hiệu năng) và Loki (để hiển thị và tìm kiếm log), tổng hợp thành các Dashboard tương tác.

---

### 1.4. Kiến trúc luồng thu thập Log — Giải pháp Syslog-ng Relay

Khi thử nghiệm cấu hình ban đầu để Promtail tiếp nhận trực tiếp luồng UDP Syslog từ pfSense, hệ thống xuất hiện lỗi phân tích cú pháp do **thiếu ký tự xuống dòng** (`\n`) trong gói tin UDP Syslog theo chuẩn RFC 5424.

Giao thức UDP Syslog truyền mỗi bản ghi log dưới dạng một gói datagram độc lập, không có dấu phân cách cuối dòng. Promtail, khi nhận luồng này, không thể xác định ranh giới giữa các bản ghi, dẫn đến các bản ghi bị nối liền nhau hoặc bị cắt sai vị trí.

**Giải pháp:** Chèn Syslog-ng làm trạm trung chuyển với nhiệm vụ duy nhất là bổ sung ký tự `\n` vào cuối mỗi bản ghi trước khi chuyển tiếp.

Luồng xử lý hoàn chỉnh sau khi tái cấu trúc:

```
pfSense ──[UDP :1514, RFC 5424, không có \n]──▶ Syslog-ng ──[TCP nội bộ, có \n]──▶ Promtail ──[HTTP push]──▶ Loki
```

Syslog-ng tiếp nhận gói UDP thô ở cổng `1514`, bổ sung ký tự `\n`, sau đó chuyển tiếp qua TCP đến container Promtail trong cùng mạng Docker. Promtail đọc luồng TCP đã được chuẩn hóa, gán nhãn `{job="pfsense-logs"}` và đẩy sang Loki.

---

## Phần 2: Cài đặt môi trường Docker trên Ubuntu

### 2.1. Cài đặt Docker và Docker Compose

Toàn bộ cụm giám sát chạy trên nền Docker để đảm bảo tính cô lập, dễ triển khai lại và không ảnh hưởng đến hệ điều hành máy chủ. Thực hiện lần lượt các lệnh sau trên terminal của máy Ubuntu:

**Bước 1 — Cập nhật danh sách gói hệ thống:**

```bash
sudo apt update
```

**Bước 2 — Cài đặt Docker Engine và Docker Compose:**

```bash
sudo apt install docker.io docker-compose -y
```

**Bước 3 — Kích hoạt Docker tự khởi động cùng hệ thống:**

```bash
sudo systemctl enable docker --now
```

Lệnh `--now` kết hợp `enable` (đăng ký khởi động tự động) và `start` (khởi động ngay lập tức) thành một bước duy nhất.

---

### 2.2. Tạo thư mục làm việc

Tất cả file cấu hình của hệ thống giám sát được tổ chức trong một thư mục riêng để dễ quản lý và backup:

```bash
mkdir ~/monitoring && cd ~/monitoring
```

Các file cần tạo trong thư mục này:
- `syslog-ng.conf` — Cấu hình trạm trung chuyển Syslog-ng.
- `promtail-config.yml` — Cấu hình agent thu thập log Promtail.
- `docker-compose.yml` — File điều phối toàn bộ cụm container.

---

## Phần 3: Xây dựng file cấu hình hệ thống

### 3.1. Cấu hình Syslog-ng — Trạm trung chuyển Log

Tạo file `syslog-ng.conf` trong thư mục `~/monitoring`:

```bash
nano syslog-ng.conf
```

Dán toàn bộ nội dung cấu hình sau vào:

```
@version: 4.0
@include "scl.conf"

# Nguồn: Tiếp nhận log UDP thô từ pfSense
source s_pfsense_udp {
    network(transport("udp") port(1514) flags(no-parse));
};

# Đích: Chuyển tiếp qua TCP nội bộ đến Promtail, chèn thêm ký tự xuống dòng \n
destination d_promtail_tcp {
    network("promtail" port(1514) transport("tcp") template("${MSG}\n"));
};

log {
    source(s_pfsense_udp);
    destination(d_promtail_tcp);
};
```

**Giải thích cấu hình:**

- `flags(no-parse)`: Yêu cầu Syslog-ng nhận gói tin nguyên vẹn mà không cố gắng phân tích cú pháp Syslog. Điều này quan trọng vì pfSense gửi định dạng RFC 5424 có thể gây lỗi nếu Syslog-ng tự phân tích.
- `template("${MSG}\n")`: Khi chuyển tiếp, bổ sung ký tự `\n` vào cuối mỗi bản ghi — đây là mấu chốt xử lý lỗi phân tích cú pháp của Promtail.
- `"promtail"`: Hostname này được Docker tự động phân giải thành địa chỉ IP của container Promtail trong cùng mạng Docker nội bộ (không cần điền IP cứng).

Lưu và thoát: `Ctrl + O` -> `Enter` -> `Ctrl + X`.

---

### 3.2. Cấu hình Promtail — Thu thập và gán nhãn Log

Tạo file `promtail-config.yml`:

```bash
nano promtail-config.yml
```

Dán nội dung cấu hình sau:

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: pfsense
    syslog:
      listen_address: 0.0.0.0:1514
      listen_protocol: tcp
      idle_timeout: 60s
      label_structured_data: yes
      labels:
        job: "pfsense-logs"
```

**Giải thích cấu hình:**

- `listen_protocol: tcp`: Promtail lắng nghe TCP (nhận từ Syslog-ng) thay vì UDP trực tiếp từ pfSense.
- `labels.job: "pfsense-logs"`: Nhãn này được gán cho mọi bản ghi log nhận được. Nhãn phải khớp với cấu hình truy vấn trong Dashboard Grafana — đây là điểm then chốt xử lý lỗi "No Data" được đề cập ở Phần 7.
- `clients.url`: Địa chỉ nội bộ Docker của Loki — Promtail đẩy log đến đây sau khi xử lý.

Lưu và thoát: `Ctrl + O` -> `Enter` -> `Ctrl + X`.

---

### 3.3. Cấu hình Docker Compose — Điều phối toàn bộ cụm

Tạo file `docker-compose.yml`:

```bash
nano docker-compose.yml
```

Dán nội dung cấu hình sau:

```yaml
version: '3.8'

services:
  influxdb:
    image: influxdb:1.8-alpine
    container_name: influxdb
    restart: always
    ports:
      - "8086:8086"
    environment:
      - INFLUXDB_DB=pfsense
    volumes:
      - influxdb_data:/var/lib/influxdb

  loki:
    image: grafana/loki:2.9.0
    container_name: loki
    restart: always
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - loki_data:/loki

  syslog-ng:
    image: balabit/syslog-ng:latest
    container_name: syslog-ng
    restart: always
    ports:
      - "1514:1514/udp"
    volumes:
      - ./syslog-ng.conf:/etc/syslog-ng/syslog-ng.conf

  promtail:
    image: grafana/promtail:2.9.0
    container_name: promtail
    restart: always
    volumes:
      - ./promtail-config.yml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml
    depends_on:
      - loki

  grafana:
    image: grafana/grafana-oss:latest
    container_name: grafana
    restart: always
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana

volumes:
  influxdb_data:
  loki_data:
  grafana_data:
```

**Giải thích cấu hình:**

- `restart: always`: Đảm bảo mọi container tự khởi động lại khi bị crash hoặc khi máy chủ Ubuntu reboot.
- `volumes` (named volumes): Dữ liệu của InfluxDB, Loki và Grafana được lưu trữ trong Docker named volumes — tách biệt khỏi container, đảm bảo dữ liệu không bị mất khi container bị xóa hoặc cập nhật.
- `depends_on: loki`: Đảm bảo container Loki khởi động trước Promtail, tránh lỗi kết nối khi Promtail cố gắng đẩy log đến Loki chưa sẵn sàng.
- `syslog-ng` chỉ expose cổng UDP `1514` ra ngoài (để nhận từ pfSense). Cổng TCP nội bộ đến Promtail không cần expose vì giao tiếp diễn ra trong mạng Docker nội bộ.

![Cấu trúc thư mục và các file cấu hình đã tạo](https://github.com/user-attachments/assets/f8c37f60-fe0a-4db5-8734-e1afcc39c626)

Lưu và thoát: `Ctrl + O` -> `Enter` -> `Ctrl + X`.

---

## Phần 4: Khởi chạy và kiểm tra cụm hệ thống

### 4.1. Khởi động cụm Docker

Đứng tại thư mục `~/monitoring`, chạy lệnh sau để Docker Compose tải image và khởi chạy tất cả container:

```bash
sudo docker-compose up -d
```

Tham số `-d` (detached mode) cho phép cụm chạy nền, không chiếm terminal. Docker sẽ tự động pull các image từ Docker Hub nếu chưa có local. Quá trình này mất khoảng 1–3 phút tùy tốc độ mạng.

![Quá trình Docker Compose kéo image và khởi chạy container](https://github.com/user-attachments/assets/270e2611-242d-4148-9904-1c4982867631)

---

### 4.2. Kiểm tra trạng thái các container

Sau khi quá trình khởi động hoàn tất, kiểm tra trạng thái tất cả container:

```bash
sudo docker-compose ps
```

Tất cả container phải ở trạng thái `Up`. Nếu có container ở trạng thái `Exit` hoặc `Restarting`, kiểm tra log của container đó bằng lệnh:

```bash
sudo docker logs <tên_container>
```

Để xác nhận Grafana hoạt động, mở trình duyệt trên máy Windows và truy cập:

```
http://172.16.1.241:3000
```

Màn hình đăng nhập Grafana xuất hiện với thông tin đăng nhập mặc định: `admin / admin`.

![Màn hình đăng nhập Grafana lần đầu](https://github.com/user-attachments/assets/bcff96ba-824c-4838-bd8c-2e9ea483f2c8)

![Trạng thái các container sau khi khởi động](https://github.com/user-attachments/assets/c34716cc-7280-4cf4-a3a0-0add6b70290a)

---

## Phần 5: Cấu hình pfSense đẩy dữ liệu về hệ thống giám sát

Bước này được thực hiện trên **cả hai** giao diện WebGUI của pfSense Master và pfSense Backup để đảm bảo dữ liệu từ cả hai nút đều được thu thập tập trung về máy Ubuntu (`172.16.1.241`).

### 5.1. Cài đặt và cấu hình Telegraf — Đẩy chỉ số hiệu năng

Telegraf là agent thu thập metrics được cài đặt trực tiếp trên pfSense dưới dạng package. Nó định kỳ đọc các chỉ số hệ thống và đẩy về InfluxDB.

**Bước 1 — Cài đặt package Telegraf:**

1. Trên pfSense, vào **System** -> **Package Manager** -> tab **Available Packages**.
2. Tìm kiếm `telegraf` và bấm **Install**.

![Cài đặt package Telegraf trên pfSense](https://github.com/user-attachments/assets/03a2dee0-d920-44af-96a3-2526783d4ab2)

**Bước 2 — Cấu hình Telegraf:**

1. Sau khi cài xong, vào **Services** -> **Telegraf**.
2. Tích chọn **Enable Telegraf**.
3. Tại mục **Telegraf Output**, chọn **InfluxDB**.
4. Tại mục **InfluxDB Server**, điền: `http://172.16.1.241:8086`
5. Tại mục **InfluxDB Database**, điền: `pfsense`
6. Tại phần **Default Telegraf Plugins**, tích chọn các plugin: `cpu`, `memory`, `system`, `interfaces`.

   - `cpu`: Ghi lại phần trăm sử dụng CPU theo từng core.
   - `memory`: Ghi lại dung lượng RAM đã dùng, còn trống và cache.
   - `system`: Ghi lại uptime, load average của hệ thống.
   - `interfaces`: Ghi lại lưu lượng bytes in/out và số gói tin qua từng giao diện mạng — đây là nguồn dữ liệu băng thông quan trọng nhất.

7. Kéo xuống dưới cùng và bấm **Save**.

![Dashboard Grafana hiển thị dữ liệu CPU, RAM và băng thông từ Telegraf](https://github.com/user-attachments/assets/3155ffb1-5bb4-4c6e-8567-d0ec8c2e4495)

![Chi tiết biểu đồ giao diện mạng từ Telegraf](https://github.com/user-attachments/assets/a56b6115-4df6-47bf-8c40-72c12022ce09)

---

### 5.2. Cấu hình Remote Syslog — Đẩy nhật ký tường lửa

pfSense hỗ trợ tích hợp sẵn tính năng đẩy log ra máy chủ Syslog từ xa, không cần cài thêm package.

**Bước 1 — Truy cập cấu hình Syslog:**

1. Vào **Status** -> **System Logs** -> tab **Settings**.

![Màn hình cấu hình System Logs Settings trên pfSense](https://github.com/user-attachments/assets/3d8a19c6-f01e-4482-b8cb-b9d6b180893b)

**Bước 2 — Cấu hình Remote Log:**

1. Kéo xuống tìm mục **Remote Log Options**.
2. Tích chọn **Enable Remote Logging**.
3. Tại mục **Source Address**, chọn **LAN** (đảm bảo pfSense gửi log qua giao diện LAN kết nối với máy Ubuntu).
4. Tại mục **IP Protocol**, chọn **IPv4**.
5. Tại mục **Remote Log Servers**, điền: `172.16.1.241:1514`
6. Tại mục **Remote Syslog Contents**, tích chọn:
   - **System Events**: Log các sự kiện hệ thống (khởi động, dịch vụ, lỗi).
   - **Firewall Events**: Log tất cả hành động chặn/cho phép của tường lửa — đây là nguồn dữ liệu chính cho Dashboard phân tích bảo mật.
7. Bấm **Save**.

![Cấu hình Remote Syslog trên pfSense](https://github.com/user-attachments/assets/1847f944-0a59-4ee0-a983-a692033031f5)

Lặp lại toàn bộ Bước 5.1 và 5.2 trên nút pfSense còn lại (Master hoặc Backup tùy thứ tự thực hiện).

---

## Phần 6: Cấu hình Grafana và triển khai Dashboard

### 6.1. Khai báo nguồn dữ liệu InfluxDB

Grafana cần được chỉ định rõ nguồn dữ liệu trước khi có thể vẽ biểu đồ.

1. Đăng nhập vào Grafana tại `http://172.16.1.241:3000`.
2. Vào menu **Connections** -> **Data sources** -> bấm **Add data source**.
3. Chọn **InfluxDB**.
4. Tại mục **Query Language**, giữ nguyên **InfluxQL**.
5. Tại mục **HTTP** -> **URL**, điền: `http://influxdb:8086`

   > Vì Grafana và InfluxDB cùng chạy trong một mạng Docker nội bộ, Grafana có thể phân giải tên container `influxdb` trực tiếp mà không cần dùng địa chỉ IP ngoài.

6. Kéo xuống phần **InfluxDB Details**, tại ô **Database** điền: `pfsense`
7. Bấm **Save & test**. Thông báo màu xanh lá "Data source is working" xác nhận kết nối thành công.

![Cấu hình InfluxDB Data Source trong Grafana](https://github.com/user-attachments/assets/6dbfd02f-99a9-42d8-8dd8-e104c8fbe86e)

![Kết quả Save & test thành công](https://github.com/user-attachments/assets/6f633a98-db3b-40cb-8227-0737ada90151)

---

### 6.2. Khai báo nguồn dữ liệu Loki

1. Quay lại màn hình **Data sources**, bấm **Add data source**.
2. Chọn **Loki**.
3. Tại mục **HTTP** -> **URL**, điền: `http://loki:3100`
4. Bấm **Save & test**. Thông báo "Data source successfully connected" xác nhận thành công.

![Cấu hình Loki Data Source trong Grafana](https://github.com/user-attachments/assets/0f4cf645-21eb-419f-914d-ac9d02a3ec12)

![Cấu hình Loki Data Source — xác nhận kết nối](https://github.com/user-attachments/assets/70b4ead8-ff1d-431f-83f5-db7b3623e7e7)

---

### 6.3. Import Dashboard hiển thị băng thông và hiệu năng

Cộng đồng Grafana đã xây dựng sẵn các Dashboard mẫu chất lượng cao dành riêng cho pfSense. Thay vì tự vẽ từng panel, sử dụng tính năng Import để kéo mẫu từ thư viện Grafana.

1. Ở menu bên trái, vào **Dashboards** -> **New** -> **Import**.
2. Tại ô **Import via grafana.com**, nhập ID Dashboard: `224410`, bấm **Load**.
3. Ở màn hình xác nhận, tại mục chọn Data Source, chọn nguồn **InfluxDB** vừa tạo ở Bước 6.1.
4. Bấm **Import**.

Dashboard sẽ lập tức hiển thị các biểu đồ CPU, RAM, băng thông và uptime của cả hai nút pfSense.

---

### 6.4. Import Dashboard hiển thị nhật ký tường lửa

1. Lặp lại thao tác vào **Dashboards** -> **New** -> **Import**.
2. Nhập ID Dashboard: `22722`, bấm **Load**.

   > **Lưu ý:** Trước khi Import, cần thực hiện bước chuẩn hóa file JSON được mô tả tại Phần 7. Nếu bỏ qua bước này, Dashboard sẽ báo lỗi "No data" trên tất cả các panel.

3. Tại mục chọn Data Source, chọn nguồn **Loki**.
4. Bấm **Import**.

![Giao diện Import Dashboard qua ID trong Grafana](https://github.com/user-attachments/assets/92a51d7a-2434-4249-899a-77141cb64cc7)

---

## Phần 7: Xử lý sự cố — Chuẩn hóa Dashboard JSON

### 7.1. Phân tích nguyên nhân lỗi No Data

Sau khi Import Dashboard ID `22722`, toàn bộ các panel báo lỗi "No data" mặc dù log đã được Promtail đẩy vào Loki thành công.

**Nguyên nhân gốc rễ:** Hầu hết các Dashboard mẫu trên cộng đồng Grafana được tác giả xây dựng với cấu hình Promtail sử dụng nhãn mặc định:

```
{hostname=~"$hostname", syslog_app="filterlog"}
```

Trong khi đó, cấu hình Promtail của hệ thống này (mục 3.2) sử dụng nhãn tối giản:

```
{job="pfsense-logs"}
```

Grafana truy vấn Loki theo nhãn được định nghĩa trong Dashboard. Vì nhãn không khớp, Loki không tìm thấy bản ghi nào phù hợp và trả về kết quả rỗng. Đây là lỗi **label mismatch** — không liên quan đến dữ liệu log, chỉ là sai lệch trong điều kiện truy vấn.

---

### 7.2. Quy trình xử lý và chuẩn hóa file JSON

Thay vì chỉnh sửa thủ công từng panel trong giao diện đồ họa Grafana (rất tốn thời gian và dễ bỏ sót), phương pháp tối ưu là can thiệp trực tiếp vào mã nguồn file JSON của Dashboard trước khi Import:

**Bước 1 — Tải file JSON của Dashboard:**

Truy cập `https://grafana.com/grafana/dashboards/22722`, tải file JSON về máy.

**Bước 2 — Mở file bằng trình soạn thảo hỗ trợ tìm kiếm hàng loạt:**

Sử dụng VS Code hoặc Notepad++ để có tính năng Find & Replace toàn bộ file.

**Bước 3 — Thay thế nhãn truy vấn chính:**

Mở tính năng Find & Replace (`Ctrl + H`):

- Tìm: `hostname=~\"$hostname\", syslog_app=\"filterlog\"`
- Thay bằng: `job=\"pfsense-logs\"`

Bấm **Replace All**.

**Bước 4 — Thay thế nhãn trong phần Variables:**

Thực hiện thêm một lần tìm kiếm phụ:

- Tìm: `{syslog_app=\"filterlog\"}`
- Thay bằng: `{job=\"pfsense-logs\"}`

Bấm **Replace All**.

**Bước 5 — Lưu file JSON đã chuẩn hóa.**

---

### 7.3. Nghiệm thu kết quả

Sau khi Import file JSON đã chuẩn hóa với nguồn dữ liệu Loki:

1. Vào **Dashboards** -> **New** -> **Import**.
2. Upload file JSON đã chỉnh sửa (thay vì dùng ID).
3. Chọn Data Source **Loki** và bấm **Import**.

Toàn bộ các panel phân tích log tường lửa lập tức hiển thị dữ liệu theo thời gian thực: Top Source IPs, Blocked Ports, Action Timeline, và phân bố sự kiện theo giao thức.

![Dashboard phân tích log tường lửa pfSense sau khi chuẩn hóa — hiển thị đầy đủ dữ liệu](https://github.com/user-attachments/assets/96a61638-9a85-46df-93ec-3eddd395445b)

![Biểu đồ chi tiết luồng traffic và các IP bị chặn theo thời gian thực](https://github.com/user-attachments/assets/18e7b7ed-4f93-4550-b077-d477fced9f02)

Lỗi lệch múi giờ và mất định dạng Syslog đều đã được xử lý triệt để ở tầng Syslog-ng, mang lại bảng điều khiển giám sát chính xác với dữ liệu khớp thời gian thực tế.
