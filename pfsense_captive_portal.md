# Captive Portal trên pfSense

---

## Mục lục

- [2.1 Giới thiệu về Captive Portal](#21-giới-thiệu-về-captive-portal)
- [2.2 Tính năng và ứng dụng của Captive Portal](#22-tính-năng-và-ứng-dụng-của-captive-portal)
- [2.3 Triển khai Captive Portal trên pfSense](#23-triển-khai-captive-portal-trên-pfsense)
  - [Giai đoạn 1: Tạo tài khoản người dùng (Authentication)](#giai-đoạn-1-tạo-tài-khoản-người-dùng-authentication)
  - [Giai đoạn 2: Cấu hình Captive Portal Zone](#giai-đoạn-2-cấu-hình-captive-portal-zone)
  - [Giai đoạn 3: Cấu hình High Availability cho Captive Portal](#giai-đoạn-3-cấu-hình-high-availability-cho-captive-portal)
  - [Giai đoạn 4: Cấu hình DHCP và DNS](#giai-đoạn-4-cấu-hình-dhcp-và-dns)
  - [Giai đoạn 5: Kiểm tra hoạt động](#giai-đoạn-5-kiểm-tra-hoạt-động)
- [2.4 Xử lý sự cố (Troubleshooting)](#24-xử-lý-sự-cố-troubleshooting)

---

## 2.1 Giới thiệu về Captive Portal

**Captive Portal** là một tính năng quan trọng được tích hợp sẵn trong pfSense, cho phép quản trị viên mạng buộc người dùng phải xác thực qua một giao diện web trước khi được cấp quyền truy cập Internet. Kỹ thuật này hoạt động bằng cách chặn tất cả lưu lượng HTTP/HTTPS từ client và chuyển hướng trình duyệt đến một trang đăng nhập do quản trị viên kiểm soát.

Captive Portal thường được triển khai trong các môi trường như:

- Điểm truy cập Wi-Fi công cộng (quán cà phê, khách sạn, trung tâm thương mại)
- Mạng có dây trong doanh nghiệp hoặc trường học
- Mạng nội bộ yêu cầu kiểm soát người dùng

Để truy cập Internet, người dùng bắt buộc phải có tài khoản xác thực hợp lệ được cấp bởi quản trị viên, hoặc thực hiện một hành động cụ thể (ví dụ: đồng ý điều khoản sử dụng).

---

## 2.2 Tính năng và ứng dụng của Captive Portal

Trong thực tế, Captive Portal còn được biết đến với tên gọi **Wi-Fi Marketing**, bởi vì nó được ứng dụng rộng rãi như một công cụ tiếp thị hiệu quả. Ngày nay, Wi-Fi Marketing được triển khai trong nhiều doanh nghiệp từ quy mô nhỏ đến lớn, nhờ khả năng truyền tải thông điệp quảng cáo trực tiếp đến khách hàng tại điểm phát sóng.

Thông qua việc tạo ra một vùng phủ sóng Wi-Fi, Captive Portal cho phép bất kỳ thiết bị nào có hỗ trợ Wi-Fi (laptop, điện thoại thông minh, máy tính bảng...) kết nối và sử dụng dịch vụ. Tuy nhiên, trước khi được phép ra Internet, người dùng sẽ bị điều hướng đến một trang web mà doanh nghiệp kiểm soát — đây là cơ hội để hiển thị thông tin sản phẩm, dịch vụ, hoặc các chương trình khuyến mãi.

Các lợi ích chính mà Captive Portal mang lại bao gồm:

**Xây dựng thương hiệu**
Doanh nghiệp có toàn quyền tùy chỉnh trang đăng nhập với nội dung, hình ảnh, logo theo nhận diện thương hiệu riêng. Đây là điểm chạm đầu tiên giữa khách hàng và thương hiệu ngay khi họ kết nối mạng.

**Tiếp cận quảng cáo trực tiếp**
Quảng cáo được hiển thị bắt buộc đến người dùng tại thời điểm họ kết nối Wi-Fi, giúp tăng tỉ lệ tiếp cận và nhận diện thương hiệu.

**Hỗ trợ đa nền tảng**
Trang đăng nhập Captive Portal hoạt động trên mọi thiết bị và hệ điều hành có trình duyệt web, bao gồm tablet, smartphone và máy tính.

**Giới hạn băng thông người dùng**
Quản trị viên có thể thiết lập giới hạn tốc độ (bandwidth throttling) cho từng phiên người dùng, tránh tình trạng một người chiếm dụng quá nhiều băng thông, đảm bảo tài nguyên mạng được chia sẻ hợp lý.

**Quản lý phiên kết nối (Session Management)**
Cho phép thiết lập thời gian tối đa của một phiên làm việc (Hard Timeout) và thời gian ngắt kết nối khi không có lưu lượng (Idle Timeout). Quản trị viên có thể điều chỉnh các thông số này bất kỳ lúc nào mà không cần khởi động lại dịch vụ.

**Cổng thông tin nội bộ**
Trang Captive Portal có thể được tùy biến thành cổng thông tin nội bộ (intranet portal), hiển thị tin tức, thông báo hoặc các liên kết nội bộ trước khi người dùng ra Internet.

---

## 2.3 Triển khai Captive Portal trên pfSense

Mô hình triển khai trong tài liệu này sử dụng cấu hình **pfSense Master/Backup (CARP High Availability)**, với Captive Portal áp dụng trên cổng **LAN (VLAN 2)**, dải địa chỉ `172.16.1.0/24`. IP VIP CARP của cổng LAN là `172.16.1.12`.

---

### Giai đoạn 1: Tạo tài khoản người dùng (Authentication)

Trong giai đoạn này, hệ thống sử dụng **Local Database** — tức là tài khoản người dùng được tạo và lưu trữ trực tiếp trên pfSense — để xác thực Captive Portal. Đây là phương thức xác thực đơn giản, phù hợp cho môi trường nhỏ hoặc mục đích thử nghiệm. Trong môi trường doanh nghiệp lớn hơn, có thể thay thế bằng RADIUS hoặc LDAP.

Thực hiện các bước sau trên **pfSense Master**:

1. Truy cập **System > User Manager > Users**.
2. Nhấn **Add** để tạo người dùng mới với các thông tin:
   - **Username:** `guest01` (hoặc tùy chọn)
   - **Password:** Nhập mật khẩu mong muốn
   - **Group:** Có thể tạo một group riêng dành cho Captive Portal để dễ quản lý quyền, hoặc để trống nếu không cần phân nhóm.
3. Nhấn **Save** để lưu.

![Tạo người dùng mới trong User Manager](https://github.com/user-attachments/assets/8dd676f1-433b-41bb-8205-b6dcff0414f5)

4. Sau khi tạo xong, quay lại chỉnh sửa user và bổ sung quyền cần thiết (ví dụ: `User - Services: Captive Portal login`) để tài khoản có thể xác thực qua Captive Portal.

![Thêm quyền cho người dùng](https://github.com/user-attachments/assets/7c20d739-c4f7-4f06-84a7-2c666fb6e0d1)

---

### Giai đoạn 2: Cấu hình Captive Portal Zone

Mỗi **Zone** trong Captive Portal đại diện cho một vùng mạng được quản lý độc lập, gắn với một hoặc nhiều interface cụ thể. Trong mô hình này, zone được áp dụng cho cổng **LAN (VLAN 2)**.

Thực hiện trên **pfSense Master**:

1. Truy cập **Services > Captive Portal**.
2. Nhấn **Add** để tạo Zone mới:
   - **Zone Name:** `LAN_Guest`
   - **Zone Description:** `Captive Portal cho mạng LAN VLAN2`

![Tạo Captive Portal Zone](https://github.com/user-attachments/assets/693d0157-b92a-47b7-9eac-7f5007d672e1)

3. Nhấn **Save & Continue** để chuyển sang trang cấu hình chi tiết.

4. Trong tab **Configuration**, thiết lập các thông số sau:
   - **Enable captive portal:** Tích chọn để kích hoạt.
   - **Interfaces:** Chọn **LAN (VLAN2)** — interface mà Captive Portal sẽ kiểm soát.
   - **Idle Timeout (Minutes):** Thời gian ngắt kết nối tự động khi người dùng không có lưu lượng, ví dụ `60` phút.
   - **Hard Timeout (Minutes):** Thời gian tối đa của một phiên, bất kể người dùng có đang sử dụng hay không, ví dụ `1440` phút (tương đương 1 ngày).
   - **Authentication Method:** Chọn **Local Database** để dùng tài khoản đã tạo ở Giai đoạn 1.
   - **Concurrent User Logins:** Tích chọn nếu muốn ngăn một tài khoản đăng nhập đồng thời trên nhiều thiết bị.

![Cấu hình các thông số Captive Portal](https://github.com/user-attachments/assets/2248c904-2db5-480d-8f34-2a3e98fe09d4)

5. Cuộn xuống cuối trang và nhấn **Save**.

6. (Tùy chọn) Tùy biến giao diện trang đăng nhập: Trong tab **HTML Page Contents**, có thể upload trang HTML tùy chỉnh (thêm logo, banner, nội dung tiếng Việt...) thay cho trang mặc định của pfSense.

![Tùy chỉnh giao diện trang đăng nhập](https://github.com/user-attachments/assets/312fd9cb-a3e1-4ae8-94f2-c790aa043ac9)

7. Ngoài ra, Captive Portal còn hỗ trợ cấu hình **danh sách trắng (Allowed)** theo IP, hostname hoặc địa chỉ MAC, cho phép một số thiết bị bỏ qua xác thực (ví dụ: máy in, camera, thiết bị IoT nội bộ).

![Cho phép theo địa chỉ IP](https://github.com/user-attachments/assets/deb698c6-646d-46ec-b679-4b82c23c9529)

![Cho phép theo hostname](https://github.com/user-attachments/assets/ba76cac1-865e-421d-8325-783a9db2b09c)

![Cho phép theo địa chỉ MAC](https://github.com/user-attachments/assets/6dfad22e-2832-42fb-80f0-539174737be9)

---

### Giai đoạn 3: Cấu hình High Availability cho Captive Portal

Đây là bước **bắt buộc** trong mô hình Master/Backup (CARP). Nếu bỏ qua bước này, khi node Master gặp sự cố và node Backup lên thay, toàn bộ client sẽ mất kết nối và buộc phải đăng nhập lại Captive Portal từ đầu — gây gián đoạn dịch vụ không cần thiết.

Cơ chế hoạt động: pfSense sử dụng **XMLRPC Sync** để đồng bộ cấu hình Captive Portal (bao gồm cả danh sách phiên đăng nhập đang hoạt động) từ Master sang Backup theo thời gian thực.

Thực hiện trên **pfSense Master**:

1. Truy cập **Services > Captive Portal > Zone `LAN_Guest`**.
2. Chuyển sang tab **High Availability** và cấu hình:
   - **Enable XMLRPC Sync:** Tích chọn để bật đồng bộ cấu hình sang node Backup.
   - **Sync IP Address:** Nhập **IP thực** (không phải VIP) của cổng SYNC trên node **pfSense Backup** (ví dụ: IP của interface pfsync/LAN trên Backup).
   - **Sync Port:** Giữ mặc định hoặc điền đúng port WebGUI của node Backup (thường là `443`).
   - **Sync Username:** `admin` (hoặc tài khoản có quyền admin trên Backup).
   - **Sync Password:** Mật khẩu tương ứng của tài khoản admin trên node Backup.

![Cấu hình High Availability Sync cho Captive Portal](https://github.com/user-attachments/assets/00c18545-02ec-4058-b5a6-2e366c488976)

3. Nhấn **Save**.

> **Lưu ý quan trọng:** Bên cạnh cấu hình tại đây, cần đảm bảo trong **System > High Avail. Sync** (trên node Master) đã tích chọn mục đồng bộ **Captive Portal**. Nếu không, cấu hình zone có thể không được đẩy sang Backup đầy đủ.

---

### Giai đoạn 4: Cấu hình DHCP và DNS

Captive Portal hoạt động bằng cách chặn lưu lượng HTTP từ client và chuyển hướng đến trang đăng nhập. Để cơ chế chuyển hướng này hoạt động chính xác, client **bắt buộc phải phân giải được DNS** và nhận đúng gateway.

Thực hiện kiểm tra và cấu hình các mục sau:

1. Đảm bảo **DHCP Server** trên cổng LAN (VLAN2) đang hoạt động và cấp dải địa chỉ `172.16.1.0/24`.
2. Trong cấu hình DHCP Server:
   - **Gateway:** Phải trỏ về **IP VIP CARP** của cổng LAN: `172.16.1.12` — không được trỏ về IP thực của Master hay Backup.
   - **DNS Server:** Nên trỏ về **IP VIP CARP**: `172.16.1.12`, đồng thời đảm bảo pfSense đang bật **DNS Resolver** hoặc **DNS Forwarder** để phục vụ truy vấn DNS từ client.

> **Lý do dùng VIP CARP thay vì IP thực:** Khi failover xảy ra, IP VIP sẽ tự động chuyển sang node Backup. Nếu client dùng IP thực của Master, kết nối sẽ gián đoạn khi Master sập. Dùng VIP đảm bảo tính liên tục của dịch vụ.

![Cấu hình DHCP Server với Gateway và DNS trỏ về VIP CARP](https://github.com/user-attachments/assets/2d88feee-b6bf-4b1c-9715-3afaf70c07c7)

---

### Giai đoạn 5: Kiểm tra hoạt động

Sau khi hoàn tất cấu hình, tiến hành kiểm tra trên máy client:

1. Chuẩn bị một máy **Client** được kết nối vào LAN VLAN2 (ví dụ: IP client là `172.16.1.21`, gateway là `172.16.1.12`).
2. Mở trình duyệt web ở **chế độ ẩn danh (Incognito/Private)** để tránh ảnh hưởng từ cache hoặc session cũ.
3. Truy cập vào một trang web **HTTP** (không phải HTTPS), ví dụ: `http://neverssl.com` hoặc `http://1.1.1.1`.

> **Lý do dùng HTTP thay vì HTTPS:** Captive Portal chặn và chuyển hướng lưu lượng HTTP (cổng 80). Với HTTPS (cổng 443), trình duyệt có thể hiển thị cảnh báo bảo mật (SSL certificate mismatch) trước khi kịp chuyển hướng, gây nhầm lẫn cho người dùng.

4. Nếu Captive Portal hoạt động đúng, trình duyệt sẽ tự động **redirect** về trang đăng nhập của pfSense.
5. Nhập tài khoản `guest01` và mật khẩu đã tạo ở Giai đoạn 1.
6. Sau khi đăng nhập thành công, client có thể truy cập Internet bình thường qua đường WAN.

![Trang đăng nhập Captive Portal hiển thị trên client](https://github.com/user-attachments/assets/5f99bd7e-fa2e-4d61-8d1d-acd0fb9c3ccd)

![Trạng thái sau khi đăng nhập thành công](https://github.com/user-attachments/assets/2e82d00d-94ac-43a6-8e6f-53af051ab0c7)

---

## 2.4 Xử lý sự cố (Troubleshooting)

Dưới đây là các vấn đề thường gặp và hướng xử lý:

**Client không hiển thị trang đăng nhập Captive Portal**

Kiểm tra theo thứ tự:
- Xác nhận client đã nhận đúng Gateway là IP VIP `172.16.1.12` (kiểm tra bằng `ipconfig` hoặc `ip route`).
- Kiểm tra DNS hoạt động: Thử phân giải tên miền bất kỳ (phải trả về IP nhưng kết nối web sẽ bị chặn cho đến khi đăng nhập).
- Thử ping `8.8.8.8` — nếu Captive Portal đang hoạt động, ping sẽ bị chặn (timeout).
- Đảm bảo đang truy cập trang **HTTP**, không phải HTTPS.
- Kiểm tra lại xem interface **LAN (VLAN2)** có được chọn đúng trong cấu hình Zone hay không.

**Giao diện trang đăng nhập muốn tùy biến (thêm logo, tiếng Việt)**

Truy cập **Services > Captive Portal > Zone `LAN_Guest` > tab HTML Page Contents** để upload file HTML tùy chỉnh.

**Sau khi failover (Master sập, Backup lên), client bị ngắt kết nối và phải đăng nhập lại**

Kiểm tra lại Giai đoạn 3: Đảm bảo XMLRPC Sync đã được bật và trong **System > High Avail. Sync** có tích chọn đồng bộ Captive Portal. Sau khi bật, nhấn **Force Sync** để đồng bộ ngay lập tức thay vì chờ chu kỳ tự động.
