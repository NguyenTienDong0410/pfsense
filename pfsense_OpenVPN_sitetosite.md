## TRIỂN KHAI OPENVPN SITE-TO-SITE KẾT HỢP HA (CARP)

## 1. Phân Tích Mô Hình & Quy Hoạch Địa Chỉ IP

Trước khi cấu hình trên VMware, cần thống nhất thông số mạng dựa trên sơ đồ thiết kế:

**Main Site (Trung Tâm - Cấu hình HA):**

* **Mạng WAN (vLAN 8):** `192.168.197.0/24`
* pfSense Master WAN IP: `192.168.197.183`
* pfSense Backup WAN IP: `192.168.197.184`
* **WAN CARP VIP:** `192.168.197.186` *(Dùng làm IP public cho OpenVPN Server)*


* **Mạng LAN (vLAN 2):** `172.16.1.0/24`
* pfSense Master LAN IP: `172.16.1.10`
* pfSense Backup LAN IP: `172.16.1.11`
* **LAN CARP VIP:** `172.16.1.12` *(Dùng làm Default Gateway cho Client)*


* **Mạng SYNC (Mạng nội bộ riêng để đồng bộ trạng thái - Khuyến nghị thêm):** Cấp một dải IP cấp phát riêng (VD: `192.168.254.0/24`) nối trực tiếp giữa Master và Backup.

**Site A (Chi Nhánh):**

* **Mạng WAN (vLAN 8):** Cùng dải hoặc qua Internet mô phỏng để ping thấy IP WAN VIP (`192.168.197.186`).
* **Mạng LAN (vLAN 3):** `10.0.0.0/24`
* pfSense Site A LAN IP (Gateway): `10.0.0.10`



---

## 2. Chuẩn Bị Môi Trường VMware

Sử dụng **Virtual Network Editor** trong VMware Workstation/ESXi để tạo các vSwitch (VMnet) tương ứng với các vLAN:

1. **VMnet8 (NAT/Custom):** Đóng vai trò là môi trường Internet/WAN. Gán card WAN của cả 3 máy ảo pfSense vào VMnet này.
2. **VMnet2 (Host-only/Custom):** Đóng vai trò là LAN Main Site. Gán card LAN của pfSense Master, Backup và Client Main Site vào đây.
3. **VMnet3 (Host-only/Custom):** Đóng vai trò là LAN Site A. Gán card LAN của pfSense Site A và Client Site A vào đây.
4. **VMnet4 (Host-only/Custom):** Đóng vai trò là đường SYNC nối trực tiếp giữa Master và Backup (Thêm card mạng thứ 3 cho 2 máy ảo này).

> **Lưu ý quan trọng trên VMware:** Để CARP hoạt động, bạn phải cho phép **Promiscuous Mode** và **Forged Transmits** trên các vSwitch (đối với ESXi) hoặc chạy VMware Workstation dưới quyền Administrator.

---

## 3. Cấu Hình High Availability (CARP) Tại Main Site

### 3.1. Cấu hình cơ bản Master & Backup

* Cài đặt cơ bản 2 node pfSense, gán đúng IP tĩnh cho các interface WAN, LAN, và SYNC theo quy hoạch ở Mục 1.

### 3.2. Cấu hình Virtual IPs (CARP) trên Node Master

1. Truy cập **Firewall > Virtual IPs**, chọn **Add**.
2. **Tạo WAN VIP:**
* Type: `CARP` | Interface: `WAN` | Address: `192.168.197.186 / 24`.
* Virtual IP Password: Nhập mật khẩu.
* VHID Group: `1` (Số định danh duy nhất).
* Advertising Frequency: Base `1`, Skew `0` (Master luôn để Skew thấp hơn Backup).


3. **Tạo LAN VIP:**
* Thực hiện tương tự nhưng chọn Interface: `LAN` | Address: `172.16.1.12 / 24` | VHID Group: `2`.



### 3.3. Cấu hình Outbound NAT & Firewall Rules

1. Truy cập **Firewall > NAT > Outbound**. Chuyển sang chế độ **Hybrid Outbound NAT** hoặc **Manual**.
2. Chỉnh sửa/Thêm các rule NAT cho mạng LAN, đổi phần **Translation (NAT Address)** từ IP của WAN interface sang **WAN CARP VIP (`192.168.197.186`)**.
3. Truy cập **Firewall > Rules > SYNC**, tạo rule cho phép toàn bộ traffic (IPv4 * * * * *) để 2 node giao tiếp.

### 3.4. Cấu hình đồng bộ (XMLRPC Sync)

1. Trên node **Master**, vào **System > High Avail. Sync**.
2. Tích chọn **Synchronize states**, chọn interface `SYNC`.
3. Tại phần **XMLRPC Sync**, nhập IP SYNC của node Backup, Username (`admin`) và Password.
4. Tích chọn các mục cần đồng bộ (Firewall rules, NAT, Virtual IPs, **OpenVPN**...). Nhấn Save.
5. Sang node Backup kiểm tra, các VIP và Rules đã được đẩy sang tự động. Node Backup sẽ có trạng thái CARP là *BACKUP*.

---

## 4. Triển Khai OpenVPN Site-to-Site

Sử dụng phương pháp **Shared Key (Peer to Peer)** cho đơn giản và hiệu quả với Site-to-Site.

### 4.1. Cấu hình OpenVPN Server trên Main Site (Thực hiện trên Master)

1. Truy cập **VPN > OpenVPN > Servers**, chọn **Add**.
2. **General Information:**
* Server Mode: `Peer to Peer (Shared Key)`
* Protocol: `UDP on IPv4 only`
* Interface: Chọn **WAN CARP VIP (`192.168.197.186`)** *(Bước này mang tính quyết định để VPN tự động chuyển đổi khi node Master sập).*
* Local Port: `1194`
<img width="1433" height="828" alt="image" src="https://github.com/user-attachments/assets/735a44af-8070-43a5-9df4-79a1520a1247" />


3. **Cryptographic Settings:**
* Tự động Generate Shared Key. (Copy đoạn mã này ra Notepad để dùng cho Site A).


4. **Tunnel Settings:**
* IPv4 Tunnel Network: Chọn một dải ảo, ví dụ .
* IPv4 Local Network: `172.16.1.0/24` (LAN Master).
* IPv4 Remote Network: `10.0.0.0/24` (LAN Site A).
<img width="1518" height="675" alt="image" src="https://github.com/user-attachments/assets/46b9ea71-9d07-464d-85b3-1b88c438dc99" />


5. Lưu cấu hình. Hệ thống HA sẽ tự động đồng bộ cấu hình Server này sang node Backup.

<img width="1482" height="478" alt="image" src="https://github.com/user-attachments/assets/b3669137-ed8a-41fc-ae14-292a465de7e3" />

### 4.2. Cấu hình OpenVPN Client trên Site A

1. Truy cập **VPN > OpenVPN > Clients**, chọn **Add**.
2. **General Information:**
* Server Mode: `Peer to Peer (Shared Key)`
* Protocol: `UDP on IPv4 only`
* Interface: `WAN`
* Server host or address: **`192.168.197.186`** (IP WAN VIP của Main Site).
* Server port: `1194`.

<img width="1480" height="723" alt="image" src="https://github.com/user-attachments/assets/d81944e3-691d-48f1-a842-f16d8c118a89" />

<img width="1440" height="738" alt="image" src="https://github.com/user-attachments/assets/a676b763-d522-4711-893e-050099626e62" />

3. **Cryptographic Settings:**
* Bỏ chọn "Automatically generate". Dán đoạn Shared Key đã copy từ Server vào đây.

#
# 2048 bit OpenVPN static key
#
-----BEGIN OpenVPN Static key V1-----
95156db26ff0315377f92e20685c3730
2e3cb6aac458a8aabc3205cb574705e3
f4abcf1b2aa649b80c886c87505d632c
29756fd3e6913482735dc90cfc8654bb
a6ec4ce2f0f239f944cba10f19929d00
270245b1a6eaaaf8d9a35e4998bcd12e
d06c9eed5eb9ef14838865e81663718f
24829920a547e285fd531fe8592bda6d
dd2e60eef41e4c9822c029042aa177e6
00f88dddcb2b81ea8d1a9fafeab28512
61f51dbacfe19e64a31e5e944cbbfa6b
6cced60b6265dfbee705e3cceac4026a
17b5b1a8a848e7637b39cfceee9d365c
4ad8201d67a55a91892d24f40ae154bb
ae1b72fc8bec8b15c94155f48dc72798
68da6055362e60bfa7567df286b0255c
-----END OpenVPN Static key V1-----

<img width="1143" height="750" alt="image" src="https://github.com/user-attachments/assets/9fd9cb7d-59f1-494e-aa30-431b105e5742" />

4. **Tunnel Settings:**
* IPv4 Tunnel Network: 
* IPv4 Remote Network: 
<img width="1480" height="662" alt="image" src="https://github.com/user-attachments/assets/084f7e86-3429-4469-adc6-9a3fbc6a03eb" />



5. Lưu cấu hình.

<img width="1464" height="437" alt="image" src="https://github.com/user-attachments/assets/cda2123a-e78a-434b-bc6a-ed5039f68b84" />

### 4.3. Cấu hình Firewall Rules cho OpenVPN (Cả 2 Site)

1. **Main Site (Master):** Vào **Firewall > Rules > WAN**, mở port UDP `1194` đích đến là `192.168.197.186`.
   <img width="1490" height="645" alt="image" src="https://github.com/user-attachments/assets/03ecf008-2eba-4a4e-86e4-a7a6193b5792" />

<img width="1476" height="743" alt="image" src="https://github.com/user-attachments/assets/62b3de0d-8528-45f5-b900-672288b45d49" />

3. **Cả 2 Site:** Vào **Firewall > Rules > OpenVPN**, tạo rule cho phép toàn bộ traffic (IPv4 * * * * *) đi qua đường hầm VPN.

<img width="1523" height="374" alt="image" src="https://github.com/user-attachments/assets/2a0db4b5-7e78-4d83-b853-58b018d6524c" />

---

## 5. Kiểm Tra Và Chuyển Giao (Failover Testing)

**Giai đoạn 1: Kiểm tra kết nối mạng (Bình thường)**

* Mở Client ở Main Site (`172.16.1.21`), ping đến Client ở Site A (`10.0.0.20`). Kết quả phải có Reply.
  <img width="573" height="296" alt="image" src="https://github.com/user-attachments/assets/a88b0000-9551-400b-bfbe-bbc52d1606ab" />
ping thành công và tương tự với chiều ngược lại
<img width="679" height="401" alt="image" src="https://github.com/user-attachments/assets/79f7f6b6-a5a1-4acd-8944-50a0e9ae97b9" />


* Trên pfSense (Status > OpenVPN), trạng thái Tunnel phải là **UP**.
  
<img width="857" height="337" alt="image" src="https://github.com/user-attachments/assets/867e419d-c578-4146-8ef6-95bee98b7ffe" />

<img width="759" height="267" alt="image" src="https://github.com/user-attachments/assets/ec380d30-518e-4857-810f-77e65f2dfb93" />


**Giai đoạn 2: Giả lập sự cố Kịch bản High Availability**

1. Ping liên tục từ Client Main Site sang Client Site A: `ping 10.0.0.20 -t`.
2. Trên VMware, **Shutdown** máy ảo pfSense Master (hoặc Disconnect card mạng WAN/LAN).
3. **Kết quả mong đợi:**
* Chỉ rớt khoảng 1-3 gói tin request time out.
* Node Backup ngay lập tức chiếm lấy (take over) dải VIP (`192.168.197.186` & `172.16.1.12`).
* OpenVPN Client tại Site A tự động kết nối lại (re-establish) tunnel tới node Backup thông qua IP WAN VIP.
* Client tại Main Site tiếp tục truy cập mạng bình thường do Default Gateway là LAN VIP không đổi.

* <img width="712" height="505" alt="image" src="https://github.com/user-attachments/assets/8c38cb18-da45-4903-a506-7e945270b8cc" />
