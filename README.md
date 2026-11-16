# An toàn dịch vụ thư điện tử

> Bài tập lớn học phần **An toàn mạng (INT1482)** · Khoa An toàn thông tin · Học viện Công nghệ Bưu chính Viễn thông
> **Nhóm 10 – Lớp 02** · GVHD: **TS. Nguyễn Ngọc Điệp**

![License](https://img.shields.io/badge/license-MIT-blue)
![MTA](https://img.shields.io/badge/MTA-Postfix-orange)
![Antivirus](https://img.shields.io/badge/antivirus-ClamAV-red)
![Auth](https://img.shields.io/badge/auth-SPF%20%7C%20DKIM%20%7C%20DMARC-green)

## Giới thiệu

Đề tài tìm hiểu cách hoạt động của hệ thống thư điện tử, phân tích các lỗ hổng phổ biến của Mail Server và **triển khai các cơ chế bảo mật** để ngăn chặn chúng. Hiệu quả được kiểm chứng bằng **2 kịch bản demo**: giả mạo địa chỉ người gửi và phát tán tệp mã độc qua email, mỗi kịch bản đều so sánh kết quả **trước** và **sau** khi cấu hình an toàn.

## Mục lục

1. [Lý thuyết](#1--lý-thuyết)
2. [Cấu hình an toàn](#2--cấu-hình-an-toàn)
3. [Demo 1 – Giả mạo địa chỉ người gửi](#3--demo-1--giả-mạo-địa-chỉ-người-gửi)
4. [Demo 2 – Phát tán tệp mã độc qua email](#4--demo-2--phát-tán-tệp-mã-độc-qua-email)
5. [Cấu trúc repo](#5--cấu-trúc-repo)
6. [Thành viên](#6--thành-viên)

---

## 1. Lý thuyết

### 1.1 Kiến trúc hệ thống Mail

| Thành phần | Vai trò | Ví dụ |
|---|---|---|
| **MUA** (Mail User Agent) | Phần mềm người dùng để soạn, gửi, đọc email | Thunderbird, Outlook |
| **MTA** (Mail Transfer Agent) | Tiếp nhận, định tuyến, chuyển tiếp email giữa các server | Postfix, Sendmail, Exim |
| **MDA** (Mail Delivery Agent) | Phân phối email vào mailbox của người nhận | Dovecot (LMTP) |

### 1.2 Giao thức

- **SMTP**: gửi và chuyển tiếp thư giữa client → server và giữa server → server.
  - *Submission*: MUA gửi thư đến server của mình (thường cổng 587, có xác thực).
  - *Relay*: server chuyển thư đến server của domain người nhận (thường cổng 25, tra cứu bản ghi **MX**).
- **POP3 / IMAP**: client lấy thư về từ server (IMAP giữ thư lại trên server, đồng bộ hai chiều).

Luồng một phiên SMTP: `EHLO` → `MAIL FROM` → `RCPT TO` → `DATA` → `QUIT`.

> **Điểm mấu chốt cho Demo 1:** địa chỉ ở lớp *envelope* (`MAIL FROM`) và trường `From:` trong header là **hai thứ khác nhau**, và SMTP gốc không bắt buộc chúng phải khớp.

### 1.3 Các mối đe dọa chính

- **Open Relay**: server cho phép bất kỳ ai chuyển tiếp thư ra ngoài, dễ bị lợi dụng phát tán spam, phishing.
- **Cấu hình sai SPF / DKIM / DMARC**: làm sụp đổ cơ chế xác thực danh tính người gửi.
- **Giả mạo người gửi (Email Spoofing)**: đặt `From:` giả để mạo danh cá nhân hoặc tổ chức.
- **Lây nhiễm mã độc**: đính kèm tệp thực thi, script, tài liệu Office chứa macro.
- **Từ chối dịch vụ (DoS)**: dồn dập kết nối hoặc gửi tệp cực lớn làm cạn tài nguyên server.
- **Man-in-the-Middle**: chặn bắt thông tin đăng nhập và nội dung khi đường truyền không được mã hoá.

### 1.4 Đánh giá rủi ro

Công thức: **Risk Score = Impact × Likelihood** (mỗi tiêu chí thang 1–5).
Phân mức: 1–4 *Thấp* · 5–9 *Trung bình* · 10–16 *Cao* · 17–25 *Rất cao*.

| # | Lỗ hổng | Likelihood | Impact | Risk | Mức độ |
|---|---|:-:|:-:|:-:|---|
| 1 | Open Relay | 4 | 5 | 20 | 🔴 Rất cao |
| 2 | Cấu hình sai SPF/DKIM/DMARC | 4 | 4 | 16 | 🟠 Cao |
| 3 | Mail Server chưa vá | 3 | 5 | 15 | 🟠 Cao |
| 4 | Giao thức mã hoá lỗi thời | 3 | 4 | 12 | 🟠 Cao |
| 5 | Giả mạo địa chỉ người gửi | 4 | 4 | 16 | 🟠 Cao |
| 6 | Lây nhiễm mã độc qua email | 4 | 5 | 20 | 🔴 Rất cao |

> Chi tiết phân tích xem trong [báo cáo đầy đủ](report/).

---

## 2. Cấu hình an toàn

### 2.1 Môi trường thực nghiệm

```
 ┌──────────────┐   SMTP    ┌────────────────────┐   SMTP    ┌──────────────────────┐   IMAP   ┌──────────────┐
 │ Client gửi   │ ────────► │ Mail Server GỬI    │ ────────► │ Mail Server NHẬN     │ ───────► │ Client nhận  │
 │ (Thunderbird)│           │ mail.lab.test      │           │ mx.receiver.test     │          │ (Thunderbird)│
 └──────────────┘           └────────────────────┘           └──────────────────────┘          └──────────────┘
```

- 2 máy ảo **Ubuntu** đóng vai Mail Server gửi (`lab.test`) và Mail Server nhận (`receiver.test`).
- **Postfix** (MTA) + **Dovecot** (IMAP/IMAPS) + **Maildir** (lưu thư).
- Client: **Thunderbird**.

### 2.2 Các biện pháp triển khai

| # | Biện pháp | Mục đích |
|---|---|---|
| 1 | Đóng Open Relay | Chặn chuyển tiếp thư trái phép |
| 2 | Kích hoạt STARTTLS | Mã hoá đường truyền, chống nghe lén (MITM) |
| 3 | SPF + DKIM + DMARC | Xác thực người gửi, chống giả mạo domain |
| 4 | Tích hợp ClamAV | Quét mã độc trong tệp đính kèm |

#### ① Đóng Open Relay (Postfix `main.cf`)

```ini
smtpd_relay_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_unauth_destination
```

- `permit_mynetworks`: cho phép máy thuộc mạng tin cậy.
- `permit_sasl_authenticated`: cho phép client đã xác thực.
- `reject_unauth_destination`: từ chối chuyển tiếp đến đích không được uỷ quyền.

**Kiểm chứng:** gửi yêu cầu relay từ máy không thuộc mạng tin cậy → server từ chối.

#### ② Kích hoạt STARTTLS

Kiểm tra bằng OpenSSL: kết nối cổng 25, yêu cầu STARTTLS và xác minh chứng chỉ theo tên máy chủ.

```bash
openssl s_client -starttls smtp -connect mail.lab.test:25 -servername mail.lab.test
```

#### ③ SPF, DKIM, DMARC (bản ghi DNS)

```dns
; SPF: chỉ IP này được gửi thư thay mặt lab.test
lab.test.         IN TXT "v=spf1 ip4:192.168.18.128 -all"

; DMARC: từ chối thư không qua kiểm tra
_dmarc.lab.test.  IN TXT "v=DMARC1; p=reject; adkim=r; aspf=r"
```

- **DKIM**: **OpenDKIM** ký thư đi bằng khoá riêng (selector `mail`, domain `lab.test`); khoá công khai được công bố trong DNS.

Kiểm tra bản ghi:
```bash
dig TXT lab.test +short
dig TXT _dmarc.lab.test +short
dig TXT mail._domainkey.lab.test +short
```

#### ④ Tích hợp ClamAV

| Thành phần | Chức năng |
|---|---|
| `clamd` | Dịch vụ nền thực hiện quét |
| `clamdscan` | Công cụ gửi yêu cầu quét đến `clamd` |
| `clamav-milter` | Kết nối ClamAV với Postfix qua giao diện Milter |

### 2.3 Kiểm chứng tổng hợp

Gửi thư thử từ Mail Server gửi sang Mail Server nhận. Header phía nhận ghi nhận:

```
spf=pass    dkim=pass (header.d=lab.test)    dmarc=pass
```

→ Thư hợp lệ vượt qua toàn bộ các lớp bảo vệ.

---

## 3. Demo 1 – Giả mạo địa chỉ người gửi

**Mục tiêu:** chứng minh `From:` hiển thị cho người dùng có thể khác địa chỉ thật ở lớp SMTP envelope, và chứng minh cấu hình an toàn chặn được hành vi này.

| Vai trò | Địa chỉ |
|---|---|
| Envelope sender (`MAIL FROM`) | `alice@lab.test` |
| Header `From:` giả mạo | `"CEO Company" <ceo@receiver.test>` |
| Người nhận | `verify@receiver.test` |

### Trước khi cấu hình an toàn

Email giả được gửi thủ công bằng công cụ `swaks` và **được chuyển thành công**. So sánh header:

| Trường | Email giả mạo | Email thật |
|---|---|---|
| `From:` | `CEO Company <ceo@receiver.test>` | `Alice <alice@lab.test>` |
| SPF | pass (chỉ xét domain envelope) | pass |
| DKIM | **none** | pass (`d=lab.test`) |
| DMARC | **fail** (`header.from=receiver.test`) | pass |

Log Postfix (`/var/log/mail.log`) xác nhận envelope sender thật là `alice@lab.test`, trong khi header `From:` là một danh tính khác.

### Sau khi cấu hình an toàn

DMARC `p=reject` **từ chối** email giả mạo: gửi thất bại, người nhận không nhận được thư, Thunderbird nhận thư báo lỗi và log Mail Server khớp với lỗi đó.

### Đánh giá rủi ro

| Likelihood | Impact | Risk Score | Mức |
|:-:|:-:|:-:|---|
| 4/5 | 4/5 | **16/25** | 🟠 Cao |

 **Video demo:** xem tại [`demo/`](demo/) · 📷 Ảnh chụp chi tiết: [báo cáo](report/)

---

## 4. Demo 2 – Phát tán tệp mã độc qua email

**Mục tiêu:** đánh giá khả năng phát hiện và chặn tệp đính kèm độc hại trong quá trình truyền nhận email.

> Dùng tệp **[EICAR](https://www.eicar.org/download-anti-malware-testfile/)**: mẫu kiểm thử chuẩn của phần mềm diệt virus, **không phải mã độc thật**. Repo này **không** chứa tệp EICAR.

### Trước khi cấu hình an toàn

Gửi email đính kèm tệp EICAR → OpenDKIM ký thư, Postfix chuyển tiếp, Mail Server nhận tiếp nhận (SPF, DKIM vẫn hoạt động) → **thư và tệp đến được hộp thư người nhận trên Thunderbird.**

### Sau khi cấu hình an toàn

Gửi lại cùng email với ClamAV đã tích hợp → thư **bị chặn**, gửi thất bại, người nhận **không thấy** email trong Thunderbird.

### Đánh giá rủi ro

| Likelihood | Impact | Risk Score | Mức |
|:-:|:-:|:-:|---|
| 4/5 | 5/5 | **20/25** | 🔴 Rất cao |

**Video demo:** xem tại [`demo/`](demo/) · 📷 Ảnh chụp chi tiết: [báo cáo](report/)

---

## 5. Cấu trúc repo

```
.
├── demo/        # Link video / tài liệu demo
├── report/      # Báo cáo đầy đủ (PDF)
├── .gitignore
├── LICENSE
└── README.md
```

## 6. Thành viên

| STT | Họ và tên | MSSV |
|:-:|---|---|
| 1 | Nguyễn Tuấn Đạt | B23DCAT043 |
| 2 | Vũ Thành Hữu | B23DCAT123 |
| 3 | Trần Đình Nhật Huy | B23DCAT134 |
| 4 | Phạm Văn Phúc | B23DCAT239 |
| 5 | Nguyễn Thanh Tùng | B23DCAT327 |

## Lưu ý

Dự án chỉ phục vụ **mục đích học tập**, thực hiện trong **môi trường lab cô lập**. Không áp dụng các kỹ thuật demo lên hệ thống thật khi chưa được phép.

## 📜 License

Phát hành theo giấy phép [MIT](LICENSE).
