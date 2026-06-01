Nội dung
2.1 Giới thiệu về Captive Portal
Trong pfSense có một tính khá là hữu ích đó là Captive Portal. Đây là một kỹ thuật buộc người dùng phải chứng thực qua 1 giao diện web trước khi kết nối vào internet. Kỹ thuât này thường áp dụng cho các điểm truy cập wifi, mạng có dây. Người dùng muốn truy cập vào, phải có một account chứng thực, ...
2.2 Tính năng, ứng dụng của Captive Portal
Trong thực tế, Captive Portal được biết đến với cái tên đầy hoa mĩ Wi-Fi Marketing bởi vì nó được ứng dụng khá nhiều trong vai trò marketing. Ngày nay, Wi-Fi Marketing được cung cấp và sử dụng trong nhiều doanh nghiệp từ lớn tới nhỏ. WiFi Marketing là một trong những cách sáng tạo và hiệu quả nhất để quảng cáo thương hiệu. Bằng cách truyền đạt thông điệp hoặc nội dung trực tiếp tới khách hàng tiềm năng hoặc người dùng gần điểm phát.

Như đã được đề cập ở nội dung phần giới thiệu về Captive Portal. Qua việc tạo ra một vùng phủ sóng dựa trên công nghệ không dây WiFi cho phép bất cứ ai có thiết bị di động (laptop, điện thoại smartphone, máy tính bảng Tablet…) Được trang bị công nghệ WiFi để kết nối và truy cập các dịch vụ hoặc nội dung đã được được cung cấp sẵn.

Hệ thống trong giải pháp WiFi Marketing tạo điều kiện cho khách hàng truy cập miễn phí. Nhưng trước tiên họ phải điều hướng request tới nội dung của một trang website mà chúng ta đã chuẩn bị sẵn. Ở đây chúng ta sẽ có cơ hội hiển thị cho họ sản phẩm, dịch vụ của ta đã và đang cung cấp, khuyến mãi hoặc đơn giản là thông tin liên quan đến doanh nghiệp của mình.

Những lợi ích của giải pháp Captive Portal đem lại khi được ứng dụng vào thực tế:

Xây dựng thương hiệu:

Các tin tức, thông tin, hình ảnh về doanh nghiệp sẽ nằm dưới quyền kiểm soát của mình bằng việc tùy chỉnh trang website mà người buộc phải truy cập tới khi muốn sử dụng internet.
Tiếp cận quảng cáo của khách hàng:

Quảng cáo sẽ được tiếp cận tới người sử dụng khi họ truy cập và sử dụng Wi-Fi
Là giải pháp đa giải pháp:

Được thể hiện qua việc cung cấp website được hỗ trợ bởi nhiều thiết bị Table, Smartphone và Laptop, PC.
Giới hạn băng thông người sử dụng:

Để tránh các vấn đề có thể bị lạm dụng. Ta có thể kiểm soát lưu lượng sử dụng internet của người dùng. Nhờ vậy mà băng thông sẽ được chia sẻ một cách hợp lý tới người sử dụng.
Quản lý phiên hoạt động:

Thời gian kết nối của một người sử dụng có thể điều chỉnh bất cứ lúc nào. Có thể thiết lập thời gian sử dụng khác nhau cho mọi người. Ta luôn có sự lựa chọn để thay đổi các thiết lập.
Cổng thông tin điện tử:

Ta có thể lợi dụng website mà Captive Portal sử dụng để xác thực để tạo lên một trang tin tức nội bộ.

### Giai đoạn 1: Chuẩn bị User/Nhóm (Authentication)

Trường hợp này ta sẽ dùng Local Database (tài khoản tạo trực tiếp trên pfSense) để xác thực. Thực hiện các bước này trên con **pfSense Master**:

1. Truy cập vào **System** > **User Manager** > **Users**.
2. Nhấn **Add** để tạo người dùng mới.
* **Username:** (ví dụ: `guest01`)
* **Password:** (nhập mật khẩu)
* **Group:** Bạn có thể tạo một group riêng cho Captive Portal hoặc để trống.

<img width="1433" height="724" alt="image" src="https://github.com/user-attachments/assets/8dd676f1-433b-41bb-8205-b6dcff0414f5" />

3. Nhấn **Save**.

 sau đó add thêm quyền vào user
<img width="1515" height="690" alt="image" src="https://github.com/user-attachments/assets/7c20d739-c4f7-4f06-84a7-2c666fb6e0d1" />

### Giai đoạn 2: Cấu hình Captive Portal

Thực hiện trên con **pfSense Master**:

1. Truy cập **Services** > **Captive Portal**.
2. Nhấn **Add** để tạo một Zone mới.
* **Zone Name:** `LAN_Guest` (hoặc tên tùy ý)
* **Zone Description:** Captive Portal cho mạng LAN vlan2
<img width="1474" height="684" alt="image" src="https://github.com/user-attachments/assets/693d0157-b92a-47b7-9eac-7f5007d672e1" />


3. Nhấn **Save & Continue**.
4. Cấu hình các thông số chính trong tab **Configuration**:
* **Enable:** Tích chọn **Enable captive portal**.
* **Interfaces:** Chọn cổng mạng áp dụng, trong mô hình của bạn là cổng **LAN (vlan2)**.
* **Idle timeout (Minutes):** Thời gian ngắt kết nối nếu người dùng không có traffic (ví dụ: 60 phút).
* **Hard timeout (Minutes):** Thời gian tối đa một phiên kết nối dù có dùng hay không (ví dụ: 1440 phút = 1 ngày).
* **Authentication Method:** Chọn **Local Database** (để dùng user đã tạo ở Giai đoạn 1).
* **Concurrent user logins:** Tích chọn nếu không muốn một user đăng nhập trên nhiều thiết bị cùng lúc.

<img width="1334" height="670" alt="image" src="https://github.com/user-attachments/assets/2248c904-2db5-480d-8f34-2a3e98fe09d4" />

5. Kéo xuống dưới cùng và nhấn **Save**.

Có thể tạo 1 web login hoặc landing page riêng cho tùy nhu cầu sử dụng

<img width="1344" height="548" alt="image" src="https://github.com/user-attachments/assets/312fd9cb-a3e1-4ae8-94f2-c790aa043ac9" />

7. Ngoài ra còn cấu hình theo IP, host, mac của thiết bị để cho phép truy cập

<img width="1422" height="650" alt="image" src="https://github.com/user-attachments/assets/deb698c6-646d-46ec-b679-4b82c23c9529" />

<img width="1423" height="664" alt="image" src="https://github.com/user-attachments/assets/ba76cac1-865e-421d-8325-783a9db2b09c" />

<img width="1488" height="584" alt="image" src="https://github.com/user-attachments/assets/6dfad22e-2832-42fb-80f0-539174737be9" />

### Giai đoạn 3: Cấu hình High Availability (HA) cho Captive Portal

Đây là bước cực kỳ quan trọng đối với mô hình Master/Backup (CARP) của bạn. Nếu không làm bước này, khi Master sập, Backup lên thay, toàn bộ client sẽ bị ngắt mạng và phải login lại Captive Portal.

1. Vẫn trong phần cấu hình Captive Portal (Services > Captive Portal > Zone `LAN_Guest`), chuyển sang tab **High Availability**.
2. Cấu hình các thông số sau:
* **Enable XMLRPC Sync:** Tích chọn để đồng bộ cấu hình Captive Portal sang con Backup.
* **Sync IP address:** Nhập IP thực (Interface IP, không phải VIP) của con **pfSense Backup** qua cổng SYNC hoặc cổng LAN (tùy theo cách bạn đang thiết lập HA, thông thường là IP cổng sync của node dự phòng).
* **Sync port:** Thường để mặc định (hoặc theo cấu hình port webGUI của Backup).
* **Sync username:** `admin` (hoặc user có quyền admin).
* **Sync password:** Mật khẩu của user admin trên node Backup.

<img width="1458" height="768" alt="image" src="https://github.com/user-attachments/assets/00c18545-02ec-4058-b5a6-2e366c488976" />


3. Nhấn **Save**.
*(Lưu ý: Bạn cũng cần đảm bảo trong **System > High Avail. Sync** đã tích chọn đồng bộ Captive Portal).*

### Giai đoạn 4: Cấu hình DHCP & DNS (Bắt buộc)

Captive Portal hoạt động dựa vào việc chặn truy cập web (HTTP) và chuyển hướng. Để chuyển hướng được, client **phải phân giải được DNS**.

1. Đảm bảo cổng LAN (vlan2) trên pfSense Master đang chạy **DHCP Server** cấp dải IP `172.16.1.0/24`.
2. Trong cấu hình DHCP Server, **Gateway** phải trỏ về IP VIP CARP: `172.16.1.12`.
3. **DNS Server** nên được trỏ về IP VIP CARP: `172.16.1.12` (Và đảm bảo pfSense đang bật DNS Resolver hoặc DNS Forwarder).

<img width="1458" height="768" alt="image" src="https://github.com/user-attachments/assets/2d88feee-b6bf-4b1c-9715-3afaf70c07c7" />

### Giai đoạn 5: Kiểm tra hoạt động

1. Lấy một máy **Client** (ví dụ IP đang là `172.16.1.21`).
2. Mở trình duyệt web (khuyên dùng trình duyệt ẩn danh để tránh cache).
3. Truy cập vào một trang web **HTTP** bất kỳ (ví dụ: `http://neverssl.com` hoặc `http://1.1.1.1`). *(Lưu ý: Truy cập trang HTTPS như google.com có thể bị trình duyệt cảnh báo bảo mật trước khi kịp chuyển hướng).*
4. Trình duyệt sẽ tự động Redirect về trang đăng nhập của pfSense (Captive Portal page).
5. Nhập tài khoản `guest01` đã tạo ở Bước 1.
6. Sau khi đăng nhập thành công, bạn sẽ truy cập được Internet bình thường ra đường WAN.
<img width="1583" height="761" alt="image" src="https://github.com/user-attachments/assets/5f99bd7e-fa2e-4d61-8d1d-acd0fb9c3ccd" />


<img width="1604" height="749" alt="image" src="https://github.com/user-attachments/assets/2e82d00d-94ac-43a6-8e6f-53af051ab0c7" />

---

**Một số lưu ý khắc phục lỗi (Troubleshooting) nhanh:**

* Nếu máy Client không hiện trang đăng nhập: Hãy kiểm tra lại xem máy Client đã nhận đúng Gateway là IP VIP `172.16.1.12` và đã phân giải được DNS chưa. Thử ping `8.8.8.8` (sẽ bị rớt nếu CP hoạt động) và ping tên miền (phải ra được IP nhưng request bị chặn).
* Nếu bạn muốn tùy biến giao diện trang đăng nhập (thêm logo, chữ tiếng Việt), bạn có thể chỉnh sửa trong tab **Captive Portal > HTML Page Contents**.
