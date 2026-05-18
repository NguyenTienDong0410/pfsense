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
