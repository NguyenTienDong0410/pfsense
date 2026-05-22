Triển khai High Availability (HA) - CARP trên pfSense
---

### Mục lục: Triển khai High Availability (HA) - CARP trên pfSense

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
(<a href "https://github.com/NguyenTienDong0410/pfsense/blob/main/pfsense_caidat_cauhinh.md">  Cài đặt cấu hình pfsense </a>)

* 3.1. Thiết lập địa chỉ IP tĩnh cho các cổng giao tiếp (WAN, LAN).
* 
* 3.2. Cấu hình Firewall Rules trên cổng SYNC (Cho phép toàn bộ traffic IPv4 qua giao thức pfsync).
* 
* 3.3. Kiểm tra kết nối cơ bản giữa 2 Node (Ping chéo qua dải IP SYNC).

**Phần 4: Thiết lập High Availability (Thực hiện trên Node Primary)**

* 4.1. Tạo CARP Virtual IPs (VIPs):
* Cấu hình VIP cho giao diện WAN.
* Cấu hình VIP cho giao diện LAN.
* Thiết lập Base vhid và Passwords cho CARP.


* 4.2. Cấu hình Outbound NAT để sử dụng CARP VIP:
* Chuyển sang chế độ Manual Outbound NAT hoặc Hybrid.
* Chỉnh sửa các luật NAT tự động để gán Translation IP thành VIP WAN.


* 4.3. Cấu hình DHCP Server cho mạng nội bộ:
* Gán Default Gateway và DNS Server trỏ về địa chỉ VIP LAN.
* Cấu hình Failover Peer IP để đồng bộ trạng thái thuê DHCP (DHCP Leases) sang Node 2.



**Phần 5: Kích hoạt đồng bộ hóa (Synchronization)**

* 5.1. Thiết lập pfsync (Đồng bộ trạng thái - State Sync):
* Kích hoạt pfsync trên cổng giao tiếp SYNC.
* Khai báo IP tĩnh của Peer Node.


* 5.2. Thiết lập XMLRPC Sync (Đồng bộ cấu hình - Config Sync):
* Nhập IP cổng SYNC của Node Secondary.
* Nhập Admin Username và Password của Node Secondary.
* Lựa chọn các thành phần cấu hình cần đồng bộ (Firewall rules, NAT, Virtual IPs, Aliases,...).


* 5.3. Kiểm tra quá trình đẩy cấu hình từ Primary sang Secondary (Trigger Sync).

**Phần 6: Kiểm thử và Đánh giá hệ thống (Failover Testing)**

* 6.1. Kiểm tra trạng thái CARP trên cả 2 Node (Status > CARP) để xác nhận vai trò MASTER / BACKUP.
* 6.2. Kịch bản test 1: Primary Node mất điện hoặc treo phần cứng.
* 6.3. Kịch bản test 2: Đứt cáp mạng WAN hoặc LAN trên Node Primary.
* 6.4. Đánh giá thời gian chuyển đổi (Downtime) và tính liên tục của kết nối (Ping test liên tục).
* 6.5. Kịch bản khôi phục (Failback): Đưa Primary Node hoạt động trở lại và thu hồi quyền MASTER.

**Phần 7: Vận hành bảo trì và Xử lý sự cố thường gặp (Troubleshooting)**

* 7.1. Chế độ bảo trì an toàn (Enter Persistent CARP Maintenance Mode) để cập nhật phần mềm.
* 7.2. Xử lý lỗi "Split-Brain" (Cả hai Node đều hiểu mình là MASTER).
* 7.3. Cách xử lý lỗi đồng bộ XMLRPC thất bại (Sai mật khẩu, kẹt port, v.v.).
* 7.4. Lưu ý đặc biệt đối với môi trường ảo hóa (Promiscuous mode, MAC Address changes trên VMware/Proxmox).

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/b81a3223-8136-4a90-a56a-cb79199a86fd" />




2 con pfsense
<img width="1920" height="1029" alt="image" src="https://github.com/user-attachments/assets/e121dab6-44c0-4dde-9643-c06792cf6b17" />

cấu hình HA trên máy active
<img width="1920" height="1029" alt="image" src="https://github.com/user-attachments/assets/56624622-c789-4f3a-b570-f4c49709cc06" />

<img width="1920" height="1029" alt="image" src="https://github.com/user-attachments/assets/733c766d-5d29-4996-887e-ae21f3fb0e22" />

<img width="1920" height="1029" alt="image" src="https://github.com/user-attachments/assets/bcc6d69d-93c9-470a-a431-bb4f2044bd31" />

cả 2 máy đã đồng bộ, 
đặt viture ip trên con fw master
ip ảo mạng wan
<img width="1920" height="1029" alt="image" src="https://github.com/user-attachments/assets/dd6513c2-6b6a-46d5-ba17-e9e3f01464f4" />

ip ảo mạng lan
<img width="1920" height="1029" alt="image" src="https://github.com/user-attachments/assets/87565ce5-a6e2-456b-a10b-815d6b1cbab3" />

đã đồng bộ 
<img width="1920" height="1029" alt="image" src="https://github.com/user-attachments/assets/3eff32e4-39cc-43a7-b378-7d023d4583e2" />

<img width="1920" height="1029" alt="image" src="https://github.com/user-attachments/assets/356ecadc-e4aa-4ea6-86f9-a3a3c0788673" />

có thể thấy mức độ ưu tiên là khác nhau
 ở máy backup tự động độ ưu tiên tăng lên skew 100 so với skew 0 của máy master
 
 <img width="1920" height="1029" alt="image" src="https://github.com/user-attachments/assets/276c11e2-ad5f-42df-bb3e-10bf974501d1" />

thêm wget, có thể thấy đã nhận diện fw1 là master và fw2 là backup
 <img width="1920" height="1029" alt="image" src="https://github.com/user-attachments/assets/6205abb7-c79d-41f3-9d90-f7fc77c0592a" />

ở đây, khi đặt gateway cần đặt ip ảo vì khi fw1 chết, gateway nằm ở bên fw1 các gói tin vẫn sẽ đi qua fw1 và không thể ra ngoài internet, vì vậy, cần gateway là ip ảo như trên và để các client trỏ hết về
 ip ảo này, để khi fw1 down, các gói tin có thể định tuyến về là fw2 để đi ra ngoài internet
 <img width="1920" height="1029" alt="image" src="https://github.com/user-attachments/assets/c17f3437-6e24-4039-a60e-4d94bf153097" />


còn về failover peer ip , khi 
Việc đặt Failover peer IP trên con Master trỏ về IP của con Backup (và ngược lại trên con Backup trỏ về Master) sử dụng giao thức DHCP Failover nhằm 3 mục đích cốt lõi sau trong hệ thống tính sẵn sàng cao (HA):

1. Đồng bộ "Sổ hộ khẩu" cấp phát IP (Lease Database Synchronization)
Khi một máy client (ví dụ: Windows Server hoặc PC) gửi yêu cầu xin IP, con Master sẽ cấp cho máy đó một địa chỉ, chẳng hạn là 10.10.20.15.

Ngay lập tức, con Master sẽ gửi một thông điệp qua địa chỉ Failover peer IP để báo cho con Backup biết: "Tôi vừa cấp IP 10.10.20.15 cho máy có địa chỉ MAC này rồi nhé, ông ghi vào sổ đi".

Con Backup sẽ cập nhật vào cơ sở dữ liệu của nó. Nhờ vậy, hai con luôn có một "cuốn sổ cái" giống hệt nhau về tình trạng các IP đang bận hay đang rảnh trong mạng.

2. Tránh tuyệt đối xung đột IP (IP Conflict Prevention)
Nếu không có dòng Failover peer IP để hai con thiết lập kênh nói chuyện riêng, hệ thống sẽ rơi vào trạng thái "mù thông tin":

Con Master tự cấp dải .10 đến .50.

Con Backup không biết Master đang làm gì, cũng tự ý cấp dải .10 đến .50.

Hậu quả là Master có thể cấp IP .15 cho máy A, trong khi Backup lại cấp đúng IP .15 đó cho máy B. Mạng LAN sẽ bị xung đột IP và tê liệt ngay lập tức.

Khi cấu hình Peer IP, hai con sẽ tự động chia nhau quyền quản lý dải IP (thường theo tỷ lệ 50/50 hoặc con Backup sẽ nhường toàn bộ cho Master và chỉ đứng nhìn) để không bao giờ cấp trùng IP.

3. Làm kênh đo nhịp tim (Heartbeat / Keepalive)
Địa chỉ Peer IP này đóng vai trò như một sợi dây liên lạc sinh tử giữa hai dịch vụ DHCP:

Con Backup sẽ liên tục gửi các gói tin kiểm tra (Keepalive) tới Peer IP của con Master để xem Master còn sống không.

Nếu con Master đột ngột bị sập (mất điện, lỗi phần cứng), con Backup sẽ thấy Peer IP của Master không còn phản hồi nữa. Nó sẽ lập tức kích hoạt trạng thái "Partner Down" (Đối tác đã sập) và một mình đứng ra thâu tóm toàn bộ dải IP để cấp phát tiếp cho các máy con mà không sợ đụng hàng với ai.

Tóm lại một câu ngắn gọn: Cấu hình Failover peer IP là để hai dịch vụ DHCP trên hai con tường lửa "bắt tay" nhau, tạo thành một thể thống nhất, giúp chia sẻ thông tin và thay thế nhau một cách an toàn mà không gây loạn mạng.

test
thử tắt con active, hiện requestime out và lập tức ping tới google.com ổn định trở lại 
<img width="1920" height="1029" alt="image" src="https://github.com/user-attachments/assets/c9fda941-090e-4e97-85f3-2eb7e8bf6f1f" />

<img width="1920" height="1029" alt="image" src="https://github.com/user-attachments/assets/a2c61353-8e04-457c-b075-f102baef7a92" />
con pfsense đã chết , và được chuyển quyền master sang cho con backup

nhưng khi con pfsense active 1 sống lại thì sao ?
<img width="1920" height="1029" alt="image" src="https://github.com/user-attachments/assets/338c05cf-54b5-49e7-8545-fbbc9505b5f6" />
nó sẽ được nhận lại quyền và con pfsense backup đã trở lại thành backup
