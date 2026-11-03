# Bài 1: Sự cố không truy cập được hệ thống quản trị



---

## Phần 1. Phân tích

### Input / Output của DNS
- **Input:** tên miền `phong-dieu-phoi.greenlog.vn`
- **Output:** địa chỉ IP của máy chủ (ví dụ `203.0.113.10`)

### Tiến trình DNS Lookup
1. Nam gõ tên miền vào trình duyệt.
2. Trình duyệt kiểm tra cache của trình duyệt, rồi cache hệ điều hành / file `hosts`.
3. Nếu chưa có, gửi truy vấn tới **DNS Resolver** (của nhà mạng hoặc 8.8.8.8).
4. Resolver hỏi **Root Server**, Root chỉ tới máy chủ quản lý đuôi `.vn`.
5. Resolver hỏi **TLD Server (.vn)**, TLD chỉ tới máy chủ quản lý `greenlog.vn`.
6. Resolver hỏi **Authoritative Name Server**, nơi này trả về địa chỉ **IP**.
7. Resolver trả IP về cho trình duyệt, trình duyệt kết nối tới IP và gửi yêu cầu HTTP.

### Sơ đồ luồng dữ liệu
```text
Tên miền -> Cache (trình duyệt/OS) -> DNS Resolver -> Root -> TLD (.vn)
         -> Authoritative NS -> Địa chỉ IP -> Kết nối Web Server -> HTTP Response
```

### Phân biệt lỗi DNS và lỗi 500
| Tiêu chí | Lỗi DNS (NXDOMAIN) | Lỗi 500 |
|---|---|---|
| Thời điểm | Trước khi kết nối server | Sau khi đã kết nối server |
| Bản chất | Không tìm thấy IP cho tên miền | Server nhận request nhưng xử lý bị lỗi |
| Có mã HTTP? | Không | Có (mã 500) |
| Phía lỗi | **Client** | **Server** |

### Hậu quả khi sai tên miền
- DNS so khớp chính xác từng ký tự, nên `grenlog.vn` và `greenlog.vn` là **hai tên miền khác nhau**.
- "Cuốn danh bạ Internet" không có mục `grenlog.vn`, nên trả về **NXDOMAIN**, trình duyệt không có IP để kết nối.
- Dễ chẩn đoán nhầm là server sập.
- Có rủi ro **typosquatting**: kẻ xấu đăng ký tên miền gần giống để dựng trang giả, đánh cắp tài khoản.

---

## Phần 2. Triển khai: quy trình kiểm tra HTTP Status Code
1. **Kiểm tra chính tả** tên miền.
2. **Kiểm tra DNS:** `nslookup phong-dieu-phoi.greenlog.vn`. Nếu `NXDOMAIN` thì dừng, chưa tới server.
3. **Kiểm tra kết nối:** `ping phong-dieu-phoi.greenlog.vn` (chỉ tham khảo, vì có thể bị firewall chặn).
4. **Gửi request:** `curl -I https://phong-dieu-phoi.greenlog.vn` hoặc F12 → tab Network → cột Status.
5. **Đọc kết quả:**

| Kết quả | Đã tới server? | Ý nghĩa |
|---|---|---|
| NXDOMAIN, timeout (không có mã HTTP) | Chưa | Lỗi DNS hoặc mạng |
| 2xx | Rồi | Thành công |
| 4xx (401, 403, 404) | Rồi | Lỗi phía Client (URL, quyền) |
| 5xx (500, 502, 503) | Rồi | Lỗi phía Server |

**Quy tắc:** có mã HTTP nghĩa là yêu cầu đã chạm tới máy chủ.

---

## Phần 3. Kiểm thử: sửa đúng tên miền nhưng vẫn 404
- DNS đã hoạt động và yêu cầu **đã tới server**, nên server **đang chạy bình thường** (không sập).
- Mã `404 Not Found` nghĩa là server **không tìm thấy tài nguyên/đường dẫn** được yêu cầu.
- Nguyên nhân có thể: sai đường dẫn (path), trang bị xóa hoặc đổi tên, cấu hình Web Server / Virtual Host sai, deploy thiếu file.
- **Hướng xử lý:** kiểm tra lại URL đầy đủ, nếu đúng mà vẫn 404 thì báo đội phát triển/vận hành kiểm tra cấu hình.

---

