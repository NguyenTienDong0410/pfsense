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
3. 1. Kiến Trúc Luồng Thu Thập Log
Để giải quyết triệt để lỗi phân tích cú pháp ký tự xuống dòng (Newline bug) của Promtail khi tiếp nhận trực tiếp luồng UDP Syslog từ tường lửa pfSense, hệ thống được tái cấu trúc với trạm trung chuyển Syslog-ng.

Luồng xử lý: Tường lửa pfSense xuất log UDP chuẩn RFC 5424 nhưng không có ký tự \n. Container Syslog-ng sẽ tiếp nhận luồng UDP thô ở cổng 1514, bổ sung ký tự \n vào cuối mỗi log để chuẩn hóa, sau đó luân chuyển qua giao thức TCP nội bộ cho Promtail. Promtail đọc log sạch, dán nhãn {job="pfsense-logs"} và ném sang cơ sở dữ liệu Loki.

2. Cấu hình file syslog-ng.conf
Tạo file cấu hình cho trạm trung chuyển để định nghĩa cổng tiếp nhận UDP và đích đến TCP.

@version: 4.0
@include "scl.conf"

# Nguồn hứng log UDP từ pfSense
source s_pfsense_udp {
    network(transport("udp") port(1514) flags(no-parse));
};

# Đích đẩy log qua TCP cho Promtail, chèn thêm dấu xuống dòng \n
destination d_promtail_tcp {
    network("promtail" port(1514) transport("tcp") template("${MSG}\n"));
};

log {
    source(s_pfsense_udp);
    destination(d_promtail_tcp);
};

3. Cấu hình file promtail-config.yml
Sửa lại cấu hình của Promtail để chuyển sang chế độ lắng nghe mạng TCP (nghe luồng nội bộ từ Syslog-ng chuyển sang).
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
  - job_name: pfsense
    syslog:
      listen_address: 0.0.0.0:1514
      listen_protocol: tcp
      idle_timeout: 60s
      label_structured_data: yes
      labels:
        job: "pfsense-logs"

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

<img width="1059" height="206" alt="image" src="https://github.com/user-attachments/assets/f8c37f60-fe0a-4db5-8734-e1afcc39c626" />

 *Lưu và thoát tương tự: `Ctrl + O` -> `Enter` -> `Ctrl + X`.*

---

### Bước 3: Khởi chạy cụm hệ thống

Vẫn đang đứng ở thư mục `monitoring`, bạn gõ lệnh cuối cùng này để tải và chạy tất cả:

```bash
sudo docker-compose up -d

```

<img width="1157" height="261" alt="image" src="https://github.com/user-attachments/assets/270e2611-242d-4148-9904-1c4982867631" />

Lúc này máy Ubuntu sẽ bắt đầu kéo (pull) các image của Grafana, InfluxDB, Loki,script về và tự động chạy. Quá trình này mất khoảng 1-2 phút tùy tốc độ mạng.

Sau khi chạy xong, để kiểm tra xem mọi thứ đã hoạt động chưa, bạn mở trình duyệt web trên máy tính thật (Windows) và gõ:
**`http://172.16.1.241:3000`**

<img width="1554" height="841" alt="image" src="https://github.com/user-attachments/assets/bcff96ba-824c-4838-bd8c-2e9ea483f2c8" />

Kiểm tra lại các service:

<img width="1489" height="572" alt="image" src="https://github.com/user-attachments/assets/c34716cc-7280-4cf4-a3a0-0add6b70290a" />


**Thêm nguồn và import biểu đồ**

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
* Tại ô *Import via grafana.com*, bạn nhập ID: **224140), rồi bấm nút **Load** ở bên cạnh.
* Ở màn hình tiếp theo, kéo xuống dưới cùng, chỗ ô trống yêu cầu chọn InfluxDB, bạn xổ xuống và chọn cái nguồn InfluxDB vừa tạo ở Bước 1.
* Bấm **Import**.

**2. Mẫu hiển thị Log tường lửa (Chặn/Cho phép IP nào):**

* Lặp lại thao tác trên, vào **Import**.
* Lần này bạn nhập ID: **`22722`**, bấm **Load**.
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
Sau đó vào phần dashboard import dash id 224410

<img width="1620" height="740" alt="image" src="https://github.com/user-attachments/assets/3155ffb1-5bb4-4c6e-8567-d0ec8c2e4495" />

<img width="1299" height="516" alt="image" src="https://github.com/user-attachments/assets/a56b6115-4df6-47bf-8c40-72c12022ce09" />

2. Cấu hình đẩy Nhật ký tường lửa (Remote Syslog):

Vào Status > System Logs, chọn tab Settings.

<img width="1530" height="770" alt="image" src="https://github.com/user-attachments/assets/3d8a19c6-f01e-4482-b8cb-b9d6b180893b" />

Kéo xuống cuối cùng tìm mục Remote Log Options.

Tích chọn Enable Remote Logging.

Mục Source Address: Chọn LAN.

Mục IP Protocol: Chọn IPv4.

Mục Remote Log Servers: Điền 172.16.1.21:1514

Mục Remote Syslog Contents: Tích chọn System Events và Firewall Events.

<img width="1423" height="727" alt="image" src="https://github.com/user-attachments/assets/1847f944-0a59-4ee0-a983-a692033031f5" />

Bấm Save.

1. Khai Báo Nguồn Dữ Liệu (Data Source)
Để Grafana có thể hiển thị được các luồng log do Promtail xử lý, cần thiết lập cầu nối HTTP nội bộ giữa Grafana và Loki.

Truy cập giao diện Web của Grafana tại địa chỉ http://172.16.1.241:3000.

Điều hướng đến menu Connections ➔ Data sources ➔ Chọn Add data source.

Lựa chọn loại cơ sở dữ liệu là Loki.

Tại mục HTTP ➔ URL, khai báo đường dẫn phân giải nội bộ của container trong mạng lưới Docker: http://loki:3100.

Nhấn Save & test. Hệ thống phản hồi "Data source successfully connected" xác nhận kết nối thành công.
<img width="1142" height="736" alt="image" src="https://github.com/user-attachments/assets/70b4ead8-ff1d-431f-83f5-db7b3623e7e7" />

2. Chuẩn Hóa Cấu Trúc File JSON Dashboard (Xử lý lỗi No Data)
Phân tích sự cố: Hầu hết các mẫu Dashboard chia sẻ trên cộng đồng Grafana được định nghĩa cứng (hardcode) với nhãn tìm kiếm mặc định là {hostname=~"$hostname", syslog_app="filterlog"}. Tuy nhiên, trong kiến trúc tối ưu đã triển khai, Promtail được cấu hình dán nhãn định danh ngắn gọn là {job="pfsense-logs"}. Sự sai lệch nhãn (Label) này khiến các biểu đồ không thể móc nối được dữ liệu và báo lỗi "No data".

Quy trình xử lý tự động hóa: Thay vì chỉnh sửa thủ công từng panel trên giao diện đồ họa, phương pháp tối ưu là can thiệp trực tiếp vào mã nguồn file JSON của Dashboard trước khi đưa lên hệ thống:

Mở file template định dạng .json bằng trình soạn thảo mã nguồn (VS Code hoặc Notepad).

Sử dụng tính năng tìm kiếm và thay thế hàng loạt (Find & Replace - Ctrl + H).

Tìm kiếm chuỗi gốc: hostname=~\"$hostname\", syslog_app=\"filterlog\"

Thay thế thành chuỗi chuẩn: job=\"pfsense-logs\"

Thực hiện thêm một lần dò tìm phụ cho các biến hệ thống (Variables): Thay thế {syslog_app=\"filterlog\"} thành {job=\"pfsense-logs\"}.

Lưu lại file JSON sau khi đã chuẩn hóa thành công.

3. Triển Khai Dashboard & Nghiệm Thu
Tại giao diện Grafana, điều hướng đến Dashboards ➔ Chọn New ➔ Import. sử dụng ip 22722

<img width="848" height="739" alt="image" src="https://github.com/user-attachments/assets/92a51d7a-2434-4249-899a-77141cb64cc7" />


Trong bảng tùy chọn kết nối hiện ra, chọn đích đến là kho dữ liệu Loki vừa thiết lập ở Bước 1. Nhấn Import.

Kết quả nghiệm thu: Ngay sau khi Import, toàn bộ các biểu đồ phân tích cảnh báo mạng (Top Source IPs, Blocked Ports, Action Timeline) sẽ lập tức phản hồi và vẽ sơ đồ luồng dữ liệu theo thời gian thực (Real-time). Lỗi lệch múi giờ hay mất định dạng Syslog đều đã được xử lý triệt để ở phân hệ máy chủ, mang lại một bảng điều khiển giám sát hoàn thiện và chính xác tuyệt đối.


<img width="1621" height="868" alt="image" src="https://github.com/user-attachments/assets/96a61638-9a85-46df-93ec-3eddd395445b" />

<img width="1560" height="718" alt="image" src="https://github.com/user-attachments/assets/18e7b7ed-4f93-4550-b077-d477fced9f02" />


