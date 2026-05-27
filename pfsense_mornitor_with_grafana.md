: **Cài đặt Docker và dựng cụm giám sát pfsense (Grafana, InfluxDB, Loki, Promtail)**.

### Bước 1: Cài đặt Docker và Docker Compose

Copy và chạy lần lượt các lệnh sau trên terminal của máy Ubuntu:

1. Cập nhật hệ thống:
```bash
sudo apt update

```


2. Cài đặt Docker và Docker Compose:
```bash
sudo apt install docker.io docker-compose -y

```


3. Cấp quyền cho Docker tự khởi động cùng hệ thống:
```bash
sudo systemctl enable docker --now

```



---

### Bước 2: Tạo các file cấu hình

Chúng ta sẽ tạo thư mục `monitoring` và nhét 2 file cấu hình vào đó.

1. Tạo và truy cập vào thư mục:
```bash
mkdir ~/monitoring && cd ~/monitoring

```


2. **Tạo file cấu hình cho Promtail:**
Gõ lệnh sau để mở trình soạn thảo:
```bash
nano promtail-config.yml

```


Sau đó **Copy toàn bộ đoạn code dưới đây** và **Paste (chuột phải)** vào màn hình nano:
```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: pfsense-syslog
    syslog:
      listen_address: 0.0.0.0:1514
      idle_timeout: 60s
      label_structured_data: yes
      labels:
        job: "pfsense-logs"
    relabel_configs:
      - source_labels: ['__syslog_connection_ip_address']
        target_label: 'pfsense_ip'
      - source_labels: ['__syslog_message_app_name']
        target_label: 'program'

```


Bấm `Ctrl + O` -> `Enter` để lưu -> Bấm `Ctrl + X` để thoát.*
3. **Tạo file Docker Compose:**
Gõ lệnh:
```bash
nano docker-compose.yml

```


Paste toàn bộ đoạn code dưới đây vào:
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

  promtail:
    image: grafana/promtail:2.9.0
    container_name: promtail
    restart: always
    ports:
      - "1514:1514/udp"
      - "1514:1514/tcp"
    volumes:
      - ./promtail-config.yml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml

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

<img width="1059" height="206" alt="image" src="https://github.com/user-attachments/assets/f8c37f60-fe0a-4db5-8734-e1afcc39c626" />

 *Lưu và thoát tương tự: `Ctrl + O` -> `Enter` -> `Ctrl + X`.*

---

### Bước 3: Khởi chạy cụm hệ thống

Vẫn đang đứng ở thư mục `monitoring`, bạn gõ lệnh cuối cùng này để tải và chạy tất cả:

```bash
sudo docker-compose up -d

```

<img width="1157" height="261" alt="image" src="https://github.com/user-attachments/assets/270e2611-242d-4148-9904-1c4982867631" />

Lúc này máy Ubuntu sẽ bắt đầu kéo (pull) các image của Grafana, InfluxDB, Loki về và tự động chạy. Quá trình này mất khoảng 1-2 phút tùy tốc độ mạng.

Sau khi chạy xong, để kiểm tra xem mọi thứ đã hoạt động chưa, bạn mở trình duyệt web trên máy tính thật (Windows) và gõ:
**`http://172.16.1.241:3000`**

<img width="1554" height="841" alt="image" src="https://github.com/user-attachments/assets/bcff96ba-824c-4838-bd8c-2e9ea483f2c8" />

Kiểm tra lại các service:

<img width="1489" height="572" alt="image" src="https://github.com/user-attachments/assets/c34716cc-7280-4cf4-a3a0-0add6b70290a" />

Bây giờ Grafana đang giống như một cái tivi mới mua, chưa có kênh nào cả. Bạn cần làm 2 việc: **Cắm ăng-ten (Thêm nguồn dữ liệu)** và **Bật kênh (Import biểu đồ)**.

Hãy làm lần lượt các bước sau ngay trên giao diện web Grafana:

### Bước 1: Khai báo nguồn dữ liệu (Data Sources)

**1. Thêm cơ sở dữ liệu băng thông (InfluxDB):**

* Ở menu bên trái, bấm vào biểu tượng bánh răng ⚙️ (hoặc **Connections** / **Data sources** tùy phiên bản Grafana).
* Bấm nút **Add data source**.
* Chọn **InfluxDB**.
* Tại mục *Query Language*, giữ nguyên là **InfluxQL**.
* Tại mục *HTTP > URL*, bạn điền: `http://influxdb:8086` *(Vì chúng đang chạy chung 1 cụm Docker nên Grafana có thể gọi thẳng tên InfluxDB).*
* Kéo xuống phần *InfluxDB Details*, ô *Database* bạn gõ chữ: `pfsense`.
* Kéo xuống dưới cùng, bấm **Save & test**. Nếu hiện thông báo màu xanh lá "Data source is working" là thành công.
<img width="1549" height="841" alt="image" src="https://github.com/user-attachments/assets/6dbfd02f-99a9-42d8-8dd8-e104c8fbe86e" />

<img width="1515" height="688" alt="image" src="https://github.com/user-attachments/assets/6f633a98-db3b-40cb-8227-0737ada90151" />

**2. Thêm cơ sở dữ liệu Log (Loki):**

* Quay lại màn hình Data sources, tiếp tục bấm **Add data source**.
* Lần này bạn chọn **Loki**.
* Tại mục *HTTP > URL*, bạn điền: `http://loki:3100`
* Kéo xuống dưới cùng, bấm **Save & test**. Thấy thông báo màu xanh lá là thành công.
<img width="1542" height="700" alt="image" src="https://github.com/user-attachments/assets/0f4cf645-21eb-419f-914d-ac9d02a3ec12" />

---

### Bước 2: Import các mẫu Dashboard có sẵn

Thay vì phải tự vẽ từng biểu đồ rất mất thời gian, cộng đồng đã làm sẵn các mẫu siêu đẹp cho pfSense. Bạn chỉ cần "kéo" về là xong.

**1. Mẫu hiển thị Băng thông, CPU, RAM:**

* Ở menu bên trái, bấm vào biểu tượng dấu cộng `+` (hoặc **Dashboards**), chọn **Import**.
* Tại ô *Import via grafana.com*, bạn nhập ID: **`17540`** (hoặc `15264`), rồi bấm nút **Load** ở bên cạnh.
* Ở màn hình tiếp theo, kéo xuống dưới cùng, chỗ ô trống yêu cầu chọn InfluxDB, bạn xổ xuống và chọn cái nguồn InfluxDB vừa tạo ở Bước 1.
* Bấm **Import**.

**2. Mẫu hiển thị Log tường lửa (Chặn/Cho phép IP nào):**

* Lặp lại thao tác trên, vào **Import**.
* Lần này bạn nhập ID: **`13801`**, bấm **Load**.
* Ở ô chọn Data source, bạn chọn nguồn **Loki**.
* Bấm **Import**.

Sau khi bấm Import xong, Grafana sẽ tự động chuyển thẳng vào màn hình Dashboard với các biểu đồ cực kỳ trực quan. Bạn có thể chọn thời gian ở góc trên cùng bên phải (ví dụ: *Last 15 minutes*).



Bước 1: Cấu hình trên cả 2 con pfSense (Master và Backup)
Bạn cần làm bước này trên cả 2 giao diện web của pfSense Master (.10) và Backup (.11) để chúng bắt đầu đẩy dữ liệu về máy Ubuntu (172.16.1.21).

1. Cấu hình đẩy Băng thông, CPU, RAM (Telegraf):

Trên pfSense, vào System > Package Manager > Available Packages.

Tìm gói telegraf và bấm Install để cài đặt.

<img width="1438" height="555" alt="image" src="https://github.com/user-attachments/assets/03a2dee0-d920-44af-96a3-2526783d4ab2" />

Cài xong, vào Services > Telegraf.

Tích chọn Enable Telegraf.

Mục Telegraf Output: Chọn InfluxDB.

Mục InfluxDB Server: Điền chính xác http://172.16.1.241:8086

Mục InfluxDB Database: Điền chữ pfsense

Kéo xuống phần Default Telegraf Plugins: Tích chọn các ô cpu, memory, system, interfaces (đây chính là dữ liệu băng thông).

Kéo xuống dưới cùng bấm Save.

2. Cấu hình đẩy Nhật ký tường lửa (Remote Syslog):

Vào Status > System Logs, chọn tab Settings.

Kéo xuống cuối cùng tìm mục Remote Log Options.

Tích chọn Enable Remote Logging.

Mục Source Address: Chọn LAN.

Mục IP Protocol: Chọn IPv4.

Mục Remote Log Servers: Điền 172.16.1.21:1514

Mục Remote Syslog Contents: Tích chọn System Events và Firewall Events.

Bấm Save.
