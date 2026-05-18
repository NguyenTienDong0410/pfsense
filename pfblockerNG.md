

### Giai đoạn 1: Cài đặt gói pfBlockerNG

1. Trên giao diện Web của pfSense, vào **System > Package Manager > Available Packages**.
2. Ô tìm kiếm (Search term), bạn gõ `pfBlockerNG`.
3. Bạn sẽ thấy 2 bản. **Bắt buộc chọn bản `pfBlockerNG-devel**` (Bản devel hiện nay là bản được cộng đồng cập nhật liên tục và ổn định nhất, bản thường đã khá cũ).
4. Bấm **Install > Confirm** và chờ hệ thống chạy chữ xanh lá cây thông báo thành công.

---

### Giai đoạn 2: Chạy Wizard (Cấu hình tự động ban đầu)

Ngay sau khi cài xong, hệ thống cần thiết lập các luồng chặn cơ bản.

1. Vào **Firewall > pfBlockerNG**. Một bảng thông báo Wizard sẽ hiện ra. Bạn bấm **Next**.
2. **Component Configuration:** Màn hình giải thích tính năng, tiếp tục bấm **Next**.
3. **Interface Configuration:** * **Inbound Interface:** Chọn **WAN** (Để chặn các IP rác từ Internet rò quét vào mặt tiền).
* **Outbound Interface:** Chọn **LAN** (Để chặn nhân viên kết nối ra các IP độc hại).
* Bấm **Next**.


4. **VIP Configuration:** pfBlockerNG cần một IP ảo (độc lập với mạng LAN) để làm trang hiển thị cảnh báo "Bạn đã bị chặn". Bạn cứ để mặc định là `10.10.10.1` (miễn là nó không trùng với dải LAN `10.10.20.x` của bạn). Bấm **Next** rồi **Finish**.

Hệ thống sẽ chạy một lúc để tải các danh sách IP rác mặc định về. Bạn chờ màn hình log chạy xong rồi tiếp tục.

---

### Giai đoạn 3: Kịch bản A - Chặn Facebook bằng DNSBL (DNS Blackholing)

Đây là tính năng bắt các truy vấn tên miền (DNS) của nhân viên và ném vào "hố đen" nếu nó nằm trong danh sách cấm.

1. Vào **Firewall > pfBlockerNG > DNSBL**.
2. Đảm bảo ô **DNSBL** đã được tích xanh (Enable).
3. Cuộn xuống dưới cùng, tìm mục **DNSBL Custom_List**. Đây là nơi bạn nhập tay các trang web muốn cấm.
4. Nhập vào ô trống các tên miền sau (mỗi tên miền 1 dòng):
```text
facebook.com
.facebook.com

```


*(Lưu ý: Dấu chấm ở trước `.facebook.com` có nghĩa là chặn tất cả các tên miền con như m.facebook.com, [www.facebook.com](https://www.google.com/search?q=https://www.facebook.com)...)*
5. Cuộn xuống dưới cùng và bấm **Save**.

---

### Giai đoạn 4: Kịch bản B - Lọc theo khu vực địa lý (GeoIP)

Tính năng này cực kỳ hữu ích để chặn các cuộc tấn công tự động từ các quốc gia lạ (ví dụ chặn toàn bộ IP từ Nga hay Trung Quốc vào máy chủ Web của bạn).

1. Vào **Firewall > pfBlockerNG > IP > GeoIP**.
2. *Lưu ý:* Lần đầu tiên vào đây, pfSense sẽ yêu cầu bạn nhập **MaxMind License Key**. (MaxMind là công ty cung cấp cơ sở dữ liệu IP toàn cầu). Đăng ký tài khoản MaxMind là hoàn toàn miễn phí, nhưng để demo nhanh lúc này, chúng ta có thể làm sau.
3. Nếu bạn đã có Key và nhập vào mục `IP > MaxMind`, bạn sẽ thấy danh sách các châu lục (Asia, Europe...).
4. Bạn chỉ cần bấm vào hình cái bút chì ở dòng **Top Spammers** hoặc **Asia**, chọn Action là **Deny Both** (Chặn cả luồng ra và luồng vào), sau đó tích chọn các quốc gia bạn ghét. Bấm **Save**.

---

### Giai đoạn 5: Ép cập nhật (BƯỚC QUAN TRỌNG NHẤT)

Bất cứ khi nào bạn thêm tên miền hay đổi luật trong pfBlockerNG, nó sẽ KHÔNG áp dụng ngay. Bạn phải bắt nó tải lại hệ thống.

1. Vào **Firewall > pfBlockerNG > Update**.
2. Tích chọn các ô sau:
* **Force** (Ép buộc)
* **Reload** (Tải lại)
* **All** (Tất cả)


3. Bấm nút **Run** ở dưới. Màn hình đen (Log) sẽ hiện ra. Bạn kiên nhẫn chờ đến khi nó báo dòng chữ **"UPDATE PROCESS ENDED"** là xong.

---

### Giai đoạn 6: Demo thành quả trên máy Client (Windows Server)

Bây giờ, hãy mở máy ảo Windows Server của bạn (nằm trong dải mạng LAN) lên để test.

**Lưu ý bắt buộc:** Máy Windows Server của bạn phải đang nhận IP của pfSense (`10.10.20.1`) làm **DNS Server**, nếu không tính năng chặn DNS sẽ vô tác dụng. (Kiểm tra trong Card mạng phần IPv4).

1. Mở CMD trên Windows Server, gõ lệnh sau để xóa bộ nhớ tạm DNS (tránh trường hợp máy vẫn nhớ IP cũ của Facebook):
`ipconfig /flushdns`
2. Thử Ping: Bạn gõ `ping facebook.com`.
👉 **Kết quả mong đợi:** Thay vì trả về IP thật của Facebook (như 157.240.x.x), nó sẽ trả về cái IP ảo `10.10.10.1` mà bạn đã thiết lập ở Giai đoạn 2.
3. Mở trình duyệt Web (Edge/Chrome) gõ `http://facebook.com`.
👉 **Kết quả mong đợi:** Bạn sẽ không vào được Facebook, trình duyệt có thể báo lỗi bảo mật (do sai chứng chỉ HTTPS) hoặc hiển thị thẳng một trang trắng có logo pfSense ghi dòng chữ thông báo: *"Website này đã bị chặn bởi quản trị viên mạng"*.

Bạn hãy thử làm theo các bước trên nhé, cảm giác tự tay "cấm cửa" được một ông lớn như Facebook từ cấp độ tường lửa cực kỳ thú vị đấy!
