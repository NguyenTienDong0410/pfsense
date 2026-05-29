

### Giai đoạn 1: Chuẩn bị User/Nhóm (Authentication)

Trường hợp này ta sẽ dùng Local Database (tài khoản tạo trực tiếp trên pfSense) để xác thực. Thực hiện các bước này trên con **pfSense Master**:

1. Truy cập vào **System** > **User Manager** > **Users**.
2. Nhấn **Add** để tạo người dùng mới.
* **Username:** (ví dụ: `guest01`)
* **Password:** (nhập mật khẩu)
* **Group:** Bạn có thể tạo một group riêng cho Captive Portal hoặc để trống.


3. Nhấn **Save**.

### Giai đoạn 2: Cấu hình Captive Portal

Thực hiện trên con **pfSense Master**:

1. Truy cập **Services** > **Captive Portal**.
2. Nhấn **Add** để tạo một Zone mới.
* **Zone Name:** `LAN_Guest` (hoặc tên tùy ý)
* **Zone Description:** Captive Portal cho mạng LAN vlan2


3. Nhấn **Save & Continue**.
4. Cấu hình các thông số chính trong tab **Configuration**:
* **Enable:** Tích chọn **Enable captive portal**.
* **Interfaces:** Chọn cổng mạng áp dụng, trong mô hình của bạn là cổng **LAN (vlan2)**.
* **Idle timeout (Minutes):** Thời gian ngắt kết nối nếu người dùng không có traffic (ví dụ: 60 phút).
* **Hard timeout (Minutes):** Thời gian tối đa một phiên kết nối dù có dùng hay không (ví dụ: 1440 phút = 1 ngày).
* **Authentication Method:** Chọn **Local Database** (để dùng user đã tạo ở Giai đoạn 1).
* **Concurrent user logins:** Tích chọn nếu không muốn một user đăng nhập trên nhiều thiết bị cùng lúc.


5. Kéo xuống dưới cùng và nhấn **Save**.

### Giai đoạn 3: Cấu hình High Availability (HA) cho Captive Portal

Đây là bước cực kỳ quan trọng đối với mô hình Master/Backup (CARP) của bạn. Nếu không làm bước này, khi Master sập, Backup lên thay, toàn bộ client sẽ bị ngắt mạng và phải login lại Captive Portal.

1. Vẫn trong phần cấu hình Captive Portal (Services > Captive Portal > Zone `LAN_Guest`), chuyển sang tab **High Availability**.
2. Cấu hình các thông số sau:
* **Enable XMLRPC Sync:** Tích chọn để đồng bộ cấu hình Captive Portal sang con Backup.
* **Sync IP address:** Nhập IP thực (Interface IP, không phải VIP) của con **pfSense Backup** qua cổng SYNC hoặc cổng LAN (tùy theo cách bạn đang thiết lập HA, thông thường là IP cổng sync của node dự phòng).
* **Sync port:** Thường để mặc định (hoặc theo cấu hình port webGUI của Backup).
* **Sync username:** `admin` (hoặc user có quyền admin).
* **Sync password:** Mật khẩu của user admin trên node Backup.


3. Nhấn **Save**.
*(Lưu ý: Bạn cũng cần đảm bảo trong **System > High Avail. Sync** đã tích chọn đồng bộ Captive Portal).*

### Giai đoạn 4: Cấu hình DHCP & DNS (Bắt buộc)

Captive Portal hoạt động dựa vào việc chặn truy cập web (HTTP) và chuyển hướng. Để chuyển hướng được, client **phải phân giải được DNS**.

1. Đảm bảo cổng LAN (vlan2) trên pfSense Master đang chạy **DHCP Server** cấp dải IP `172.16.1.0/24`.
2. Trong cấu hình DHCP Server, **Gateway** phải trỏ về IP VIP CARP: `172.16.1.12`.
3. **DNS Server** nên được trỏ về IP VIP CARP: `172.16.1.12` (Và đảm bảo pfSense đang bật DNS Resolver hoặc DNS Forwarder).

### Giai đoạn 5: Kiểm tra hoạt động

1. Lấy một máy **Client** (ví dụ IP đang là `172.16.1.21`).
2. Mở trình duyệt web (khuyên dùng trình duyệt ẩn danh để tránh cache).
3. Truy cập vào một trang web **HTTP** bất kỳ (ví dụ: `http://neverssl.com` hoặc `http://1.1.1.1`). *(Lưu ý: Truy cập trang HTTPS như google.com có thể bị trình duyệt cảnh báo bảo mật trước khi kịp chuyển hướng).*
4. Trình duyệt sẽ tự động Redirect về trang đăng nhập của pfSense (Captive Portal page).
5. Nhập tài khoản `guest01` đã tạo ở Bước 1.
6. Sau khi đăng nhập thành công, bạn sẽ truy cập được Internet bình thường ra đường WAN.

---

**Một số lưu ý khắc phục lỗi (Troubleshooting) nhanh:**

* Nếu máy Client không hiện trang đăng nhập: Hãy kiểm tra lại xem máy Client đã nhận đúng Gateway là IP VIP `172.16.1.12` và đã phân giải được DNS chưa. Thử ping `8.8.8.8` (sẽ bị rớt nếu CP hoạt động) và ping tên miền (phải ra được IP nhưng request bị chặn).
* Nếu bạn muốn tùy biến giao diện trang đăng nhập (thêm logo, chữ tiếng Việt), bạn có thể chỉnh sửa trong tab **Captive Portal > HTML Page Contents**.
