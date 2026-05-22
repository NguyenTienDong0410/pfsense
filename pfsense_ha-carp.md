Triển khai High Availability (HA) - CARP trên pfSense
---


**Phần 1: Tổng quan về kiến trúc High Availability trên pfSense**

* 1.1. Khái niệm High Availability (HA) và tầm quan trọng trong hệ thống mạng.
  
 High Availability có nghĩa là “Độ sẵn sàng cao“, những máy chủ, thiết bị loại này luôn luôn sẵn sàng phục vụ, người sử dụng không cảm thấy nó bị trục trặc, hỏng hóc gây gián đoạn. Để đảm bảo được điều đó, tối thiểu có một cặp máy, thiết bị chạy song song, liên tục liên lạc với nhau, cái chính hỏng, cái phụ sẽ lập tức biết và tự động thay thế.

 Thông thường các nhóm máy chủ được gọi là "CARP cluster" nhưng CARP chỉ là một phần. High Availability đạt được bằng cách sử dụng sự kết hợp của nhiều kỹ thuật có liên quan, bao gồm cả CARP, trạng thái đồng bộ (pfsync), và cấu hình đồng bộ (XMLRPC Sync).

 
* 1.2. Giao thức CARP (Common Address Redundancy Protocol) hoạt động như thế nào?
Giao thức CARP (Common Address Redundancy Protocol) là một giao thức cung cấp tính năng dự phòng địa chỉ IP (failover), cho phép nhiều thiết bị mạng chia sẻ chung một địa chỉ IP ảo (Virtual IP - VIP) nhằm loại bỏ điểm lỗi duy nhất (Single Point of Failure) và đảm bảo hệ thống luôn hoạt động xuyên suốt.

CARP hoạt động dựa trên mô hình Master - Backup (hoặc Master - Slave) xoay quanh việc nắm giữ địa chỉ IP ảo.

Virtual IP (VIP): Các thiết bị trong cụm (cluster) được cấu hình để dùng chung một địa chỉ IP và một địa chỉ MAC ảo. Các máy trạm (client) trong mạng LAN sẽ lấy VIP này làm Default Gateway thay vì trỏ đến IP vật lý của từng thiết bị.

Trạng thái Master: Tại bất kỳ thời điểm nào, chỉ có một thiết bị duy nhất (Master) nắm giữ VIP. Nó chịu trách nhiệm phản hồi ARP request cho VIP và xử lý toàn bộ lưu lượng mạng (như Routing, NAT, VPN) gửi đến địa chỉ này.

Trạng thái Backup: Các thiết bị còn lại sẽ đóng vai trò dự phòng, liên tục lắng nghe tín hiệu từ Master.

Heartbeat (Nhịp tim): Thiết bị Master liên tục phát các gói tin quảng bá (heartbeat) ra môi trường mạng để thông báo rằng nó vẫn đang sống. Theo mặc định, các nhịp tim này được gửi đi 1 lần mỗi giây.

* 1.3. Phân biệt các cơ chế đồng bộ:
* Đồng bộ trạng thái kết nối (pfsync - State Synchronization).
* Tường lửa pfSense hoạt động theo cơ chế Stateful (lưu trạng thái). Nghĩa là khi một user trong LAN truy cập Internet, firewall sẽ tạo một "state" (trạng thái kết nối) để cho phép gói tin phản hồi quay lại.

Chức năng: pfsync liên tục sao chép bảng trạng thái (State Table) chứa thông tin các phiên TCP/UDP, NAT sessions từ thiết bị Primary sang thiết bị Secondary theo thời gian thực.

Mục đích (Tại sao cần?): Để đạt được Failover không gián đoạn (Seamless Failover). Nếu Primary chết, Secondary lên thay và tiếp quản IP CARP. Nhờ có pfsync, Secondary đã biết sẵn mọi kết nối đang diễn ra. Các luồng download, gọi video, hay VPN của người dùng sẽ tiếp tục chạy bình thường mà không bị rớt (không cần kết nối lại).

Giao thức hoạt động: Hoạt động ở Layer 4, sử dụng giao thức pfsync (Protocol 240) gửi qua địa chỉ Multicast hoặc Unicast.
* Đồng bộ cấu hình (XMLRPC Sync - Configuration Synchronization).
XMLRPC Sync (Đồng bộ cấu hình)
Chức năng: Tự động đẩy các thay đổi về mặt cấu hình (Configuration) từ thiết bị Primary sang Secondary. Các thành phần đồng bộ bao gồm: Firewall Rules, NAT, Aliases, VPN, Certificates, DHCP, v.v.

Mục đích (Tại sao cần?): Đảm bảo hai thiết bị luôn nhất quán về chính sách. Nếu bạn tạo một rule chặn IP trên firewall chính, rule đó tự động xuất hiện trên firewall phụ. Điều này giúp bạn chỉ cần cấu hình trên một máy, tiết kiệm thời gian và tránh rủi ro quên cấu hình máy dự phòng dẫn đến lọt lỗ hổng bảo mật khi xảy ra failover.

Giao thức hoạt động: Hoạt động ở tầng Ứng dụng (Application Layer), đẩy dữ liệu qua giao thức HTTPS (Port 443 hoặc port WebGUI) bằng XMLRPC.

* 1.4. Khái niệm Virtual IP (VIP) và vai trò trong việc định tuyến mượt mà.

Virtual IP (VIP) là một địa chỉ IP không gắn chết vào một cổng mạng vật lý (Physical Interface) cụ thể nào một cách vĩnh viễn. Thay vì thuộc về một thiết bị phần cứng duy nhất, VIP được "chia sẻ" trên một cụm (cluster) gồm nhiều thiết bị (ví dụ: hai firewall pfSense chạy HA).

Vấn đề: Nếu các máy trạm (client) trong mạng nội bộ trỏ Default Gateway về IP vật lý của Firewall 1 (ví dụ 172.16.1.10). Khi Firewall 1 sập, mạng sẽ mất hoàn toàn cho đến khi quản trị viên đi cấu hình lại Gateway của từng máy trạm sang IP của Firewall 2 ( 172.16.1.11).

Giải pháp với VIP: Bạn tạo một LAN VIP (ví dụ  172.16.1.12) và cấp phát IP này làm Default Gateway cho toàn bộ mạng nội bộ. Khi Firewall 1 sập, Firewall 2 lập tức tiếp quản địa chỉ  172.16.1.12 Toàn bộ luồng dữ liệu tự động chảy qua Firewall 2 mà các máy tính của người dùng không hề nhận ra sự thay đổi ở tầng Gateway.


**Phần 2: Chuẩn bị mô hình mạng và Quy hoạch IP**

* 2.1. Yêu cầu về phần cứng và hạ tầng
  
  Có tối thiểu 3 IP mỗi subnet trên các cổng mạng của pfsense (1 IP cho Master, 1 IP cho Backup và 1 Virtual IP chính dùng để giao tiếp với các thiết bị khác)
Các thiết bị layer 2 phải cho phép multicast
Upstream/ISP/các routers có liên quan phải có khả năng gọi được các virtual IP được sử dụng bởi CARP

* 2.2. Xây dựng sơ đồ kết nối (Network Topology):
* 
* <img width="888" height="880" alt="image" src="https://github.com/user-attachments/assets/d3517e30-21a0-41ee-89ff-0c3a43788ff1" />

Với:
* Cổng WAN (Kết nối Internet) - Master
    - IP wan: 192.168.197.183/24
    - IP lan: 172.16.1.10/24
- Backup
    - IP wan: 192.168.197.184/24
    - IP lan: 172.16.1.11/24
- PC
    - IP: 172.16.1.21/24
  
CARP VIP IP: 172.16.1.12 (gateway)
CARP VIP IP: 192.168.197.186




**Phần 3: Cấu hình cơ sở trên cả hai Node (Primary & Secondary)**

Ở phần cấu hình đã set up dải mạng và cấu hình cơ bản cho node Primary, xem tại báo cáo : 
<a href="https://github.com/NguyenTienDong0410/pfsense/blob/main/pfsense_caidat_cauhinh.md">Cài đặt và cấu hình pfsense</a>
tiếp đến cũng setup cấu hình của secondary tương tự với dải mạng như trên 

* 3.1. Thiết lập địa chỉ IP tĩnh cho các cổng giao tiếp (WAN, LAN).
  
* 3.2. Cấu hình Firewall Rules trên cổng SYNC (Cho phép toàn bộ traffic IPv4 qua giao thức pfsync).
  
* 3.3. Kiểm tra kết nối cơ bản giữa 2 Node (Ping chéo qua dải IP SYNC).

**Phần 4: Thiết lập High Availability (Thực hiện trên Node Primary)**

* 4.1. Tạo CARP Virtual IPs (VIPs):
  
* Cấu hình VIP cho giao diện WAN.
  
<img width="1559" height="780" alt="image" src="https://github.com/user-attachments/assets/a4c927c9-b633-4f3d-bea1-949401b90cc2" />

* Cấu hình VIP cho giao diện LAN.
  
<img width="1458" height="731" alt="image" src="https://github.com/user-attachments/assets/2022442e-b8ac-44c6-80d9-9ffd3c7a6db9" />

* Thiết lập Base vhid và Passwords cho CARP.

<img width="1498" height="487" alt="image" src="https://github.com/user-attachments/assets/fb413c0f-38f9-4071-a46e-ca7ad486ec09" />

vào máy Backup(secondary) , có thể thấy các ip được đồng bộ và tự động chỉnh skew độ ưu tiên thành 100, khác với máy master là skew = 0

<img width="1532" height="736" alt="image" src="https://github.com/user-attachments/assets/65838311-0d38-4dba-9c49-9d4374461e03" />

* 4.2. Cấu hình Outbound NAT để sử dụng CARP VIP:
* Chuyển sang chế độ Manual Outbound NAT hoặc Hybrid.

<img width="1520" height="674" alt="image" src="https://github.com/user-attachments/assets/2e1a7815-dd57-4c2d-a136-44a0d8fe6945" />

* Chỉnh sửa các luật NAT tự động để gán Translation IP thành VIP WAN.
<img width="1488" height="424" alt="image" src="https://github.com/user-attachments/assets/d9fb6664-c238-4ff9-b686-b74deb0ff490" />


* 4.3. Cấu hình DHCP Server cho mạng nội bộ:
* Gán Default Gateway và DNS Server trỏ về địa chỉ VIP LAN.
* Cấu hình Failover Peer IP để đồng bộ trạng thái thuê DHCP (DHCP Leases) sang Node 2.

<img width="1334" height="665" alt="image" src="https://github.com/user-attachments/assets/cf0ad4c5-89c1-4359-a471-6d10ff21323f" />


**Phần 5: Kích hoạt đồng bộ hóa (Synchronization)**

* 5.1. Thiết lập pfsync (Đồng bộ trạng thái - State Sync):
* Kích hoạt pfsync trên cổng giao tiếp SYNC.
* Khai báo IP tĩnh của Peer Node.
  
<img width="1483" height="707" alt="image" src="https://github.com/user-attachments/assets/80516a9d-bde5-4364-9ee5-9b0673066324" />


* 5.2. Thiết lập XMLRPC Sync (Đồng bộ cấu hình - Config Sync):
* Nhập IP cổng SYNC của Node Secondary.
* Nhập Admin Username và Password của Node Secondary.
* Lựa chọn các thành phần cấu hình cần đồng bộ (Firewall rules, NAT, Virtual IPs, Aliases,...).
<img width="1514" height="651" alt="image" src="https://github.com/user-attachments/assets/8b91bc88-6a0f-445b-af9d-17211a8b339c" />

<img width="1500" height="797" alt="image" src="https://github.com/user-attachments/assets/f1a273d2-4684-4862-b442-91baa501ad65" />


* 5.3. Kiểm tra quá trình đẩy cấu hình từ Primary sang Secondary (Trigger Sync).
Test tạo user, ngay lập tức được đẩy qua Secondary
<img width="1500" height="797" alt="image" src="https://github.com/user-attachments/assets/0407cf64-307d-4324-aa94-d201de323c85" />

**Phần 6: Kiểm thử và Đánh giá hệ thống (Failover Testing)**

* 6.1. Kiểm tra trạng thái CARP trên cả 2 Node (Status > CARP) để xác nhận vai trò MASTER / BACKUP.
  <img width="1130" height="753" alt="image" src="https://github.com/user-attachments/assets/7af41380-c96b-434e-854c-77b0e3661853" />

  <img width="1545" height="734" alt="image" src="https://github.com/user-attachments/assets/57ab28c1-1846-4a61-923a-55bd7d784161" />


* 6.2. Kịch bản test 1: Primary Node mất điện hoặc treo phần cứng (tắt con pfsense master).

  
* 6.4. Đánh giá thời gian chuyển đổi (Downtime) và tính liên tục của kết nối (Ping test liên tục) ping tới google.com liên tục.

  <img width="669" height="711" alt="image" src="https://github.com/user-attachments/assets/b6ba3fc4-c3ea-45b5-ab22-c5dee36e10fc" />
  ngay sau khi tắt tín hiệu đã đứt đoạn nhưng chỉ khoảng 1 giây , con  pfsensebackup đã lên thay làm máy master 

<img width="1565" height="789" alt="image" src="https://github.com/user-attachments/assets/2bd12196-9a0b-4bfd-8ed8-7106b58f49fd" />

* 6.5. Kịch bản khôi phục (Failback): Đưa Primary Node hoạt động trở lại và thu hồi quyền MASTER ngay sau khi được bật trở lại.

<img width="1553" height="725" alt="image" src="https://github.com/user-attachments/assets/8739f4c2-76d5-4a8e-81af-2d336f2a5e0b" />


**Phần 7: Vận hành bảo trì và Xử lý sự cố thường gặp (Troubleshooting)**

* 7.1. Chế độ bảo trì an toàn (Enter Persistent CARP Maintenance Mode) để cập nhật phần mềm.
  
* 7.2. Xử lý lỗi "Split-Brain" (Cả hai Node đều hiểu mình là MASTER).
  
* Lỗi này xảy ra khi Node Backup không nhận được nhịp tim từ Node Primary qua cổng WAN. Vì không nghe thấy gì, nó tưởng Node Primary đã "chết" nên tự động nâng quyền của nó lên làm MASTER. Hậu quả là cả 2 con đều tranh nhau một địa chỉ IP ảo.
* <img width="1546" height="735" alt="image" src="https://github.com/user-attachments/assets/4e137105-7fd8-4e0f-9f3e-11e7ab4cf1e3" />

* 7.3. Cách xử lý lỗi đồng bộ XMLRPC thất bại (Sai mật khẩu, kẹt port, v.v.).
* 7.4. Lưu ý đặc biệt đối với môi trường ảo hóa (Promiscuous mode, MAC Address changes trên VMware/Proxmox).

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/b81a3223-8136-4a90-a56a-cb79199a86fd" />





