# Triển khai và Quản trị Hệ thống Lọc mạng với pfBlockerNG

**Phần 1: Tổng quan về pfBlockerNG và Kiến trúc hoạt động**

* 1.1. pfBlockerNG là gì? Vai trò của pfBlockerNG trong hệ thống Firewall.
* 1.2. Phân biệt hai thành phần lõi:
* IP Filtering (Lọc theo dải IP và Vị trí địa lý - GeoIP).
* DNSBL - DNS Blackhole (Lọc theo tên miền để chặn quảng cáo, malware).


* 1.3. Tại sao nên sử dụng phiên bản `pfBlockerNG-devel` thay vì bản tiêu chuẩn?

**Phần 2: Chuẩn bị và Cài đặt**

* 2.1. Yêu cầu về tài nguyên phần cứng (CPU, RAM) khi chạy cơ sở dữ liệu lớn.
* 2.2. Gỡ cài đặt bản cũ an toàn (Lưu ý mục Keep Settings).
* 2.3. Quy trình cài đặt package `pfBlockerNG-devel` từ giao diện pfSense.
* 2.4. Khởi chạy pfBlockerNG Setup Wizard (Cấu hình tự động ban đầu).

**Phần 3: Cấu hình tính năng Lọc IP (IP Filtering) và GeoIP**

* 3.1. Đăng ký và thiết lập MaxMind License Key:
* Tạo tài khoản MaxMind miễn phí.
* Tích hợp License Key vào tab IP để kích hoạt dữ liệu GeoIP.


* 3.2. Cấu hình GeoIP (Lọc theo khu vực địa lý):
* Phân tích các Action: Deny Inbound, Deny Outbound, Permit/Whitelist.
* Lựa chọn chặn theo toàn châu lục hoặc từng quốc gia cụ thể.


* 3.3. Xây dựng IPv4/IPv6 Source Definitions (Danh sách chặn/thả tự động):
* Tích hợp các Feeds bên ngoài (Spamhaus, dshield,...).
* Cấu hình Custom List cho các IP nội bộ hoặc IP lẻ.



**Phần 4: Cấu hình tính năng lọc tên miền (DNSBL)**

* 4.1. Cơ chế hoạt động của DNSBL (Chuyển hướng truy cập xấu về Virtual IP - Sinkhole).
* 4.2. Cấu hình DNSBL Category và Feeds:
* Thêm danh sách chặn quảng cáo (Adaway, Easylist).
* Thêm danh sách chặn tên miền độc hại (Malware, Phishing).


* 4.3. Thiết lập DNSBL Whitelist (Loại trừ các tên miền hợp lệ bị nhận diện nhầm).
* 4.4. Kích hoạt tính năng SafeSearch (Bảo vệ tìm kiếm an toàn trên Google, YouTube, Bing).

**Phần 5: Xử lý pfBlockerNG trong môi trường có Proxy/CDN (Nâng cao)**

* 5.1. Bản chất sự cố khi kết hợp pfSense với Reverse Proxy (Cloudflare, HAProxy).
* 5.2. Kịch bản giải quyết 1: Đẩy tác vụ chặn GeoIP lên tầng WAF (Cloudflare).
* 5.3. Kịch bản giải quyết 2: Khóa chặt máy chủ gốc (Origin Server):
* Sử dụng IPv4 Source Definitions tải danh sách IP động của Cloudflare.
* Tạo luật Permit riêng cho cụm IP này và Drop mọi truy cập trực tiếp khác.



**Phần 6: Vận hành, Theo dõi Logs và Xử lý sự cố (Troubleshooting)**

* 6.1. Nguyên tắc áp dụng luật: Tính năng Force Reload / Update (Update -> Cron / Force).
* 6.2. Theo dõi và phân tích Logs trên tab Alerts:
* Xem chi tiết IP/Domain bị chặn, cổng giao tiếp và quốc gia nguồn.
* Tùy chỉnh bật/tắt Logging để tối ưu tài nguyên hệ thống pfSense.


* 6.3. Xử lý hiện tượng False Positives (Chặn nhầm IP hợp lệ của dịch vụ).
* 6.4. Backup và Restore cấu hình pfBlockerNG an toàn.
## 1. Phương pháp chặn truy cập truyền thống (Firewall Rule/Alias) và các hạn chế

Trong giai đoạn đầu triển khai quản lý truy cập mạng, phương pháp cơ bản nhất được áp dụng là sử dụng Firewall Rule mặc định của pfSense kết hợp với tính năng Alias.

**Cách thức hoạt động:**
Quản trị viên thu thập địa chỉ IP của trang web cần chặn (ví dụ: thông qua lệnh `ping` hoặc `nslookup`), hoặc khai báo tên miền (FQDN - Fully Qualified Domain Name) vào một Alias. Sau đó, tạo một Firewall Rule ở cổng LAN để rớt (Drop/Block) toàn bộ các gói tin có địa chỉ đích (Destination) trùng với Alias này.

**Tuy nhiên, qua quá trình thử nghiệm thực tế, phương pháp "chặn thủ công" ở Tầng 3 và Tầng 4 (Network/Transport Layer) này bộc lộ nhiều điểm yếu chí mạng khi đối đầu với kiến trúc Web hiện đại:**

* **Kiến trúc CDN và sự thay đổi IP liên tục:** Các hệ thống lớn như Facebook, Google không sử dụng một IP cố định. Chúng sử dụng Mạng phân phối nội dung (CDN) với hàng triệu IP thay đổi liên tục và xoay vòng theo vị trí địa lý. Việc chặn 1 hoặc vài IP thủ công là vô nghĩa vì trình duyệt sẽ tự động kết nối qua IP dự phòng khác ngay khi người dùng tải lại trang (F5).
* **Độ trễ của hệ thống Alias (FilterDNS):** Khi sử dụng Alias dạng tên miền (FQDN), pfSense dựa vào dịch vụ chạy ngầm `filterdns` để cập nhật IP định kỳ (mặc định là 5 phút/lần). Trong khi đó, các dịch vụ DNS hiện đại thay đổi IP tính bằng giây. Sự "lệch pha" này khiến Tường lửa luôn đi sau một bước, dẫn đến hiện tượng thiết bị Client vẫn lọt lưới truy cập thành công.
* **Rủi ro "Block nhầm" diện rộng (Shared IP / Reverse Proxy):** Đối với các website sử dụng Cloudflare hoặc các dịch vụ Reverse Proxy, IP phân giải được thực chất là IP của máy chủ Cloudflare. Một IP này đang phục vụ đồng thời hàng chục ngàn website khác nhau. Nếu đưa IP này vào Firewall Rule để chặn một trang web xấu, hệ thống sẽ vô tình đánh sập luôn quyền truy cập vào hàng ngàn trang web hợp pháp khác đang dùng chung hạ tầng mạng đó.

**Tiểu kết:** Phương pháp Firewall Rule truyền thống đòi hỏi quá nhiều công sức quản lý thủ công, không có khả năng tự động hóa và thiếu sự chính xác, tiềm ẩn rủi ro làm gián đoạn các dịch vụ mạng khác.

---

## 2. Giải pháp tối ưu hóa với pfBlockerNG và công nghệ DNSBL

Nhận thấy việc cố gắng "chặn ngọn" bằng địa chỉ IP đã không còn phù hợp với thực tiễn, bài toán đặt ra là phải chuyển hướng kiểm soát sang "chặn từ gốc" — kiểm soát quá trình phân giải tên miền (DNS). Đó là lý do hệ thống được nâng cấp và tích hợp gói mở rộng **pfBlockerNG**.

**pfBlockerNG** là một Next-Generation Firewall package dành cho pfSense, biến hệ thống thành một bộ lọc nội dung cấp doanh nghiệp với tính tự động hóa cao. Thay vì quản lý IP thủ công, pfBlockerNG giải quyết hoàn toàn các hạn chế cũ thông qua hai cơ chế cốt lõi:

* **Cơ chế đánh chặn DNS (DNS Blackhole - DNSBL):**
Thay vì cố gắng đuổi theo sự thay đổi IP của các dịch vụ CDN hay Cloudflare, pfBlockerNG can thiệp trực tiếp vào dịch vụ DNS Resolver nội bộ của pfSense.
Khi người dùng truy cập một trang web bị cấm (ví dụ: `facebook.com`), hệ thống DNSBL sẽ ngay lập tức trả về một địa chỉ IP ảo (Sinkhole IP, ví dụ `0.0.0.0` hoặc `10.10.10.1`). Trình duyệt của máy Client nhận IP giả nên không thể thiết lập kết nối tới máy chủ thật. Phương pháp này chặn chính xác đích danh tên miền mà **không gây ảnh hưởng đến bất kỳ website nào khác** dùng chung IP thực của Cloudflare.
* **Tự động hóa thông qua Danh sách đen (Feeds/Blacklists):**
pfBlockerNG tự động đồng bộ và tải về các danh sách IP/Tên miền đen (Feeds) từ các tổ chức an ninh mạng uy tín trên thế giới (như danh sách chặn Malware, Ads, Social Network,...). Quản trị viên chỉ cần chọn hạng mục cần chặn, hệ thống sẽ tự động cập nhật hàng ngày mà không cần thêm bớt thủ công bất kỳ Rule nào.

---

## 3. Hướng dẫn triển khai và cấu hình pfBlockerNG (DNSBL)

Để hiện thực hóa giải pháp chặn lọc bằng tên miền, quy trình triển khai pfBlockerNG trên hệ thống pfSense được thực hiện qua các giai đoạn sau:

### 3.1. Cài đặt gói mở rộng (Package Installation)

Khác với các tính năng có sẵn, pfBlockerNG là một module mở rộng cần được cài đặt từ kho lưu trữ của pfSense:

1. Truy cập vào giao diện WebGUI của pfSense.
2. Điều hướng tới **System** -> **Package Manager** -> tab **Available Packages**.
3. Tìm kiếm từ khóa `pfBlockerNG` (hoặc `pfBlockerNG-devel` để có các tính năng cập nhật mới nhất) và nhấn **Install** để tiến hành cài đặt.
<img width="1581" height="755" alt="image" src="https://github.com/user-attachments/assets/8ee509c5-3a50-4317-94a9-1cf775894467" />

<img width="1479" height="632" alt="image" src="https://github.com/user-attachments/assets/8ec86688-bc9c-4d94-8903-ef4e295da86c" />

### 3.2. Thiết lập cơ bản thông qua Trình hướng dẫn (Wizard)

Sau khi cài đặt hoàn tất, hệ thống sẽ yêu cầu cấu hình các thông số nền tảng cho pfBlockerNG:

1. Điều hướng tới **Firewall** -> **pfBlockerNG**. Màn hình Wizard sẽ tự động xuất hiện.
<img width="1506" height="776" alt="image" src="https://github.com/user-attachments/assets/3c0a23f9-5722-4289-a192-9671465bf0de" />

3. **Cấu hình Interface (Giao diện mạng):**
* Chọn cổng **WAN** cho Inbound (Traffic đi từ ngoài Internet vào).
* Chọn cổng **LAN** cho Outbound (Traffic từ mạng nội bộ đi ra).
<img width="1536" height="762" alt="image" src="https://github.com/user-attachments/assets/ec39a0e4-3ca8-4bbe-a47a-cc08b258d4f4" />


3. **Cấu hình VIP (Virtual IP) cho DNSBL:** Đây là bước quan trọng nhất tạo nên cơ chế Sinkhole.
* Cấp một địa chỉ IP ảo không nằm trong dải mạng LAN hiện tại (ví dụ: `10.10.10.1`).
* Khi DNSBL phát hiện truy cập vào tên miền bị cấm, nó sẽ điều hướng truy cập đó về địa chỉ IP ảo này thay vì IP thật của máy chủ đích.

<img width="1478" height="752" alt="image" src="https://github.com/user-attachments/assets/686a199d-4460-4d62-a874-7da1041df1d8" />

Xin lỗi bạn, tôi quên mất là từ bản **pfSense 2.7** (đi kèm với gói pfBlockerNG bản 3.x mới), giao diện và cách cấu hình đã được làm lại khá nhiều, đặc biệt là phần DNSBL. Tính năng nhập tay Custom List không còn nằm tênh hênh ở trang ngoài nữa mà bị đưa vào trong các "Group".

Đồng thời, bản `pfBlockerNG-devel` đã bị khai tử và gộp chung vào bản `pfBlockerNG` chính thức.

Dưới đây là bản cập nhật **chính xác các bước thao tác trên pfSense 2.7** để bạn đưa vào báo cáo và làm demo:

---

### 3.1. Cài đặt gói mở rộng trên pfSense 2.7

1. Vào **System** -> **Package Manager** -> tab **Available Packages**.
2. Tìm kiếm `pfBlockerNG` (ở bản 2.7, bạn chỉ cần cài bản chuẩn, không cần tìm bản `-devel` nữa vì nó đã được gộp chung). Nhấn **Install**.

### 3.2. Kích hoạt và cấu hình DNSBL nền tảng

Ở bản mới, đôi khi Wizard không tự chạy, bạn sẽ cấu hình tay rất nhanh như sau:

1. Vào **Firewall** -> **pfBlockerNG** -> tab **General**.
* Tích chọn **Enable pfBlockerNG** -> Bấm **Save**.


2. Chuyển sang tab **DNSBL**:
* Tích chọn **Enable DNSBL**.
* Ở mục *DNSBL Mode*, bạn chọn **Unbound mode** (hoặc *Python mode* nếu hệ thống mặc định bật).
* Kéo xuống dưới cùng và bấm **Save**.



### 3.3. Cấu hình danh sách chặn tên miền (Bước này khác hoàn toàn bản cũ)

Trên pfSense 2.7, để thêm các tên miền chặn thủ công (như Facebook), bạn phải tạo một "Nhóm" (Group):

1. Vẫn trong menu pfBlockerNG, vào tab **DNSBL** -> Chọn menu con **DNSBL Groups** (nằm ở thanh ngang bên dưới các tab).
2. Bấm nút **+ Add** để tạo nhóm chặn mới.
3. Điền các thông số cơ bản:
* **Name:** `Block_Custom_Web` (Không chứa dấu cách).
* **Description:** Danh sách chặn web thủ công.
* **State:** Đổi thành **ON**.
* **Action:** Chọn **Unbound** (hoặc *Null Block*).

<img width="1465" height="730" alt="image" src="https://github.com/user-attachments/assets/a0b9a5e9-09d3-4bac-8a40-69462a770e4b" />

4. Kéo thanh cuộn xuống tìm mục **DNSBL Custom_List**:
* Tại ô văn bản trống, bạn nhập danh sách các tên miền cần chặn, mỗi tên miền 1 dòng.
* *Ví dụ nhập:*
`facebook.com`
`[www.facebook.com](https://www.facebook.com)`

<img width="1492" height="246" alt="image" src="https://github.com/user-attachments/assets/9f3831d8-8157-4a48-9704-2de22f180935" />

5. Kéo xuống cuối cùng và bấm **Save**.

### 3.4. Cập nhật và áp dụng luật (Force Update)

1. Chuyển sang tab **Update** trong menu pfBlockerNG.
2. Tại mục *Select 'Force' option*, tích chọn **Force**.
3. Tại mục *Select 'Reload' option*, tích chọn **DNSBL** (hoặc *All*).
4. Bấm nút **Run** ở dưới cùng.
5. Cửa sổ Log sẽ hiện ra. Bạn đợi các dòng chữ chạy xong và báo `UPDATE PROCESS ENDED` là hệ thống đã nhận cấu hình.

<img width="1473" height="564" alt="image" src="https://github.com/user-attachments/assets/301565ff-99a9-4716-baab-fddfb75e33a0" />

### 3.5. Kiểm tra trên máy Client


1. Trên máy Windows Client: Mở CMD, gõ `ipconfig /flushdns` để xóa cache cũ.
2. Mở trình duyệt ẩn danh và vào `facebook.com`.
3. Trang web sẽ báo không thể truy cập
4. Khi sử dụng pfBlockerNG DNSBL để chặn một trang web HTTPS (như Facebook), người dùng sẽ nhận được cảnh báo lỗi chứng chỉ bảo mật (Certificate Invalid) thay vì báo mất mạng.

Lý do là hệ thống đã điều hướng tên miền về một IP ảo. Trình duyệt phát hiện IP ảo này không sở hữu chứng chỉ SSL hợp pháp của Facebook nên đã tự động ngắt kết nối. Kết hợp với chính sách bảo mật HSTS, người dùng hoàn toàn không thể vượt qua lớp rào chắn này.

Đây là hành vi kỹ thuật hoàn toàn chính xác, chứng minh pfBlockerNG đã bảo vệ hệ thống thành công từ khâu phân giải tên miền (DNS) mà không cần phải can thiệp giải mã dữ liệu (SSL Inspection) phức tạp."*
<img width="1244" height="575" alt="image" src="https://github.com/user-attachments/assets/3ba716fe-19b7-48b2-928b-6c5312b5900f" />

<img width="1552" height="409" alt="image" src="https://github.com/user-attachments/assets/0835b227-9d47-43b7-9a83-4056e86c567e" />

---

# 4. IP

Chặn theo vùng bằng GeoIP
<img width="1513" height="636" alt="image" src="https://github.com/user-attachments/assets/dca020c6-08c9-4305-8a70-30a8a4e0d8f4" />

Chặn theo IPV4
<img width="1501" height="639" alt="image" src="https://github.com/user-attachments/assets/032a3b81-db49-4ee4-a127-45f553ee1632" />

<img width="1518" height="736" alt="image" src="https://github.com/user-attachments/assets/812457d3-5639-41fe-ac3a-2bc611db8be5" />

<img width="1477" height="736" alt="image" src="https://github.com/user-attachments/assets/9363d4ca-c660-4655-b4bf-3baab898a253" />

có thể import IPv4 Source Definitions chặn theo list

Ngoài ra còn ohaanf IPv4 Custom_List để tự thêm từng IP riêng lẻ

phần cấu hình ở đây cũng như phần DNSBL đều được ghi vào Rules trong phần Firewall
