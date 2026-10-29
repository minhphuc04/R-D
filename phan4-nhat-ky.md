# Nhật ký làm bài

**Họ tên:** [BẠN ĐIỀN]
**Nhận đề lúc:** [BẠN ĐIỀN: giờ nhận email, ví dụ 27/10/2026 08:xx]
**Nộp bài lúc:** [BẠN ĐIỀN]
**Tổng thời gian làm thật (ước tính):** [BẠN ĐIỀN: tự cộng các phiên bên dưới, tính cả thời gian không chat với AI]

## Các phiên làm việc

> Chỉ ghi những gì đúng với thực tế của bạn. Mốc gợi ý lấy từ lịch sử chat; hãy sửa theo thời gian bạn thực sự làm.

| Thời điểm | Thời lượng | Làm phần nào | Ghi chú (kẹt ở đâu, đã thử gì) |
|---|---|---|---|
| 27/10 [giờ] | [..] | Đọc đề, chuẩn bị | [BẠN ĐIỀN] |
| 27/10 ~17:50-19:30 | [..] | Phần 1: đọc log, dựng postmortem | Dữ liệu nhiều (log LB ~1,1MB, nginx ~210KB); loại trừ giả thuyết bot bằng cách so mốc thời gian. [BẠN ĐIỀN: chỗ bạn tự chạy lại lệnh] |
| 27-28/10 [giờ] | [..] | Phần 2: viết tool, test | [BẠN ĐIỀN: bạn tự viết/sửa gì] |
| 28/10 ~20:10-21:10 | [..] | Phần 2: sửa tool theo review, sinh README | Sửa: bảng hiển thị tải trước migrate, mã thoát khác 0 khi lỗi, thêm test còn thiếu, cảnh báo node không migrate được |
| [..] | [..] | Phần 3 | [BẠN ĐIỀN: chọn đề A hay B] |

## Giả định đã đặt ra

Phần 1:
- Việc kỹ sư `hieunv` gây ra lỗi `dailly` là **suy luận** (log chỉ cho thấy anh ấy mở file bằng vim lúc 13/10 17:42; logrotate chạy tốt ngày 13/10 và lỗi từ 14/10 00:00).
- Log LB chỉ có 5 domain trong khung 00:50-02:25; mình coi đó là mẫu của ~40 site trên máy.
- `dmesg | tail -8` không có dòng OOM, nhưng chỉ có 8 dòng nên kết luận loại trừ OOM có độ tin cậy hạn chế. Không có `df -i` nên không kiểm chứng được inode.

Phần 2:
- `cpu_usage` trong khoảng (1, 100] được hiểu là phần trăm và chia 100 (đề nói 0-1 nhưng dữ liệu có `31.0`, `98.0`, `20.8`).
- Node `hcm-c3-n06` có `mem_total=262144` được coi là dữ liệu hỏng, không làm node nguồn/đích; nghi exporter xuất sai đơn vị (nếu là MiB thì 256 GiB).
- Ngưỡng 80% CPU / 85% RAM / 4x overcommit là chọn theo kinh nghiệm chung, chưa hiệu chỉnh bằng dữ liệu thật.
- VM `stopped` không tính vào tải; node `offline` không nhận VM.
- Snapshot có `vmid` 2001 xuất hiện trên cả `hcm-c3-n01` và `hcm-c3-n02`; tool chưa xử lý trường hợp trùng (xem phần chưa xong).

## Câu hỏi đã gửi cho người ra đề

- [BẠN ĐIỀN, hoặc ghi "Không có". Nên có ít nhất một mail xác nhận nhận đề trong 24 giờ.]

## Dùng AI và công cụ

| Bước | Công cụ | Dùng để làm gì | Bạn đã kiểm tra, sửa lại gì |
|---|---|---|---|
| Phần 1: đọc log, tìm nguyên nhân, dựng khung postmortem | Claude | Đọc syslog, snapshot, log nginx/php/mysql/LB; đếm 502/504, so mốc giờ bot với lỗi đầu tiên; soạn `phan1-postmortem.md` | [BẠN ĐIỀN: ví dụ tự grep lại các dòng trích dẫn, tự chạy lệnh trên log, viết lại các đoạn nào, sửa chỗ nào] |
| Phần 2: review tool và README | Claude | Nhận xét 4 điểm cần sửa (bảng trước/sau migrate, exit code, số test, node không migrate được); chạy test và tool; viết `generate_readme.py` sinh README bằng output thật | [BẠN ĐIỀN: phần code bạn tự viết, phần bạn sửa sau review, đã tự chạy lại tool/test chưa] |
| Phần 2: phát hiện dữ liệu | Claude | Phát hiện `vmid` 2001 trùng trên hai node | [BẠN ĐIỀN: bạn tự xác minh trong file JSON chưa, quyết định xử lý thế nào] |
| Ảnh chụp bằng chứng phần 1 | (bạn) | [BẠN ĐIỀN: chỉ ghi nếu bạn tự chụp từ lệnh bạn chạy] | |
| Phần 3 | [BẠN ĐIỀN] | [BẠN ĐIỀN] | [BẠN ĐIỀN] |

## Phần chưa xong hoặc chưa hài lòng

- Phần 1: tốc độ ghi `access.log` (~6 GB/ngày) cao hơn nhiều so với lưu lượng thấy trong log LB; cần đọc `access.log` thật để biết vhost/IP nào ghi nhiều. Các lệnh khắc phục chưa chạy trên máy thật.
- Phần 1: request 5xx đầu tiên trong log LB có `rt=60.5s` ghi lúc 01:33:05, tức bắt đầu sớm hơn lỗi `tar` khoảng 1 phút; chưa giải thích được.
- Phần 2: tool chưa phát hiện `vmid` trùng; chưa có test riêng cho VM thiếu `vmid` và ngưỡng biên. Greedy không tối ưu toàn cục và không tính chi phí migrate.
- [BẠN ĐIỀN: phần 3 và các phần khác]

## Nếu làm lại, bạn sẽ làm khác điều gì?

- [BẠN ĐIỀN bằng suy nghĩ của chính bạn. Gợi ý để nghĩ: tự đọc log trước khi nhờ AI, viết test cho dữ liệu bẩn sớm hơn, xác nhận nhận đề sớm.]
