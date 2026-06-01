# Triển khai và Quản trị Hệ thống Lọc mạng với pfBlockerNG

---

## Mục lục

- [Phần 1: Tổng quan về pfBlockerNG và Kiến trúc hoạt động](#phần-1-tổng-quan-về-pfblockerng-và-kiến-trúc-hoạt-động)
  - [1.1. pfBlockerNG là gì? Vai trò trong hệ thống Firewall](#11-pfblockerng-là-gì-vai-trò-trong-hệ-thống-firewall)
  - [1.2. Phân biệt hai thành phần lõi](#12-phân-biệt-hai-thành-phần-lõi)
  - [1.3. Tại sao nên sử dụng pfBlockerNG-devel?](#13-tại-sao-nên-sử-dụng-pfblockerng-devel)
- [Phần 2: Chuẩn bị và Cài đặt](#phần-2-chuẩn-bị-và-cài-đặt)
  - [2.1. Yêu cầu tài nguyên phần cứng](#21-yêu-cầu-tài-nguyên-phần-cứng)
  - [2.2. Gỡ cài đặt bản cũ an toàn](#22-gỡ-cài-đặt-bản-cũ-an-toàn)
  - [2.3. Cài đặt package pfBlockerNG](#23-cài-đặt-package-pfblockerng)
  - [2.4. Khởi chạy Setup Wizard](#24-khởi-chạy-setup-wizard)
- [Phần 3: Cấu hình Lọc IP và GeoIP](#phần-3-cấu-hình-lọc-ip-và-geoip)
  - [3.1. Đăng ký và thiết lập MaxMind License Key](#31-đăng-ký-và-thiết-lập-maxmind-license-key)
  - [3.2. Cấu hình GeoIP — Lọc theo khu vực địa lý](#32-cấu-hình-geoip--lọc-theo-khu-vực-địa-lý)
  - [3.3. Xây dựng IPv4/IPv6 Source Definitions](#33-xây-dựng-ipv4ipv6-source-definitions)
- [Phần 4: Cấu hình Lọc tên miền — DNSBL](#phần-4-cấu-hình-lọc-tên-miền--dnsbl)
  - [4.1. Cơ chế hoạt động của DNSBL](#41-cơ-chế-hoạt-động-của-dnsbl)
  - [4.2. Cấu hình DNSBL Category và Feeds](#42-cấu-hình-dnsbl-category-và-feeds)
  - [4.3. Thiết lập DNSBL Whitelist](#43-thiết-lập-dnsbl-whitelist)
  - [4.4. Kích hoạt tính năng SafeSearch](#44-kích-hoạt-tính-năng-safesearch)
- [Phần 5: pfBlockerNG trong môi trường có Proxy/CDN (Nâng cao)](#phần-5-pfblockerng-trong-môi-trường-có-proxyCDN-nâng-cao)
  - [5.1. Bản chất sự cố khi kết hợp pfSense với Reverse Proxy](#51-bản-chất-sự-cố-khi-kết-hợp-pfsense-với-reverse-proxy)
  - [5.2. Kịch bản 1 — Đẩy tác vụ chặn GeoIP lên tầng WAF](#52-kịch-bản-1--đẩy-tác-vụ-chặn-geoip-lên-tầng-waf)
  - [5.3. Kịch bản 2 — Khóa chặt Origin Server](#53-kịch-bản-2--khóa-chặt-origin-server)
- [Phần 6: Vận hành, Theo dõi Logs và Xử lý sự cố](#phần-6-vận-hành-theo-dõi-logs-và-xử-lý-sự-cố)
  - [6.1. Force Reload / Update](#61-force-reload--update)
  - [6.2. Theo dõi và phân tích Logs trên tab Alerts](#62-theo-dõi-và-phân-tích-logs-trên-tab-alerts)
  - [6.3. Xử lý False Positives](#63-xử-lý-false-positives)
  - [6.4. Backup và Restore cấu hình](#64-backup-và-restore-cấu-hình)

---

## Phần 1: Tổng quan về pfBlockerNG và Kiến trúc hoạt động

### 1.1. pfBlockerNG là gì? Vai trò trong hệ thống Firewall

#### Bối cảnh — Hạn chế của phương pháp chặn truyền thống

Trước khi đi vào pfBlockerNG, cần hiểu tại sao phương pháp quản lý truy cập truyền thống bằng Firewall Rule và Alias thuần túy lại không còn đáp ứng được yêu cầu thực tiễn.

Cách thức hoạt động của phương pháp cũ như sau: quản trị viên thu thập địa chỉ IP của trang web cần chặn (thông qua lệnh `ping` hoặc `nslookup`), hoặc khai báo tên miền FQDN vào một Alias, rồi tạo Firewall Rule tại cổng LAN để Drop toàn bộ gói tin có địa chỉ đích trùng với Alias đó.

Qua thử nghiệm thực tế, phương pháp "chặn thủ công" ở Tầng 3 và Tầng 4 (Network/Transport Layer) bộc lộ ba điểm yếu căn bản:

**Kiến trúc CDN và sự thay đổi IP liên tục:** Các hệ thống lớn như Facebook, Google sử dụng mạng phân phối nội dung (CDN) với hàng triệu địa chỉ IP xoay vòng liên tục theo vị trí địa lý. Việc chặn thủ công một vài IP là vô nghĩa vì trình duyệt tự động kết nối qua IP dự phòng ngay khi người dùng tải lại trang.

**Độ trễ của hệ thống Alias (FilterDNS):** Khi dùng Alias dạng FQDN, pfSense dựa vào dịch vụ `filterdns` để cập nhật IP định kỳ (mặc định mỗi 5 phút). Trong khi đó, các dịch vụ DNS hiện đại thay đổi IP tính bằng giây. Sự lệch pha này khiến tường lửa luôn đi sau thực tế một bước, dẫn đến hiện tượng thiết bị Client vẫn truy cập thành công.

**Rủi ro block nhầm diện rộng (Shared IP / Reverse Proxy):** Đối với các website dùng Cloudflare hoặc Reverse Proxy, IP phân giải được thực chất là IP của máy chủ Cloudflare — đang phục vụ đồng thời hàng chục nghìn website khác. Đưa IP này vào rule chặn sẽ vô tình cắt quyền truy cập vào tất cả các trang hợp lệ dùng chung hạ tầng đó.

**Tiểu kết:** Phương pháp Firewall Rule truyền thống đòi hỏi quá nhiều công sức quản lý thủ công, không có khả năng tự động hóa và tiềm ẩn rủi ro làm gián đoạn các dịch vụ mạng hợp lệ.

#### pfBlockerNG là gì?

**pfBlockerNG** là một Next-Generation Firewall package dành cho pfSense, biến hệ thống thành bộ lọc nội dung cấp doanh nghiệp với tính tự động hóa cao. Thay vì quản lý IP thủ công, pfBlockerNG giải quyết toàn bộ các hạn chế trên thông qua hai cơ chế cốt lõi:

**Cơ chế đánh chặn DNS (DNS Blackhole — DNSBL):** Thay vì đuổi theo sự thay đổi IP của CDN hay Cloudflare, pfBlockerNG can thiệp trực tiếp vào dịch vụ DNS Resolver nội bộ của pfSense. Khi người dùng truy cập một tên miền bị cấm, DNSBL trả về một địa chỉ IP ảo (Sinkhole IP). Trình duyệt nhận IP giả, không thể thiết lập kết nối tới máy chủ thật — và điều quan trọng là không gây ảnh hưởng đến bất kỳ website nào khác dùng chung IP thực của Cloudflare.

**Tự động hóa thông qua Feeds/Blacklists:** pfBlockerNG tự động đồng bộ và tải về các danh sách IP/tên miền đen từ các tổ chức an ninh mạng uy tín trên thế giới. Quản trị viên chỉ cần chọn hạng mục cần chặn, hệ thống tự động cập nhật hàng ngày mà không cần thêm bớt rule thủ công.

---

### 1.2. Phân biệt hai thành phần lõi

pfBlockerNG hoạt động dựa trên hai module độc lập, bổ sung cho nhau:

**IP Filtering — Lọc theo dải IP và Vị trí địa lý (GeoIP)**

Module này hoạt động ở tầng mạng (Layer 3), kiểm soát lưu lượng dựa trên địa chỉ IP nguồn hoặc đích. Hai chức năng chính bao gồm:

- **GeoIP Blocking:** Chặn hoặc cho phép lưu lượng theo quốc gia hoặc châu lục, dựa trên cơ sở dữ liệu địa lý của MaxMind. Ví dụ: chặn toàn bộ traffic từ một quốc gia không có quan hệ kinh doanh, giảm thiểu tấn công từ các vùng có lịch sử phát tán mã độc cao.
- **IP Feed Blocking:** Tự động tải và áp dụng các danh sách IP đen từ nguồn uy tín như Spamhaus, dshield, Emerging Threats. Các danh sách này được cập nhật định kỳ, đảm bảo hệ thống luôn phòng thủ trước các IP tấn công mới nhất.

**DNSBL — DNS Blackhole (Lọc theo tên miền)**

Module này hoạt động ở tầng ứng dụng (Layer 7), can thiệp vào quá trình phân giải tên miền. Thay vì chặn IP, DNSBL chặn chính tên miền — giải quyết triệt để bài toán CDN và Shared IP. Ứng dụng chính bao gồm chặn quảng cáo, chặn tên miền phát tán malware/phishing, và kiểm soát truy cập mạng xã hội.

---

### 1.3. Tại sao nên sử dụng pfBlockerNG-devel?

> **Lưu ý quan trọng về phiên bản:** Từ pfSense 2.7 trở đi, gói `pfBlockerNG-devel` đã được khai tử và gộp chung vào bản `pfBlockerNG` chính thức. Toàn bộ tài liệu này áp dụng cho pfSense 2.7+ với gói pfBlockerNG bản 3.x.

Trên các phiên bản pfSense cũ hơn (2.6 trở về trước), `pfBlockerNG-devel` là lựa chọn được ưu tiên vì:

- Nhận các bản vá bảo mật và tính năng mới sớm hơn bản stable.
- Hỗ trợ Python mode cho DNSBL — hiệu năng cao hơn Unbound mode truyền thống.
- Giao diện cấu hình được cải tiến liên tục, nhiều tùy chọn chi tiết hơn.

Trên pfSense 2.7+, toàn bộ cải tiến trên đã được tích hợp vào bản chính thức. Người dùng chỉ cần cài đặt `pfBlockerNG` là đủ.

---

## Phần 2: Chuẩn bị và Cài đặt

### 2.1. Yêu cầu tài nguyên phần cứng

pfBlockerNG, đặc biệt khi kích hoạt cả IP Filtering lẫn DNSBL với nhiều Feed, đặt ra yêu cầu tài nguyên đáng kể hơn so với pfSense thuần túy:

**RAM:** Đây là tài nguyên quan trọng nhất. Cơ sở dữ liệu GeoIP của MaxMind (GeoLite2) và các bảng IP Feed khi được nạp vào bộ nhớ có thể chiếm từ 256 MB đến hơn 512 MB tùy số lượng Feed và quốc gia được chọn. Khuyến nghị tối thiểu là 2 GB RAM; với môi trường nhiều Feed và tải lớn, nên dùng từ 4 GB trở lên.

**CPU:** Quá trình Force Reload — tải xuống, phân tích và áp dụng toàn bộ danh sách — tiêu tốn CPU đáng kể trong vài phút. CPU đa nhân và tốc độ cao sẽ rút ngắn thời gian này. Trong quá trình vận hành bình thường (sau khi đã reload), tải CPU từ pfBlockerNG là không đáng kể.

**Dung lượng ổ đĩa:** Các Feed và cơ sở dữ liệu GeoIP được lưu trữ trên ổ đĩa. Cần ít nhất 2–4 GB dung lượng trống tại `/var/db/pfblockerng/`.

---

### 2.2. Gỡ cài đặt bản cũ an toàn

Nếu hệ thống đang chạy một phiên bản pfBlockerNG cũ, cần gỡ cài đặt đúng quy trình để tránh xung đột cấu hình:

1. Trước khi gỡ, vào **Firewall** -> **pfBlockerNG** -> tab **General**, tắt (Disable) pfBlockerNG và thực hiện **Force Reload** một lần để hệ thống dọn sạch các rule và bảng đã tạo.
2. Vào **System** -> **Package Manager** -> tab **Installed Packages**, tìm pfBlockerNG và nhấn **Remove**.
3. **Lưu ý mục Keep Settings:** Khi hộp thoại xác nhận xuất hiện, nếu muốn giữ lại cấu hình cũ để tái sử dụng sau khi cài bản mới, tích chọn **Keep Settings**. Nếu muốn cài đặt sạch từ đầu, bỏ tích tùy chọn này.
4. Sau khi gỡ xong, kiểm tra lại **Firewall** -> **Rules** để đảm bảo các rule tự động của pfBlockerNG đã được dọn sạch.

---

### 2.3. Cài đặt package pfBlockerNG

1. Truy cập giao diện WebGUI của pfSense.
2. Điều hướng tới **System** -> **Package Manager** -> tab **Available Packages**.
3. Tìm kiếm từ khóa `pfBlockerNG`.
4. Trên pfSense 2.7+, chỉ cần cài bản `pfBlockerNG` chuẩn (không cần tìm bản `-devel` vì đã được gộp chung). Nhấn **Install** để tiến hành.

![Màn hình Package Manager tìm kiếm pfBlockerNG](https://github.com/user-attachments/assets/8ee509c5-3a50-4317-94a9-1cf775894467)

![Xác nhận cài đặt pfBlockerNG](https://github.com/user-attachments/assets/8ec86688-bc9c-4d94-8903-ef4e295da86c)

5. Chờ quá trình cài đặt hoàn tất. Hệ thống sẽ hiển thị thông báo `Installation Successful`.

---

### 2.4. Khởi chạy Setup Wizard

Sau khi cài đặt, lần đầu truy cập **Firewall** -> **pfBlockerNG**, giao diện Wizard có thể tự động xuất hiện để hướng dẫn cấu hình ban đầu.

![Màn hình pfBlockerNG Setup Wizard](https://github.com/user-attachments/assets/3c0a23f9-5722-4289-a192-9671465bf0de)

> **Lưu ý:** Trên pfSense 2.7 với pfBlockerNG 3.x, đôi khi Wizard không tự khởi chạy. Trong trường hợp đó, cấu hình tay theo các bước tại Phần 3 và Phần 4.

**Cấu hình Interface trong Wizard:**

- Chọn cổng **WAN** cho phần Inbound (traffic đi từ ngoài Internet vào).
- Chọn cổng **LAN** cho phần Outbound (traffic từ mạng nội bộ đi ra ngoài).

![Cấu hình Interface WAN/LAN trong Wizard](https://github.com/user-attachments/assets/ec39a0e4-3ca8-4bbe-a47a-cc08b258d4f4)

**Cấu hình VIP (Virtual IP) cho DNSBL — Sinkhole:**

Đây là bước quan trọng nhất tạo nên cơ chế Sinkhole của DNSBL:

- Cấp một địa chỉ IP ảo không nằm trong dải mạng LAN hiện tại (ví dụ: `10.10.10.1`).
- Khi DNSBL phát hiện truy cập vào tên miền bị cấm, nó điều hướng truy vấn DNS về địa chỉ IP ảo này thay vì IP thật của máy chủ đích. Trình duyệt không thể kết nối, yêu cầu bị ngắt.

![Cấu hình Virtual IP cho DNSBL Sinkhole](https://github.com/user-attachments/assets/686a199d-4460-4d62-a874-7da1041df1d8)

---

## Phần 3: Cấu hình Lọc IP và GeoIP

### 3.1. Đăng ký và thiết lập MaxMind License Key

Tính năng GeoIP của pfBlockerNG phụ thuộc vào cơ sở dữ liệu **GeoLite2** của MaxMind. Đây là cơ sở dữ liệu ánh xạ địa chỉ IP với vị trí địa lý (quốc gia, thành phố). MaxMind cung cấp GeoLite2 miễn phí nhưng yêu cầu đăng ký tài khoản và sử dụng License Key.

**Bước 1 — Tạo tài khoản MaxMind miễn phí:**

1. Truy cập [https://www.maxmind.com/en/geolite2/signup](https://www.maxmind.com/en/geolite2/signup) và tạo tài khoản miễn phí.
2. Sau khi đăng nhập, vào mục **Account** -> **Manage License Keys** -> **Generate new license key**.
3. Sao chép License Key vừa tạo.

**Bước 2 — Tích hợp License Key vào pfBlockerNG:**

1. Vào **Firewall** -> **pfBlockerNG** -> tab **IP**.
2. Tìm mục **MaxMind GeoIP** hoặc **GeoIP Settings**.
3. Dán License Key vào trường tương ứng.
4. Nhấn **Save** và thực hiện **Force Reload** để hệ thống tải xuống cơ sở dữ liệu GeoLite2.

---

### 3.2. Cấu hình GeoIP — Lọc theo khu vực địa lý

Sau khi License Key được tích hợp thành công, tab **IP** -> **GeoIP** cho phép cấu hình chính sách chặn theo địa lý.

![Màn hình cấu hình GeoIP trong pfBlockerNG](https://github.com/user-attachments/assets/dca020c6-08c9-4305-8a70-30a8a4e0d8f4)

**Các Action có thể áp dụng:**

- **Deny Inbound:** Chặn tất cả kết nối đến (Inbound) từ quốc gia/châu lục được chọn. Ứng dụng điển hình: từ chối toàn bộ truy cập từ các vùng không có quan hệ kinh doanh.
- **Deny Outbound:** Chặn tất cả kết nối ra ngoài (Outbound) đến quốc gia/châu lục được chọn. Ứng dụng điển hình: ngăn máy tính nội bộ kết nối tới các máy chủ C&C (Command & Control) của mã độc đặt tại các quốc gia nguy hiểm.
- **Permit / Whitelist:** Cho phép vô điều kiện. Dùng để tạo ngoại lệ cho các IP hoặc vùng địa lý cần được đảm bảo kết nối.

**Lựa chọn phạm vi chặn:**

- **Theo châu lục:** Nhanh chóng, phù hợp khi cần chặn cả một khu vực địa lý lớn (ví dụ: toàn bộ châu Á trừ Việt Nam).
- **Theo từng quốc gia cụ thể:** Linh hoạt và chính xác hơn, giảm thiểu rủi ro block nhầm.

---

### 3.3. Xây dựng IPv4/IPv6 Source Definitions

Ngoài GeoIP, pfBlockerNG cho phép tích hợp các danh sách IP đen từ nhiều nguồn bên ngoài.

![Màn hình cấu hình IPv4 Source Definitions](https://github.com/user-attachments/assets/032a3b81-db49-4ee4-a127-45f553ee1632)

![Chi tiết cấu hình Feed trong IPv4](https://github.com/user-attachments/assets/812457d3-5639-41fe-ac3a-2bc611db8be5)

![Tùy chọn Action và cấu hình nâng cao](https://github.com/user-attachments/assets/9363d4ca-c660-4655-b4bf-3baab898a253)

**Tích hợp các Feeds bên ngoài:**

Các Feed phổ biến và uy tín bao gồm:

- **Spamhaus DROP/EDROP:** Danh sách các dải IP của tổ chức tội phạm mạng, spam chuyên nghiệp.
- **dshield:** Danh sách IP có lịch sử tấn công, được tổng hợp từ hàng nghìn firewall toàn cầu.
- **Emerging Threats:** Danh sách IP liên quan đến các mối đe dọa mới nổi, cập nhật liên tục.
- **Firehol:** Tập hợp nhiều danh sách IP đen từ nhiều nguồn khác nhau.

Mỗi Feed được cấu hình với một **Action** riêng (Deny Inbound, Deny Both,...) và tần suất cập nhật (hàng ngày, hàng giờ).

**Cấu hình Custom List cho IP riêng lẻ:**

Ngoài việc import Feed theo URL, phần **IPv4 Custom_List** cho phép quản trị viên nhập thủ công từng dải IP hoặc IP đơn lẻ cần kiểm soát — phù hợp cho các IP nội bộ đặc biệt hoặc IP lẻ được xác định qua điều tra sự cố.

> **Lưu ý:** Toàn bộ cấu hình IP Filtering (cả Feed tự động lẫn Custom List) đều được pfBlockerNG ghi thành các Rules trong **Firewall** -> **Rules**, giúp quản trị viên kiểm tra và theo dõi dễ dàng.

---

## Phần 4: Cấu hình Lọc tên miền — DNSBL

### 4.1. Cơ chế hoạt động của DNSBL

DNSBL (DNS Blackhole) là cơ chế can thiệp vào quá trình phân giải tên miền ở cấp độ DNS Resolver nội bộ của pfSense (Unbound). Quy trình hoạt động như sau:

1. Người dùng nhập tên miền vào trình duyệt (ví dụ: `facebook.com`).
2. Trình duyệt gửi truy vấn DNS đến DNS Resolver của pfSense.
3. pfBlockerNG DNSBL kiểm tra tên miền yêu cầu với danh sách đen.
4. Nếu tên miền khớp, DNSBL trả về địa chỉ **Sinkhole IP** (ví dụ: `10.10.10.1`) thay vì IP thật.
5. Trình duyệt nhận IP giả, gửi yêu cầu HTTP/HTTPS đến IP ảo — kết nối thất bại.
6. Đối với các trang HTTPS (như Facebook), trình duyệt nhận lỗi **Certificate Invalid** vì IP ảo không có chứng chỉ SSL hợp lệ. Kết hợp với chính sách HSTS, người dùng hoàn toàn không thể bypass.

**Ưu điểm then chốt so với chặn IP:** Vì can thiệp ở tầng tên miền, DNSBL không bị ảnh hưởng bởi CDN hay Shared IP. Chặn `facebook.com` chỉ ảnh hưởng đến Facebook, không tác động đến bất kỳ website nào khác dù chúng dùng chung IP Cloudflare.

---

### 4.2. Cấu hình DNSBL Category và Feeds

**Bước 1 — Kích hoạt DNSBL:**

1. Vào **Firewall** -> **pfBlockerNG** -> tab **General**.
   - Tích chọn **Enable pfBlockerNG** -> Bấm **Save**.

2. Chuyển sang tab **DNSBL**:
   - Tích chọn **Enable DNSBL**.
   - Tại mục **DNSBL Mode**, chọn **Unbound mode** (hoặc **Python mode** nếu hệ thống mặc định bật — Python mode có hiệu năng cao hơn nhưng yêu cầu nhiều RAM hơn).
   - Kéo xuống dưới cùng và bấm **Save**.

**Bước 2 — Tạo DNSBL Group và thêm danh sách chặn:**

Trên pfSense 2.7+, việc thêm tên miền chặn (kể cả nhập thủ công) phải thực hiện thông qua **DNSBL Groups**:

1. Trong menu pfBlockerNG, vào tab **DNSBL** -> chọn menu con **DNSBL Groups**.
2. Bấm **+ Add** để tạo nhóm mới.
3. Điền các thông số:
   - **Name:** `Block_Custom_Web` (không chứa dấu cách).
   - **Description:** Mô tả ngắn gọn mục đích của nhóm.
   - **State:** Chuyển thành **ON**.
   - **Action:** Chọn **Unbound** (hoặc **Null Block**).

![Màn hình tạo DNSBL Group mới](https://github.com/user-attachments/assets/a0b9a5e9-09d3-4bac-8a40-69462a770e4b)

4. Kéo xuống tìm mục **DNSBL Custom_List**. Nhập danh sách tên miền cần chặn, mỗi tên miền trên một dòng riêng. Ví dụ:

```
facebook.com
www.facebook.com
```

![Nhập tên miền vào DNSBL Custom_List](https://github.com/user-attachments/assets/9f3831d8-8157-4a48-9704-2de22f180935)

5. Kéo xuống cuối cùng và bấm **Save**.

**Thêm Feed tự động (Adaway, Easylist, Malware, Phishing):**

Ngoài Custom List, DNSBL hỗ trợ tích hợp các Feed từ URL bên ngoài, cập nhật định kỳ:

- **Chặn quảng cáo:** Adaway, EasyList, StevenBlack Hosts.
- **Chặn tên miền độc hại:** Malwarebytes, URLhaus (abuse.ch), PhishTank.
- **Chặn mạng xã hội:** Có thể tự tạo Feed từ các danh sách cộng đồng.

---

### 4.3. Thiết lập DNSBL Whitelist

Whitelist là cơ chế ngoại lệ, đảm bảo các tên miền hợp lệ bị nhận diện nhầm (False Positive) vẫn hoạt động bình thường.

1. Vào **Firewall** -> **pfBlockerNG** -> tab **DNSBL** -> **DNSBL Whitelist** (hoặc **Permit Domain**).
2. Nhập tên miền cần loại trừ. Ví dụ: nếu `api.facebook.com` được dùng cho ứng dụng nội bộ nhưng bị block theo nhóm, thêm nó vào Whitelist.

**Nguyên tắc quản lý Whitelist:**

- Whitelist có độ ưu tiên cao hơn Blacklist.
- Chỉ thêm các tên miền đã được xác minh an toàn và thực sự cần thiết.
- Ghi chú lý do whitelist để tiện kiểm toán sau này.

---

### 4.4. Kích hoạt tính năng SafeSearch

SafeSearch là tính năng buộc các công cụ tìm kiếm lớn (Google, Bing, YouTube, DuckDuckGo) trả về kết quả đã được lọc nội dung người lớn, không cần cấu hình từng thiết bị đầu cuối.

**Cơ chế:** pfBlockerNG ghi đè kết quả DNS của các tên miền tìm kiếm (như `www.google.com`) về các endpoint SafeSearch tương ứng (ví dụ `forcesafesearch.google.com`). Mọi truy vấn từ mạng nội bộ đến Google sẽ tự động bật chế độ SafeSearch mà người dùng không thể tắt.

**Kích hoạt:**

1. Vào tab **DNSBL** -> tìm mục **SafeSearch**.
2. Tích chọn **Enable SafeSearch**.
3. Chọn các dịch vụ cần áp dụng (Google, YouTube, Bing, DuckDuckGo,...).
4. Bấm **Save** và thực hiện **Force Reload DNSBL**.

---

## Phần 5: pfBlockerNG trong môi trường có Proxy/CDN (Nâng cao)

### 5.1. Bản chất sự cố khi kết hợp pfSense với Reverse Proxy

Khi hệ thống pfSense/pfBlockerNG được triển khai song song với một Reverse Proxy như Cloudflare hoặc HAProxy, xuất hiện một mâu thuẫn kiến trúc:

**Vấn đề với GeoIP Blocking:** Toàn bộ lưu lượng từ Internet đến máy chủ gốc (Origin Server) đều đi qua Cloudflare, có nghĩa là pfBlockerNG chỉ thấy địa chỉ IP của Cloudflare, không thấy IP thật của người dùng cuối. GeoIP Blocking vì thế không thể xác định đúng quốc gia nguồn của traffic và sẽ hoặc chặn nhầm (nếu chặn IP Cloudflare) hoặc bỏ qua (nếu Whitelist IP Cloudflare).

**Vấn đề ngược lại — lộ Origin Server:** Nếu Origin Server (máy chủ gốc đằng sau Cloudflare) không được bảo vệ riêng, kẻ tấn công có thể trực tiếp kết nối tới IP Origin mà không đi qua lớp bảo vệ của Cloudflare, vô hiệu hóa toàn bộ WAF và GeoIP Blocking.

---

### 5.2. Kịch bản 1 — Đẩy tác vụ chặn GeoIP lên tầng WAF

Giải pháp đơn giản nhất: không dùng pfBlockerNG GeoIP cho traffic đến từ Internet, mà thay vào đó cấu hình tính năng **Firewall Rules** hoặc **Bot Fight Mode** + **Country Blocking** trực tiếp tại Cloudflare dashboard.

**Ưu điểm:**
- Cloudflare nhìn thấy IP thật của người dùng cuối, nên GeoIP hoạt động chính xác.
- Chặn được tấn công ngay tại edge (trước khi traffic vào hệ thống), giảm tải cho pfSense.

**Nhược điểm:**
- Phụ thuộc vào Cloudflare (dịch vụ bên thứ ba).
- Cần plan Cloudflare phù hợp để có đầy đủ tính năng Firewall Rules.

---

### 5.3. Kịch bản 2 — Khóa chặt Origin Server

Giải pháp này dùng pfBlockerNG để đảm bảo **chỉ có IP của Cloudflare** mới được phép kết nối trực tiếp vào Origin Server, chặn toàn bộ truy cập trực tiếp từ mọi nguồn khác.

**Bước 1 — Tạo IPv4 Feed chứa danh sách IP Cloudflare:**

Cloudflare công bố công khai toàn bộ dải IP của mình tại:
- `https://www.cloudflare.com/ips-v4`
- `https://www.cloudflare.com/ips-v6`

Trong pfBlockerNG, tạo một **IPv4 Source Definition** mới với URL trỏ tới danh sách IP Cloudflare. Hệ thống sẽ tự động cập nhật danh sách này định kỳ.

**Bước 2 — Tạo luật Permit cho cụm IP Cloudflare:**

Cấu hình Action của Feed Cloudflare IP là **Permit Inbound** — cho phép vô điều kiện mọi kết nối từ các IP này.

**Bước 3 — Drop toàn bộ truy cập Inbound trực tiếp:**

Tạo một Firewall Rule ở cổng WAN với Action **Block** áp dụng cho tất cả traffic, đặt bên dưới rule Permit Cloudflare (pfSense áp dụng rule theo thứ tự từ trên xuống). Kết quả: chỉ IP Cloudflare mới qua được, mọi kết nối trực tiếp khác đều bị Drop.

---

## Phần 6: Vận hành, Theo dõi Logs và Xử lý sự cố

### 6.1. Force Reload / Update

Mọi thay đổi cấu hình trong pfBlockerNG (thêm Feed, sửa Custom List, bật/tắt Group) **không được áp dụng ngay lập tức**. Phải thực hiện **Force Reload** để hệ thống tải lại và biên dịch toàn bộ danh sách thành Rules/Aliases.

**Các bước thực hiện Force Reload:**

1. Vào tab **Update** trong menu pfBlockerNG.
2. Tại mục **Select 'Force' option**, tích chọn **Force**.
3. Tại mục **Select 'Reload' option**, chọn:
   - **DNSBL** — chỉ reload phần lọc tên miền.
   - **IP** — chỉ reload phần lọc IP.
   - **All** — reload toàn bộ (dùng sau khi thay đổi lớn hoặc cài đặt lần đầu).
4. Bấm nút **Run**.
5. Cửa sổ log hiện ra. Chờ đến khi xuất hiện dòng `UPDATE PROCESS ENDED` là hệ thống đã nhận và áp dụng cấu hình mới.

![Màn hình Force Reload pfBlockerNG](https://github.com/user-attachments/assets/301565ff-99a9-4716-baab-fddfb75e33a0)

**Tự động hóa bằng Cron:**

pfBlockerNG hỗ trợ cấu hình lịch tự động cập nhật Feed qua **Cron Job** (mặc định là 1 lần/ngày vào lúc 0h00). Quản trị viên có thể điều chỉnh tần suất này tại tab **Update** -> mục **Cron Settings**.

---

### 6.2. Theo dõi và phân tích Logs trên tab Alerts

Tab **Alerts** là trung tâm giám sát hoạt động của pfBlockerNG theo thời gian thực.

**Thông tin hiển thị trên mỗi bản ghi log:**

- **Thời gian** xảy ra sự kiện.
- **Địa chỉ IP** nguồn bị chặn.
- **Tên miền hoặc dải IP** khớp với rule.
- **Cổng giao tiếp** (Port) liên quan.
- **Quốc gia nguồn** (từ GeoIP) — giúp đánh giá nhanh mức độ nguy hiểm.
- **Feed/Group** đã kích hoạt rule chặn — giúp xác định nguồn gốc quyết định.
- **Action** đã áp dụng (Block/Deny).

**Tùy chỉnh Logging để tối ưu tài nguyên:**

Logging toàn bộ sự kiện có thể chiếm dung lượng ổ đĩa và RAM đáng kể, đặc biệt trên hệ thống nhiều traffic. Quản trị viên có thể tắt logging cho từng Feed riêng lẻ (vào cấu hình của Feed, tắt tùy chọn **Enable Logging**) để chỉ ghi log cho những nguồn quan trọng cần theo dõi.

---

### 6.3. Xử lý False Positives

False Positive là hiện tượng pfBlockerNG chặn nhầm IP hoặc tên miền hợp lệ của dịch vụ cần dùng (ví dụ: một API của dịch vụ thanh toán bị đưa vào danh sách Spamhaus vì dùng chung dải IP với một đơn vị khác từng bị spam).

**Quy trình xử lý:**

1. Phát hiện qua tab **Alerts**: Tìm bản ghi có IP/domain bị chặn và xác minh đây là dịch vụ hợp lệ.
2. **Đối với IP:** Thêm IP/dải IP đó vào **Custom_List** với Action **Permit** (ưu tiên cao hơn Deny từ Feed).
3. **Đối với tên miền:** Thêm tên miền vào **DNSBL Whitelist**.
4. Thực hiện **Force Reload** để áp dụng ngoại lệ.
5. Ghi chú lý do trong trường Description để tiện kiểm toán.

**Lưu ý khi báo cáo False Positive cho nhà cung cấp Feed:** Nếu chắc chắn đây là lỗi của Feed (IP/domain sạch bị đưa nhầm vào danh sách đen), nên liên hệ báo cáo về tổ chức quản lý Feed (Spamhaus, dshield,...) để họ cập nhật danh sách.

---

### 6.4. Backup và Restore cấu hình

**Backup toàn bộ pfBlockerNG:**

Cách đơn giản nhất là sử dụng tính năng backup cấu hình pfSense:

1. Vào **Diagnostics** -> **Backup & Restore**.
2. Bấm **Download configuration as XML**. File XML này chứa toàn bộ cấu hình pfSense bao gồm cả pfBlockerNG.

**Backup riêng cấu hình pfBlockerNG:**

pfBlockerNG lưu cấu hình của mình trong cây XML của pfSense, nhưng các Feed đã tải về và cơ sở dữ liệu GeoIP nằm ở `/var/db/pfblockerng/`. Với môi trường nhiều Feed, nên backup thêm thư mục này để tránh phải tải lại toàn bộ sau khi restore.

**Restore:**

1. Vào **Diagnostics** -> **Backup & Restore** -> tab **Restore**.
2. Upload file XML backup và chọn **Restore Configuration**.
3. Sau khi restore, vào pfBlockerNG và thực hiện **Force Reload All** để đảm bảo tất cả Feed và rule được áp dụng đúng.

> **Khuyến nghị:** Thực hiện backup cấu hình trước mỗi thay đổi lớn (thêm nhiều Feed mới, thay đổi cấu trúc GeoIP, nâng cấp phiên bản pfBlockerNG).

---

## Phụ lục: Kiểm tra hoạt động trên máy Client

Sau khi cấu hình DNSBL và Force Reload thành công, kiểm tra hiệu lực chặn như sau:

1. Trên máy Windows Client: Mở CMD, gõ lệnh `ipconfig /flushdns` để xóa cache DNS cũ.
2. Mở trình duyệt ở chế độ ẩn danh (Incognito) và truy cập tên miền đã chặn (ví dụ: `facebook.com`).
3. Kết quả mong đợi: Trình duyệt hiển thị lỗi chứng chỉ bảo mật (Certificate Invalid / NET::ERR_CERT_AUTHORITY_INVALID) thay vì trang web.

![Lỗi Certificate Invalid khi truy cập tên miền bị chặn](https://github.com/user-attachments/assets/3ba716fe-19b7-48b2-928b-6c5312b5900f)

![Kết quả kiểm tra trên trình duyệt](https://github.com/user-attachments/assets/0835b227-9d47-43b7-9a83-4056e86c567e)

**Giải thích kỹ thuật:** Hệ thống đã điều hướng tên miền về Sinkhole IP — một địa chỉ ảo không sở hữu chứng chỉ SSL hợp lệ của Facebook. Trình duyệt phát hiện sự không khớp này và tự động ngắt kết nối. Kết hợp với chính sách HSTS (HTTP Strict Transport Security) mà các trang như Facebook áp dụng, người dùng hoàn toàn không thể bypass lớp rào chắn này.

Đây là hành vi kỹ thuật hoàn toàn chính xác, chứng minh pfBlockerNG đã bảo vệ hệ thống thành công từ khâu phân giải tên miền (DNS) mà không cần can thiệp giải mã dữ liệu (SSL Inspection) phức tạp.
