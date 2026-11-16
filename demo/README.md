# Thực Nghiệm (Demo Videos)

Thư mục này lưu trữ đường dẫn truy cập toàn bộ video ghi hình quá trình thực nghiệm tấn công và cấu hình phòng thủ hệ thống Mail Server (`mail.lab.test` & `mx.receiver.test`) của **Nhóm 10 - Lớp 02**[cite: 1].

---

## 1. Link Thư Mục Google Drive Tổng Hợp
Nhấn vào nút bên dưới để mở trực tiếp thư mục Google Drive chứa toàn bộ video chất lượng gốc:

[![Google Drive Folder](https://img.shields.io/badge/Google_Drive-Mở_Thư_Mục_Video_Demo-4285F4?style=for-the-badge&logo=googledrive&logoColor=white)](https://drive.google.com/drive/folders/1VQiaYkCP0wRt1Exgrvl-KPwUILK6UtlD)

---

## 2. Danh Sách Các Video Thành Phần

Nếu bạn muốn dẫn link tới từng video lẻ trên Drive, hãy thay link vào bảng dưới đây (nếu chỉ dùng 1 link thư mục chung ở trên thì có thể xóa bảng này):

| STT | Kịch bản thực nghiệm | Mô tả chi tiết nội dung video |
| :-: | :--- | :--- | :-: |
| 1 | **Cấu hình cơ bản & Đóng Open Relay** | Kiểm thử gửi thư trái phép qua `swaks` và xác minh chặn thành công với mã lỗi `554 5.7.1 Relay access denied`[cite: 1] |
| 2 | **Kích hoạt STARTTLS & Bộ bản ghi DNS** | Kiểm tra kết nối mã hóa TLSv1.3 bằng `openssl` và truy vấn xác thực các bản ghi `SPF`, `DKIM`, `DMARC` bằng `dig`[cite: 1] |
| 3 | **Demo 1: Giả mạo địa chỉ (Email Spoofing)** | Thực nghiệm mạo danh `CEO Company <ceo@receiver.test>` trước và sau khi bật chính sách `DMARC Reject` (`550 5.7.1`)[cite: 1] |
| 4 | **Demo 2: Phát tán mã độc qua Email** | Kiểm thử gửi tệp đính kèm `EICAR` trước và sau khi tích hợp bộ lọc quét virus `clamav-milter` trên Postfix[cite: 1] |

---
