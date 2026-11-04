# K26CNTT2_IT108_Session16_01
# 🛠️ Session 16 - Bài tập 1: Sự cố không truy cập được hệ thống quản trị

> **Môn học:** Nhập môn Công nghệ Thông tin (IT108)  
> **Sinh viên thực hiện:** Lê Hồ Thanh Thảo  
> **MSSV:** N26DTCN030  
> **Lớp:** K26CNTT2  

---

## 📌 1. Bối cảnh & Đặt vấn đề

Nam là thực tập sinh tại **GreenLogistics**. Nam nhận nhiệm vụ truy cập trang quản trị tại địa chỉ `phong-dieu-phoi.greenlog.vn` để cập nhật lộ trình xe tải.

* **Hiện tượng:** Trình duyệt báo lỗi `DNS_PROBE_FINISHED_NXDOMAIN`.
* **Giả định của Nam:** Nam khẳng định máy chủ (Server) công ty bị sập nguồn.
* **Thực tế:** Nam đang nhập nhầm địa chỉ thành `phong-dieu-phoi.grenlog.vn` (thiếu chữ `e` trong tên miền `greenlog`).

---

## 🔍 Phần 1: Phân tích kỹ thuật & Luồng dữ liệu

### 1.1. Luồng dữ liệu DNS Lookup (Input / Output)

* **Input:** Tên miền dạng văn bản con người đọc được (ví dụ: `phong-dieu-phoi.greenlog.vn`).
* **Output:** Địa chỉ IP định danh máy chủ (ví dụ: `192.0.2.1`).

#### Sơ đồ tiến trình truy vấn DNS (DNS Resolution Process)

```text
[Trình duyệt Nam] 
       │
       ▼ (1. Kiểm tra DNS Cache địa phương: Browser -> OS -> file hosts)
[Local DNS Cache] ──(Nếu tìm thấy)──► [Trả về IP & Truy cập Server]
       │ (Không tìm thấy)
       ▼
[DNS Resolver / ISP] (Bộ phân giải DNS của nhà mạng/Cloudflare 1.1.1.1)
       │
       ├─► (2. Truy vấn) ──► [Root Name Server (.)]
       │   └─► (Trả về IP của TLD Server .vn)
       │
       ├─► (3. Truy vấn) ──► [TLD Name Server (.vn)]
       │   └─► (Trả về IP của Authoritative Server greenlog.vn)
       │
       └─► (4. Truy vấn) ──► [Authoritative Name Server (greenlog.vn)]
           └─► (Trả về IP chính xác của phong-dieu-phoi.greenlog.vn)
       │
       ▼
[Trả IP về Trình duyệt] ──► [Khởi tạo kết nối TCP/IP đến Server]
```

1.2. Hậu quả của việc sai lệch tên miền trong "Danh bạ Internet"DNS đóng vai trò như cuốn danh bạ điện thoại của Internet:NXDOMAIN (Non-Existent Domain): Hệ thống DNS đã tra cứu qua toàn bộ các Name Server nhưng không tồn tại bản ghi A/CNAME nào dành cho tên miền sai grenlog.vn.Khẳng định sai lầm: Nam cho rằng Server bị sập nguồn là sai hoàn toàn. Lỗi DNS_PROBE_FINISHED_NXDOMAIN xuất hiện ngay ở bước tra cứu "danh bạ" (trên không gian mạng), yêu cầu chưa từng gửi đến máy chủ thực tế của công ty.🛠️ Phần 2: Triển khai Chẩn đoán & Phân biệt Lỗi2.1. Phân biệt lỗi DNS (NXDOMAIN) vs Lỗi Server (500 Internal Server Error)Tiêu chíLỗi DNS (DNS_PROBE_FINISHED_NXDOMAIN)Lỗi Server (500 Internal Server Error)Bản chấtKhông tìm thấy địa chỉ IP tương ứng với tên miền.Yêu cầu đã đến Server, nhưng Server gặp lỗi xử lý bên trong.Nơi xảy raPhía Client / Tầng dịch vụ Tên miền (DNS network).Phía Server (Backend/Application).Kết nối HTTPChưa được khởi tạo (chưa gửi gói tin HTTP nào).Đã khởi tạo thành công và nhận phản hồi HTTP Status 500.2.2. Quy trình kiểm tra HTTP Status Code (Xác nhận yêu cầu đã chạm Server chưa)Để xác định chính xác yêu cầu đã gửi tới máy chủ hay chưa, thực hiện theo quy trình sau bằng Developer Tools (F12) trên trình duyệt:Mở DevTools: Bấm F12 (hoặc Ctrl + Shift + I / Cmd + Option + I).Chuyển sang tab Network: Tích chọn Preserve log và Disable cache.Thực hiện gửi lại request: Nhập địa chỉ và nhấn Enter.Kiểm tra trạng thái:Nếu thất bại ở bước DNS: Tab Network không hiển thị Status Code (hoặc báo lỗi (failed) net::ERR_NAME_NOT_RESOLVED). ➡️ Yêu cầu chưa chạm tới Server.Nếu chạm tới Server: Tab Network xuất hiện Mã trạng thái HTTP (HTTP Status Code) cụ thể (ví dụ: 200, 404, 500). ➡️ Yêu cầu đã chạm tới Server.🧪 Phần 3: Kiểm thử tình huống mở rộngGiả định: Nam đã sửa đúng tên miền phong-dieu-phoi.greenlog.vn, lỗi DNS biến mất nhưng trình duyệt nhận mã 404 Not Found.Nguyên nhân hệ thống đang gặp phải:Bản chất mã 404: Yêu cầu ĐÃ CHẠM TỚI SERVER thành công, tên miền và IP phân giải chính xác.Vấn đề thực sự:Đường dẫn (URL Path) không tồn tại: Đoạn đường dẫn phía sau tên miền bị gõ sai hoặc trang/tài nguyên đã bị xóa, di chuyển địa chỉ.Cấu hình Web Server (Nginx/Apache/IIS): Server chưa được cấu hình Route hoặc Virtual Host thích hợp để xử lý yêu cầu đến sub-domain phong-dieu-phoi.Ứng dụng Web chưa được Deploy: Web Server đang chạy nhưng thư mục chứa mã nguồn ứng dụng quản trị chưa được upload hoặc khởi chạy thành công.





