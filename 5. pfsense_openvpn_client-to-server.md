
---

## CÀI ĐẶT VÀ SỬ DỤNG PFSENSE KẾT HỢP VỚI OPENVPN

> **Ghi chú:** Phần này trình bày lại toàn bộ quy trình cài đặt từ đầu và bổ sung các nội dung nghiên cứu tiếp theo, kế thừa từ phần cài đặt  trong báo cáo ngày 16/04.

---

## 1. Đặc điểm của hai chế độ TUN và TAP

Trước khi đi vào thực hành, cần hiểu rõ sự khác biệt giữa hai chế độ hoạt động của OpenVPN là **TUN** và **TAP**, vì mỗi chế độ phù hợp với một kịch bản triển khai khác nhau.

| Tiêu chí | TUN (Tunnel) | TAP (Bridge) |
|---|---|---|
| **Lớp hoạt động** | Layer 3 (IP Routing) | Layer 2 (Ethernet Bridging) |
| **Loại lưu lượng** | Chỉ IP (IPv4/IPv6) | Mọi loại Ethernet (IP, ARP, broadcast…) |
| **Cấp phát địa chỉ IP** | Dải IP riêng biệt cho VPN | Cùng dải IP với mạng LAN nội bộ |
| **Hiệu năng** | Cao hơn, ít overhead | Thấp hơn do xử lý broadcast |
| **Trường hợp dùng** | Kết nối Remote Access thông thường, Site-to-Site | Cần máy khách ở cùng segment LAN (dùng chung DHCP, NetBIOS, …) |
| **Cấu hình Firewall** | Đơn giản hơn | Phức tạp hơn, cần tạo Bridge Interface |


---

## 2. Triển khai chế độ VPN TUN

Chế độ TUN hoạt động ở tầng mạng (Layer 3), tạo một đường hầm IP riêng biệt giữa máy khách và máy chủ pfSense. Máy khách sau khi kết nối sẽ nhận một địa chỉ IP thuộc dải mạng ảo riêng (ví dụ: `10.0.8.0/24`) và được định tuyến để truy cập tài nguyên nội bộ.

### Bước 1: Tạo Certificate Authority (CA)

Truy cập menu **System > Cert. Manager**, sau đó nhấn **Add** để tạo mới một CA.

<img width="1499" height="635" alt="image" src="https://github.com/user-attachments/assets/a86eddcd-dfba-4e08-8b9a-15cf20da37e6" />


Điền các thông tin cần thiết theo hướng dẫn sau:

- **Descriptive Name**: Đặt tên nhận diện cho chứng chỉ CA.
- **Method**: Chọn `Create an internal Certificate Authority`.
- **Key Type**: Chọn loại khóa phù hợp (RSA, ECDSA, …).
- **Key Length**: Chọn độ dài khóa (2048, 4096, …).
- **Digest Algorithm**: Chọn thuật toán băm (sha256, sha512, …).
- **Common Name**: Đặt tên chung cho CA (ví dụ: `internal-ca`, `own-ca`, …).

Nhấn **Save** để lưu lại.



---

### Bước 2: Tạo chứng chỉ cho máy chủ (Server Certificate)


Truy cập **System > Certificates**, chọn tab **Certificates** trong sub-menu, sau đó nhấn **Add/Sign** ở góc phải bên dưới.

<img width="1540" height="898" alt="image" src="https://github.com/user-attachments/assets/de640611-80da-4b54-9a7a-15ce15ae4d97" />

Điền các thông tin theo hướng dẫn:

- **Method**: Chọn `Create an internal Certificate`.
- **Descriptive Name**: Nhập tên mô tả cho chứng chỉ.
- **Key Length** và **Digest Algorithm**: Đặt giống với CA đã tạo ở Bước 1.
- **Lifetime**: Đặt thời hạn hiệu lực là `365` ngày (không vượt quá 398 ngày theo tiêu chuẩn trình duyệt hiện hành).
- **Common Name**: Đặt tên chung cho chứng chỉ máy chủ.
- **Certificate Type**: Chọn `Server Certificate`.

Nhấn **Save** để lưu. Server Certificate đã được tạo hoàn tất.

<img width="1542" height="830" alt="image" src="https://github.com/user-attachments/assets/9ce1b613-3b6e-4a2d-88a4-8fbd559d04e1" />


---

### Bước 3: Tạo tài khoản người dùng OpenVPN

Truy cập **System > User Manager**, nhấn **Add** và điền thông tin người dùng.

- **Username**: Nhập tên đăng nhập.
- **Password**: Nhập mật khẩu.

<img width="1373" height="669" alt="image" src="https://github.com/user-attachments/assets/f985b37e-2488-4825-b14c-010e40d2df62" />

Nhấn **Save** để lưu. Hệ thống sẽ chuyển trở về giao diện danh sách người dùng.

<img width="1456" height="473" alt="image" src="https://github.com/user-attachments/assets/d21a1d31-2307-4d36-ba0c-f3b9418d6591" />


Tiếp theo, thiết lập phương thức xác thực. Hệ thống hỗ trợ xác thực bằng chứng chỉ, mật khẩu, hoặc cả hai. Nhấn biểu tượng chỉnh sửa (icon hình bút chì) bên cạnh tài khoản vừa tạo, sau đó chọn **Add → User Certificate** để gắn chứng chỉ cho người dùng.


Điền các thông tin chứng chỉ người dùng:

- **Method**: Chọn `Create an internal Certificate`.
- **Key Length**, **Key Type**, **Digest Algorithm**: Đặt giống với CA đã tạo.
- **Lifetime**: Đặt `365` ngày.
- **Certificate Type**: Xác nhận là `User Certificate`.

Nhấn **Save** để hoàn tất liên kết giữa chứng chỉ người dùng và tài khoản OpenVPN.

<img width="1075" height="848" alt="image" src="https://github.com/user-attachments/assets/2ac57198-6bfb-4cfc-8cc0-917a8a8fef10" />

---

### Bước 4: Tạo máy chủ VPN (OpenVPN Server)

Truy cập **VPN > OpenVPN**, nhấn **Add** để tạo máy chủ mới.

<img width="1068" height="405" alt="image" src="https://github.com/user-attachments/assets/94f03366-1412-48fe-a330-b0a3858725e6" />

#### 4.1. Thông tin chung (General Information)

- **Server Mode**: Chọn `Remote Access (SSL/TLS + User Auth)`.
- **Device Mode**: Chọn `tun – Layer 3 Tunnel Mode`.
- **Local Port**: Nhập cổng kết nối, trong hướng dẫn này sử dụng cổng `1194`.
- **Description**: Nhập mô tả cho máy chủ VPN.


<img width="1231" height="740" alt="image" src="https://github.com/user-attachments/assets/8e4a097f-c45b-4c79-a65b-811e78bde97b" />


#### 4.2. Cài đặt mật mã (Cryptographic Settings)

- Bật **TLS Key** và chọn **Automatically generate a TLS Key** để hệ thống tự sinh khóa TLS.
- **Peer Certificate Authority**: Chọn CA đã tạo ở Bước 1.
- **Server Certificate**: Chọn chứng chỉ máy chủ đã tạo ở Bước 2.
- **DH Parameter Length**: Chọn `4096`.
- **Auth Digest Algorithm**: Chọn `SHA512 (512-bit)`.

<img width="1277" height="778" alt="image" src="https://github.com/user-attachments/assets/a5131da8-d547-4af7-a43c-80c7dd7ade81" />

#### 4.3. Cài đặt đường hầm (Tunnel Settings)

- **IPv4 Tunnel Network**: Đặt dải mạng ảo, ví dụ `10.0.8.0/24`.
- **IPv6 Tunnel Network**: Để trống nếu không sử dụng IPv6.
- Bật **Redirect IPv4 Gateway** để chuyển hướng toàn bộ lưu lượng IPv4 của máy khách qua VPN.

<img width="1212" height="589" alt="image" src="https://github.com/user-attachments/assets/1db1dc3d-9e83-4507-84ae-83013463ab3a" />

#### 4.4. Cấu hình nâng cao (Advanced Configuration)

- Bật **UDP Fast I/O** để tăng hiệu suất xử lý gói tin UDP.
- **Gateway Creation**: Chọn `IPv4 Only` nếu chỉ sử dụng IPv4; chọn `Both` nếu cần hỗ trợ cả IPv4 lẫn IPv6.

<img width="1159" height="658" alt="image" src="https://github.com/user-attachments/assets/500bd3c8-5fd9-4fd1-9bb5-0934e672a3e8" />


Nhấn **Save** để lưu và áp dụng toàn bộ cấu hình máy chủ OpenVPN.

---

### Bước 5: Xác minh trạng thái dịch vụ

Truy cập **Status > Services** để kiểm tra dịch vụ OpenVPN đã khởi động thành công. Trạng thái hiển thị màu xanh (Running) xác nhận máy chủ đang hoạt động bình thường.

<img width="1305" height="246" alt="image" src="https://github.com/user-attachments/assets/035b44a5-c02a-4eed-9468-c57825a7b8c4" />

---

### Bước 6: Cấu hình Firewall — Cho phép lưu lượng trong tunnel VPN

Truy cập **Firewall > Rules**, chọn tab **OpenVPN**, nhấn **Add** để thêm rule mới.

- **Action**: Pass
- **Address Family**: IPv4
- **Protocol**: Any
- **Source**: Network — nhập dải IP đã thiết lập ở Bước 4 (ví dụ: `10.0.8.0/24`).
- **Description**: Nhập mô tả rõ ràng cho rule.

Nhấn **Save** và **Apply Changes** để áp dụng cấu hình.

<img width="1063" height="729" alt="image" src="https://github.com/user-attachments/assets/d18fd7a5-fe3f-40bf-8453-f9a95a682c49" />

---

### Bước 7: Cấu hình Firewall — Cho phép kết nối vào cổng VPN từ ngoài WAN

Truy cập **Firewall > Rules**, chọn tab **WAN**, nhấn **Add** để thêm rule mới.

- **Action**: Pass
- **Address Family**: IPv4
- **Protocol**: UDP
- **Source**: Any
- **Destination Port Range**: `1194` đến `1194` (cổng đã thiết lập ở Bước 4).
- **Description**: Nhập mô tả.

Nhấn **Save** và **Apply Changes** để áp dụng.
<img width="1172" height="272" alt="image" src="https://github.com/user-attachments/assets/d0d307bf-a582-4116-a658-a8bd7106afc1" />


---

### Bước 8: Cài đặt tiện ích OpenVPN Client Export Utility

Tiện ích này hỗ trợ xuất file cấu hình VPN Client (.ovpn) trực tiếp từ giao diện pfSense, giúp đơn giản hoá quá trình cấu hình phía người dùng.

Truy cập **System > Package Manager**, chọn tab **Available Packages**. Tìm kiếm gói `openvpn-client-export` và nhấn **Install**.

<img width="1193" height="480" alt="image" src="https://github.com/user-attachments/assets/4ac6f3e0-88c1-421c-a113-904192d083ac" />

Nhấn **Confirm** để xác nhận cài đặt. Khi giao diện **Package Installer** hiển thị thông báo **Success**, quá trình cài đặt đã hoàn tất.

---

### Bước 9: Xuất file cài đặt VPN Client

Truy cập **VPN > OpenVPN**, chọn tab **Client Export** trong sub-menu.

<img width="1210" height="713" alt="image" src="https://github.com/user-attachments/assets/1d2b618d-66d1-4f42-ad0b-7d6a8848d27f" />

Tại mục **Remote Access Server**, chọn đúng máy chủ OpenVPN đã tạo.

- Nếu sử dụng **Dynamic DNS** để truy cập WAN của pfSense: Chọn **Other** tại **Host Name Resolution** và nhập tên miền DDNS. Cấu hình này cho phép truy cập máy chủ OpenVPN bằng tên miền thay vì địa chỉ IP, tránh mất kết nối khi ISP thay đổi IP động.
- Nếu không sử dụng DDNS: Đặt **Host Name Resolution** thành **Interface IP Address**.

Kéo xuống danh sách người dùng, chọn tài khoản cần xuất và tải về file cài đặt phù hợp với hệ điều hành của máy khách.

<img width="1243" height="459" alt="image" src="https://github.com/user-attachments/assets/78528b11-caec-4cca-a4f0-b4f02cffa361" />


---

### Bước 10: Kiểm tra kết nối OpenVPN

Chạy file cài đặt VPN Client vừa tải về trên máy khách. Sau khi cài đặt hoàn tất, nhập thông tin tài khoản người dùng đã tạo ở Bước 3 và thực hiện kết nối.

<img width="766" height="595" alt="image" src="https://github.com/user-attachments/assets/d0ee0b7e-2bb7-400d-b043-70926bac4e16" />

> **Lưu ý:** Bỏ tích hai tùy chọn sau trong phần cài đặt của OpenVPN Connect để tránh gặp lỗi kết nối:

<img width="1287" height="352" alt="image" src="https://github.com/user-attachments/assets/0e135e5f-5cbe-409d-a687-81a87021fece" />

Sau khi kết nối thành công, kiểm tra thông tin mạng trên máy khách để xác nhận đã nhận địa chỉ IP thuộc dải tunnel VPN (`10.0.8.x`):

<img width="614" height="269" alt="image" src="https://github.com/user-attachments/assets/0827fc0d-e445-4657-b77b-4dfd2abeef22" />

<img width="866" height="591" alt="image" src="https://github.com/user-attachments/assets/f84041be-82ee-4ef8-97bf-45d3386e28f0" />

<img width="1083" height="218" alt="image" src="https://github.com/user-attachments/assets/80f2b27b-05bd-45ba-9a6a-29b75025e35d" />

---

## 3. Triển khai chế độ VPN TAP (Layer 2 Bridge)

Chế độ TAP hoạt động ở tầng liên kết dữ liệu (Layer 2), biến pfSense thành một switch ảo, cho phép kéo thẳng máy khách từ xa vào cùng dải mạng LAN nội bộ (ví dụ: `172.16.1.x`). Nhờ đó, máy khách VPN có thể nhận địa chỉ IP từ DHCP server của LAN, truy cập tài nguyên chia sẻ (SMB, NetBIOS, …) như thể đang ngồi trong cùng mạng vật lý.

---

