
## 1.1 Giới thiệu về pfSense

**pfSense** là một nền tảng tường lửa (firewall) và bộ định tuyến (router) mã nguồn mở vô cùng mạnh mẽ, được xây dựng dựa trên hệ điều hành **FreeBSD**. Được duy trì bởi Netgate, pfSense có thể cài đặt trên các phần cứng vật lý chuyên dụng hoặc trên môi trường ảo hóa như VMware, VirtualBox, Proxmox.

Nhờ sở hữu giao diện quản lý trên nền web (WebGUI) trực quan và linh hoạt, pfSense cho phép người quản trị hệ thống dễ dàng cấu hình các tính năng mạng phức tạp mà không cần phải gõ lệnh thủ công. Nó thường được triển khai tại rìa mạng (edge) để bảo vệ hệ thống nội bộ, kết nối các chi nhánh, và quản lý luồng truy cập của người dùng.

## 1.2 Một số lợi ích của pfSense

pfSense được các quản trị viên hệ thống (System/Network Administrators) đặc biệt ưa chuộng nhờ những ưu điểm vượt trội:

* **Hoàn toàn miễn phí (Mã nguồn mở):** Cung cấp các tính năng cấp độ doanh nghiệp (enterprise-grade) mà không tốn chi phí cấp phép bản quyền (license) đắt đỏ như các thiết bị của Cisco, Fortinet hay Palo Alto.
* **Định tuyến và Tường lửa mạnh mẽ:** Hỗ trợ Stateful Firewall, định tuyến tĩnh/động (OSPF, BGP) và NAT (Network Address Translation) linh hoạt.
* **Đa dạng kết nối VPN:** Tích hợp sẵn IPsec, OpenVPN và WireGuard, giúp thiết lập mạng riêng ảo an toàn cho nhân viên làm việc từ xa hoặc kết nối Site-to-Site giữa các văn phòng.
* **Tính năng bảo mật mở rộng:** Dễ dàng cài đặt thêm các gói (packages) như **Snort** hoặc **Suricata** để phát hiện và ngăn chặn xâm nhập (IDS/IPS), hoặc **pfBlockerNG** để chặn quảng cáo và lọc IP độc hại.
* **Cân bằng tải và Dự phòng (HA):** Hỗ trợ Load Balancing nhiều đường truyền WAN và cấu hình High Availability (CARP) để đảm bảo mạng luôn hoạt động ngay cả khi một thiết bị chết.
* **Quản lý truy cập (Captive Portal):** Cho phép tạo trang đăng nhập dành cho mạng Wi-Fi khách (Guest Wi-Fi) để kiểm soát và giới hạn băng thông.

---

## 1.3 Cài đặt pfSense trên VMWare

Trước khi bắt đầu, hãy tải file ISO cài đặt pfSense bản Community Edition (AMD64) từ trang chủ Netgate và giải nén file `.iso` ra máy tính.
https://repo.ialab.dsu.edu/pfsense/

1. **Tạo máy ảo (Virtual Machine) trên VMware:**
Mở VMware, chọn **Create a New Virtual Machine** và thiết lập các thông số cơ bản:

* **Guest Operating System:** Chọn `Other` > `FreeBSD 64-bit` (bởi vì pfSense chạy trên nhân FreeBSD).
* **RAM:** Tối thiểu 1GB (khuyến nghị 2GB trở lên nếu có dùng các tính năng nặng như Snort/VPN).
* **CPU:** 1 hoặc 2 cores.
* **Disk Size:** 10GB - 20GB là quá đủ cho cài đặt cơ bản.
* Trỏ file cài đặt CD/DVD vào file `.iso` của pfSense bạn vừa tải.


2. **Thiết lập Card mạng (Network Adapters) - Bước cốt lõi:** Phải có ít nhất 2 card mạng để làm Tường lửa.
Một firewall cần phân tách mạng bên ngoài và mạng bên trong. Bạn cần điều chỉnh phần cứng máy ảo:

<img width="1156" height="257" alt="image" src="https://github.com/user-attachments/assets/e915ac08-0db0-44b9-8493-3435233ab854" />

* **Network Adapter 1 (sẽ làm WAN):** Chọn `Bridged` (để nhận IP cùng dải mạng thật của bạn) hoặc `NAT` (để nhận IP do VMware cấp ra Internet).
* **Network Adapter 2 (sẽ làm LAN):** Nhấn *Add...* để thêm một card mạng thứ hai. Chọn `Custom: Specific virtual network` và chọn một VMnet tùy ý (ví dụ: `VMnet2` hoặc `LAN segment`). Các máy ảo client sau này muốn vào mạng phải kết nối chung vào VMnet này.

Tạo các VMnet với dải mạng như trên: Vmnet2: 172.16.1.0 và có sẵn VMnet8 của Vmware với dải mạng là : 192.168.197.0

<img width="950" height="808" alt="image" src="https://github.com/user-attachments/assets/8fadc8c9-49ed-4c06-bf9d-2725ab44c9d8" />

Trên máy client
<img width="1089" height="255" alt="image" src="https://github.com/user-attachments/assets/9dfbf657-0fe5-4ecb-96df-dcc4a2aa2de2" />

3. **Bắt đầu cài đặt hệ điều hành pfSense:**
Khởi động máy ảo (Power on). Quá trình boot sẽ tự động diễn ra. Khi màn hình menu hiện lên, bạn thực hiện các thao tác sau (dùng phím mũi tên và Enter để chọn):

* **Accept** các điều khoản (Copyright).
* Chọn **Install** pfSense.
* **Keymap Selection:** Cứ để mặc định (Continue with default keymap).
* **Partitioning:** Chọn **Auto (ZFS)** (khuyên dùng vì tính ổn định và tự sửa lỗi tốt) hoặc **Auto (UFS)**. Tiếp tục nhấn Enter cho các tùy chọn mặc định của ZFS (stripe).
* Quá trình giải nén file hệ thống mất khoảng 1-2 phút.


4. **Hoàn tất và Khởi động lại:**
Khi hệ thống hỏi "Manual Configuration", chọn **No**. Sau đó chọn **Reboot**.
*Lưu ý:* Khi máy ảo bắt đầu khởi động lại, nhớ ngắt kết nối ổ CD/DVD (hoặc đổi thứ tự boot ưu tiên ổ cứng) để tránh máy ảo lại boot vào bộ cài đặt một lần nữa.


---

## 1.4 Cấu hình cho pfSense

Sau khi khởi động lại xong, màn hình console (dòng lệnh) của pfSense sẽ xuất hiện. Lúc này bạn cần làm 2 việc: gán các card mạng bằng dòng lệnh, và thiết lập cấu hình trên trình duyệt web.
menu của pfsense ctl
<img width="849" height="558" alt="image" src="https://github.com/user-attachments/assets/bc57e29f-5262-43cb-8309-93c0a87578af" />

1. **Gán cổng mạng (Assign Interfaces) trên Console:**
Hệ thống sẽ hỏi bạn có muốn thiết lập VLAN không (`Should VLANs be set up now`). Nhấn **n** (No) và Enter.
Sau đó hệ thống sẽ yêu cầu bạn chỉ định cổng WAN và LAN (dựa trên tên card mạng ảo, thường là `em0`, `em1` hoặc `vmx0`, `vmx1`):

* **Enter the WAN interface name:** Gõ tên cổng đầu tiên (ví dụ: `em0`) rồi Enter.
* **Enter the LAN interface name:** Gõ tên cổng thứ hai (ví dụ: `em1`) rồi Enter.
* Hệ thống hỏi về cổng tùy chọn (OPT1), nhấn Enter để bỏ qua.
* Xác nhận lại bảng map cổng mạng bằng cách gõ **y** và Enter.

<img width="920" height="573" alt="image" src="https://github.com/user-attachments/assets/52bb4bc7-061b-4426-8d31-1e5b8339457c" />

<img width="819" height="560" alt="image" src="https://github.com/user-attachments/assets/28c2e8fa-ecde-4390-9296-ec9c97fceb82" />

2. **Thiết lập địa chỉ IP tĩnh cho cổng Wan va LAN:**
Tại Menu chính của console (pfSense menu), chọn số **2 (Set interface(s) IP address)**.

Chọn số 1 là cổng WAN, cấu hình như sau với ip tĩnh là 192.168.197.183/24

<img width="768" height="552" alt="image" src="https://github.com/user-attachments/assets/31f27eaa-8578-448a-8f6c-cad5da678f28" />

<img width="799" height="578" alt="image" src="https://github.com/user-attachments/assets/7818f9ea-b7c9-4482-8fda-71bf73c6918a" />

* Chọn cổng **LAN** (thường là phím số 2).
  
* <img width="899" height="487" alt="image" src="https://github.com/user-attachments/assets/40df77c0-da52-4da6-a668-edc421560d81" />

* Nhập địa chỉ IP mới, ví dụ: `172.16.1.10`.
* Nhập Subnet Mask: `24` (tương đương 255.255.255.0).
  
* <img width="760" height="507" alt="image" src="https://github.com/user-attachments/assets/5e05e580-880b-4e39-92fd-dfc990d5b6c3" />

* Upstream gateway: Nhấn Enter để trống (chỉ set gateway cho WAN, không set cho LAN).
* Bật DHCP Server cho mạng LAN: Bấm **y**.
* Nhập dải IP sẽ cấp phát: Start (VD: `172.16.1.20`), End (VD: `172.16.1.200`).
* Revert to HTTP webConfigurator: Nhấn **n** (nên để mặc định dùng HTTPS).

<img width="783" height="341" alt="image" src="https://github.com/user-attachments/assets/e7403bd9-f4df-48a6-a81c-ab5829ac37f7" />

<img width="824" height="258" alt="image" src="https://github.com/user-attachments/assets/f71661c1-e80b-4018-a7fb-e20b431f588c" />


3. **Đăng nhập vào giao diện WebGUI:**

Mở một máy ảo khác (Client ảo chạy Windows/Ubuntu) có card mạng được kết nối vào chung VMnet2 (LAN) với pfSense.
Ở đây có máy Window Server 22 đã cấu hình cùng chung Lan 
Thêm DNS cho máy ảo
<img width="625" height="592" alt="image" src="https://github.com/user-attachments/assets/3d259737-df8d-4bd4-b478-ffe2c3c12bf5" />


* Mở trình duyệt web trên máy ảo đó, truy cập địa chỉ IP bạn vừa thiết lập: `[https://192.168.10.254](https://192.168.10.254)`.
* Trình duyệt sẽ cảnh báo bảo mật SSL, cứ bỏ qua (Proceed / Accept the risk).
* Tài khoản đăng nhập mặc định là:
* **Username:** `admin`
* **Password:** `pfsense`

<img width="1516" height="682" alt="image" src="https://github.com/user-attachments/assets/dce0940d-b950-4290-b58a-cd5a0b1ed54f" />



4. **Hoàn tất Setup Wizard trên WebGUI:** Các bước thiết lập khởi tạo trên nền web.
Ngay lần đăng nhập đầu tiên, màn hình Setup Wizard sẽ hiện ra. Bạn nhấn Next và cấu hình theo các bước:

* **General Information:** Đặt tên cho máy (Hostname: `pfsense`), tên miền (Domain: `localdomain`), khai báo Primary DNS (VD: `8.8.8.8`, `1.1.1.1`).
* <img width="1019" height="626" alt="image" src="https://github.com/user-attachments/assets/80ce6be6-1897-44ab-9dc7-7a47f051bf45" />

* **Time Server:** Chọn Timezone phù hợp (ví dụ: `Asia/Ho_Chi_Minh`).
* <img width="892" height="512" alt="image" src="https://github.com/user-attachments/assets/04a0b6a4-877b-4a94-946f-cb6aa2fc098f" />

* **WAN Configuration:** Mặc định nhận IP động (DHCP). Nếu mạng ảo cấp tĩnh thì chọn Static. Dưới cùng có 2 tick box chặn dải mạng nội bộ (Block RFC1918) và dải mạng giả mạo (Block bogon networks) đi từ ngoài vào WAN, thường cứ để mặc định.
<img width="869" height="676" alt="image" src="https://github.com/user-attachments/assets/56510fc7-2ab3-48f2-b898-6be0d0a3c28c" />

* **LAN Configuration:** Kiểm tra lại IP của LAN (đã cấu hình đúng ở console rồi thì Next).
* <img width="1066" height="492" alt="image" src="https://github.com/user-attachments/assets/def2d730-717f-41f8-9f8f-3045779d88ee" />

* **Set Admin Password:** Đổi mật khẩu mới cho tài khoản `admin`. Dong123@
* Cuối cùng nhấn **Reload** để pfSense áp dụng cấu hình và kết thúc quy trình.


Lúc này, hệ thống pfSense của bạn đã hoàn toàn sẵn sàng định tuyến Internet từ WAN vào cho các máy con trong dải mạng LAN.
