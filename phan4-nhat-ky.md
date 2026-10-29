# Nhật ký làm bài

**Họ tên:** [điền]
**Nhận đề lúc:** [điền: giờ nhận email]
**Nộp bài lúc:** [điền]
**Tổng thời gian làm thật (ước tính):** [điền: tự cộng các phiên bên dưới, tính cả thời gian không chat với AI]

## Các phiên làm việc

> Chỉ ghi đúng với thực tế của bạn. Các mốc "gợi ý" lấy từ lịch sử chat, hãy sửa theo thời gian bạn thực sự làm.

| Thời điểm | Thời lượng | Làm phần nào | Ghi chú (kẹt ở đâu, đã thử gì) |
|---|---|---|---|
| [điền] (gợi ý: 27/10 chiều tối) | [điền] | Phần 1: điều tra sự cố | Dữ liệu nhiều (log LB ~1,1 MB, nginx ~210 KB). Loại trừ giả thuyết bot bằng cách so mốc thời gian. [điền: chỗ bạn tự chạy lại lệnh grep] |
| [điền] (gợi ý: 27-28/10) | [điền] | Phần 2: viết tool Proxmox và test | [điền: phần bạn tự viết, chỗ bị kẹt] |
| [điền] (gợi ý: 28/10 tối) | [điền] | Phần 2: sửa tool theo review, sinh README | Sửa: bảng hiển thị tải trước migrate, mã thoát khác 0 khi lỗi, cảnh báo node không migrate được, số test. Chạy lại test và tool trên dữ liệu thật |
| [điền] | [điền] | Phần 3 (Đề B): viết bản nháp | Tự viết bản đầu, đặt giả định LB truyền IP gốc qua XFF/PROXY protocol [điền: xác nhận đúng thực tế] |
| [điền] | [điền] | Phần 3: rà soát và sửa | Phát hiện lỗi logic ở lớp L4 (xem mục Giả định); viết lại bảng so sánh cho dễ đọc; bổ sung benchmark |
| [điền] | [điền] | Phần 4: nhật ký, đóng gói, gửi bài | |

## Giả định đã đặt ra

**Phần 1**
- Việc kỹ sư `hieunv` gây ra lỗi `dailly` là **suy luận**. Log chỉ cho thấy anh ấy mở file bằng vim lúc 13/10 17:42; logrotate chạy tốt ngày 13/10 và báo lỗi từ 14/10 00:00.
- Log LB chỉ có 5 domain trong khung 00:50-02:25; mình coi đó là mẫu của khoảng 40 site trên máy.
- `dmesg | tail -8` không có dòng OOM, nhưng chỉ có 8 dòng nên kết luận loại trừ OOM có độ tin cậy hạn chế. Không có `df -i` nên không kiểm chứng được inode.

**Phần 2**
- `cpu_usage` trong khoảng (1, 100] được hiểu là phần trăm và chia 100 (đề nói 0-1 nhưng dữ liệu có `31.0`, `98.0`, `20.8`).
- Node `hcm-c3-n06` có `mem_total=262144` được coi là dữ liệu hỏng, không làm node nguồn hoặc đích; nghi exporter xuất sai đơn vị (nếu là MiB thì đúng 256 GiB).
- Ngưỡng 80% CPU / 85% RAM / 4x overcommit chọn theo kinh nghiệm chung, chưa hiệu chỉnh bằng dữ liệu thật.
- VM `stopped` không tính vào tải; node `offline` không nhận VM.
- Snapshot có `vmid` 2001 xuất hiện trên cả `hcm-c3-n01` và `hcm-c3-n02`; tool chưa xử lý trường hợp trùng (xem mục chưa xong).

**Phần 3**
- LB (`lb-hcm-01`) là L7 vì có access log HTTP ở Phần 1. Do đó nftables trên node chỉ thấy IP của LB, không đọc được `X-Forwarded-For` hay PROXY protocol, nên lớp L4 chặn IP khách được đặt ở LB/edge, còn trên node L4 chỉ dùng để cho phép 80/443 từ IP LB.
- Ngưỡng `rate`, `burst` trong cấu hình mẫu chỉ là điểm khởi đầu, chưa đo trên traffic thực; dải IP của LB trong `set_real_ip_from` là ví dụ.
- Benchmark Cloudflare (2018) đo drop gói UDP chứ không đo chi phí Nginx từ chối HTTP request, nên chỉ dùng để xếp thứ bậc giữa các điểm chặn.

## Câu hỏi đã gửi cho người ra đề

- [điền, hoặc ghi "Không có". Nên có mail xác nhận nhận đề trong 24 giờ và nêu mốc dự kiến nộp]

## Dùng AI và công cụ

| Bước | Công cụ | Dùng để làm gì | Bạn đã kiểm tra, sửa lại gì |
|---|---|---|---|
| Phần 1: đọc log, tìm nguyên nhân, dựng khung postmortem | Claude | Đọc syslog, snapshot, log nginx/php/mysql/LB; đếm 502/504, so mốc giờ bot với lỗi đầu tiên; soạn `phan1-postmortem.md` | [điền: ví dụ tự grep lại các dòng trích dẫn, tự chạy lệnh, viết lại các đoạn nào, sửa chỗ nào] |
| Phần 2: review tool và README | Claude | Chỉ ra 4 điểm cần sửa (bảng trước/sau migrate, exit code, số test, node không migrate được); chạy test và tool; viết `generate_readme.py` sinh README bằng output thật | [điền: phần code bạn tự viết, phần bạn sửa sau review, đã tự chạy lại tool và test chưa] |
| Phần 2: kiểm tra dữ liệu | Claude | Phát hiện `vmid` 2001 trùng trên hai node | [điền: bạn tự xác minh trong file JSON chưa, quyết định xử lý thế nào] |
| Phần 3: rà soát bản nháp | Claude | Đối chiếu bản nháp với yêu cầu đề, chỉ ra chỗ sai | [điền: đã tự suy luận lại vì sao nftables không đọc được XFF/PROXY protocol và đồng ý với kết luận] |
| Phần 3: đối chiếu nguồn | Claude (đọc tài liệu chính thức) | Đối chiếu số liệu và câu chữ với tài liệu nginx `limit_req`, nftables Meters, nginx realip | [điền: đã tự mở các trang để xác nhận các con số, ví dụ 16.000/8.000 state mỗi MB, `limit_req_dry_run` từ 1.17.1, `ct count` không dùng với set có `timeout`] |
| Phần 3: tìm benchmark | Claude (tìm kiếm web) | Tìm bài benchmark uy tín về chi phí drop gói | [điền: đã tự mở bài gốc trên blog Cloudflare, kiểm tra lại các số 175k, 333k, 1,69M, 1,53M pps; tự tính tỉ lệ 4,6 đến 8,7 lần; đã ghi rõ giới hạn của benchmark] |
| Phần 3: trình bày | Claude | Viết lại bảng so sánh gọn hơn, thêm cột "Nghiêng về" | [điền: đã đọc lại và chỉnh gì] |
| Ảnh chụp bằng chứng phần 1 | (bạn tự làm) | [điền: chỉ ghi nếu bạn tự chụp từ lệnh bạn chạy] | |

## Phần chưa xong hoặc chưa hài lòng

- **Phần 1:** tốc độ ghi `access.log` (~6 GB/ngày) cao hơn nhiều so với lưu lượng thấy trong log LB; cần đọc `access.log` thật để biết vhost/IP nào ghi nhiều. Các lệnh khắc phục chưa chạy trên máy thật.
- **Phần 1:** request 5xx đầu tiên trong log LB có `rt=60.5s` ghi lúc 01:33:05, tức bắt đầu sớm hơn lỗi `tar` khoảng 1 phút; chưa giải thích được.
- **Phần 2:** tool chưa phát hiện `vmid` trùng; chưa có test riêng cho VM thiếu `vmid` và ngưỡng biên. Greedy không tối ưu toàn cục và không tính chi phí migrate.
- **Phần 3:** chưa có benchmark từ lab cá nhân. Nếu có thêm thời gian: dựng Nginx + PHP-FPM, giả lập crawl bằng `wrk`/`hey`, so sánh số PHP-FPM worker bận, p95 latency và CPU giữa ba trường hợp: không giới hạn / chỉ L7 / L7 + L4.
- **Phần 3:** chưa trích source code Nginx cho cơ chế đếm và xóa state; các nhận định đó dựa trên tài liệu chính thức.
- **Phần 3:** độ dài có thể sát hoặc vượt một trang A4, cần kiểm tra bản in.

## Nếu làm lại, bạn sẽ làm khác điều gì?

- Đặt rõ giả định về kiến trúc LB (L4 hay L7) ngay từ đầu, trước khi thiết kế lớp L4, vì giả định này quyết định L4 đặt ở đâu.
- Dựng lab đo trước khi viết, để mọi nhận định hiệu năng có số liệu thật thay vì dựa vào benchmark bên ngoài.
- Tự đọc log và chạy lại các lệnh trước khi nhờ AI, và nhắc việc xác nhận nhận đề từ sớm.
- [điền thêm bằng suy nghĩ của chính bạn]
