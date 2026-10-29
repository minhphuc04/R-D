## Phần 1: Điều tra sự cố (Thời lượng ước tính: ~1.5 giờ)

**Bối cảnh:** máy `web-hcm-07` là một node shared hosting (Nginx + PHP-FPM 8.1 + MySQL 8) chạy khoảng 40 website của khách hàng. Rạng sáng ngày **20/10/2026** (khoảng từ 01:00), hàng loạt website trên máy này bắt đầu trả về lỗi 502/504. Đến 02:21, kỹ sư on-call SSH vào kiểm tra và lưu lại một số output hiện trạng.
--
## 1. Tóm tắt sự cố

Từ khoảng 01:33 ngày 20/10, các website trên `web-hcm-07` trả 502/504 cho gần như toàn bộ request động (PHP). Trong log LB, 2.775 request lỗi so với 157 request động trả 200 sau mốc này, tức khoảng 95% request động lỗi. Cả 5 domain xuất hiện trong log LB đều bị ảnh hưởng. Ảnh hưởng kéo dài đến lúc on-call vào máy (02:21), khoảng 48 phút.

**Nguyên nhân gốc:** phân vùng `/` (118 GB) đầy 100%. Việc xoay log nginx bị hỏng từ 14/10 vì lỗi gõ sai `dailly` trong `/etc/logrotate.d/nginx`, nên `access.log` phình lên 37 GB. Khi job backup 01:00 hằng ngày ghi thêm ~16 GB vào cùng phân vùng, ổ đĩa đầy lúc 01:32:57. MySQL treo ở bước ghi binlog, các worker PHP-FPM chờ MySQL và chiếm hết `pm.max_children`, nginx không còn worker để giao request, nên trả 502/504.

## 2. Timeline

| Thời điểm | Sự kiện | Nguồn |
|---|---|---|
| 13/10 00:00 | logrotate chạy bình thường (không có lỗi) | `syslog` |
| **13/10 17:42–17:44** | `hieunv` sửa `/etc/logrotate.d/nginx` bằng vim, rồi `systemctl reload nginx`| `syslog` |
| **14/10 00:00:02** | Lần đầu logrotate báo `unknown option 'dailly'`, bỏ qua cấu hình nginx. Lỗi lặp lại mỗi đêm đến 20/10 | `syslog` |
| 14/10 → 20/10 | `disk-check.sh` ghi mức dùng đĩa: 57% → 62 → 67 → 72 → 77 → **82% (WARN, 19/10)** → **87% (WARN, 20/10 00:30)**, tăng khoảng 5 điểm % (~5,9 GB) mỗi ngày. Trước 13/10 chỉ tăng khoảng 1%/ngày | `syslog` |
| 20/10 01:00:02 | Backup `/home` → `/var/backups/sites-20261020.tar.gz` bắt đầu, file backup cũ (19 GB) vẫn còn trên đĩa | `syslog`, `backup-sites.sh` |
| **01:32:57** | `tar: Wrote only 4096 of 10240 bytes`, lỗi không phục hồi. Lúc 01:32:58 script ghi `ERROR backup failed`, **file dở dang ~16 GB vẫn nằm lại** | `syslog`, `du-var.txt` |
| 01:32:59–01:33:06 | Nginx `pwrite() ... (28: No space left on device)`, không ghi được `access.log`. MySQL: `Disk is full writing './binlog.000187'`. PHP: `session_start(): ... No space left on device` | `nginx/error.log`, `mysql/error.log`, `php/php8.1-fpm.log` |
| 01:33:05 | Request 5xx đầu tiên trên LB (`rt=60.587`). Trước đó 1.566 request từ 00:50 đều không có 5xx | `lb/access-lb-hcm-01.log` |
| 01:34:02 | `server reached pm.max_children setting (50)`, lặp lại đều đặn | `php/php8.1-fpm.log` |
| 01:34:40 | Zabbix: `vfs.fs.size[/,pfree] became not supported: Cannot write to log file`. Giám sát mất khả năng theo dõi đĩa | `syslog` |
| 01:41:03 | IP 45.135.232.17 (UA tự xưng AhrefsBot) bắt đầu crawl phân trang sâu của `tintuc24h.net` | `lb/...log` |
| 02:21:11 | On-call SSH vào máy; 02:23 chạy các lệnh chẩn đoán, `df` cho `/` 100% | `syslog`, `snapshot/` |

## 3. Nguyên nhân gốc và chuỗi hệ quả

**(a) Log nginx không được xoay.** Cấu hình bị gõ sai:
```
/etc/logrotate.d/nginx:   dailly          <- phải là "daily", nhưng chưa phải là nguyên nhân.
syslog 14/10..20/10 00:00:02: logrotate: error: /etc/logrotate.d/nginx:3 unknown option 'dailly' -- ignoring line
syslog 14/10..20/10 00:00:02: logrotate: error: found error in /var/log/nginx/*.log , skipping
du-var.txt:  -rw-r----- 1 www-data adm 37G Oct 20 01:33 access.log      (file xoay gần nhất: access.log.1, mtime 13/10)
```
`access.log` 37G cộng với `error.log` 641M chiếm phần lớn `/var/log` (38G). Mức tăng ~6 GB/ngày của `access.log` khớp với tốc độ tăng đĩa ~5,9 GB/ngày (5 điểm %) kể từ 14/10.

**(b) Đĩa đầy nhờ backup ghi cùng phân vùng.** `backup-sites.sh` ghi vào `/var/backups` trên cùng `/`, và chỉ xóa bản cũ sau khi bản mới thành công, nên cần chỗ cho hai bản (~19 GB mỗi bản) cùng lúc. Lúc 00:30 đĩa dùng 87% (còn khoảng 15 GB trống), bản backup dở dang dừng ở ~16 GB, khớp với mức trống đó. Script không xóa file dở dang khi lỗi (`else` chỉ ghi log), nên phân vùng giữ nguyên ở 100%:
```
df-h.txt (02:23:41):  /dev/sda2  118G  118G  0  100% /
du-var.txt:  sites-20261019.tar.gz 19G ; sites-20261020.tar.gz 16G ; /var/backups = 35G
```

**(c) MySQL treo, PHP-FPM chờ MySQL.**
```
mysql/error.log: [ERROR] [MY-000035] Disk is full writing './binlog.000187' (OS errno 28) ... Waiting for someone to free space... Retry in 60 secs.
php-fpm.log:     mysqli_real_connect(): Error while reading greeting packet   (tintuc24h)
snapshot/ss-summary.txt:  mysqladmin processlist | grep -c "Waiting for"  ->  49
```
Số tiến trình chờ (49) rất sát giới hạn `pm.max_children = 50`.

**(d) Hết worker nên sinh 502 và 504.**
```
ss -xlp | grep php:  LISTEN 511 511 /run/php/php8.1-fpm.sock      <- Recv-Q = backlog = 511: hàng đợi socket đã đầy
nginx: connect() to unix:/run/php/php8.1-fpm.sock failed (11: Resource temporarily unavailable)   -> 502
nginx: upstream timed out (110: Connection timed out) while reading response header from upstream -> 504
```
LB xác nhận hai nhóm: 1.231 lỗi 502 có `rt` ≤ 0,01s (bị từ chối ngay vì hàng đợi đầy) và 1.544 lỗi 504 có `rt` 60–61s (đã vào hàng đợi, chờ hết timeout ~60s).

**Nginx vẫn sống:** sau 01:33, tất cả 2.775 request 5xx đều là request động, trong khi 215 request tĩnh (`/wp-content/uploads/...`) vẫn trả 200.

## 4. Các giả thuyết đã loại trừ

| Giả thuyết | Lý do loại / hạ mức |
|---|---|
| **Bot crawl / DDoS** (IP 45.135.232.17, UA AhrefsBot) | Bot chỉ bắt đầu lúc **01:41:03**, trong khi lỗi 5xx đầu tiên ở khách khác lúc **01:33:05** (sớm hơn 8 phút). Lưu lượng nền 00:50–01:30 ổn định ~185 request/5 phút, rt ≈ 0,2s. Cả 1.259 request của bot đều nhận 5xx, chỉ vào `tintuc24h.net`. Các domain khác (không dính bot) lỗi cùng tỷ lệ ~80% từ 01:33. Bot là **yếu tố làm trầm trọng thêm** (chiếm hàng đợi), không phải nguyên nhân. Ngoài ra cùng IP bị UFW chặn khi dò cổng 3306 lúc 01:51 nên UA AhrefsBot có thể giả mạo |
| **CPU / RAM / OOM** | `top`: CPU idle 94,9%, load 1,84 (máy rảnh, worker bị chặn). `free -m`: available 22 GB, swap dùng 12 MB. `dmesg | tail -8` không có OOM (chỉ có 8 dòng cuối, nên kết luận hạn chế) |
| **`pm.max_children` quá thấp** | Cảnh báo chỉ xuất hiện sau 01:34:02, ngay sau lỗi ENOSPC. Trước đó (00:12) log chỉ ghi NOTICE bình thường. Đây là triệu chứng, không phải gốc. Tăng giá trị này chỉ dời lúc hết worker |
| **MySQL lỗi nội tại** (crash, max_connections, DNS) | `mysqld` vẫn chạy và trả lời `processlist`. Lỗi duy nhất là ENOSPC khi ghi binlog. Warning 14/10 `IP address '10.20.1.31' could not be resolved` là cảnh báo reverse DNS, không liên quan |
| **Mạng / firewall / LB** | Nginx trả được file tĩnh, LB nhận được phản hồi. `ss`: estab 412, orphaned 3, không có dấu hiệu cạn socket. Các dòng `UFW BLOCK` là quét cổng ngẫu nhiên, đã có từ 12/10 |
| **Cạn inode** | Chưa kiểm chứng được, vì không có output `df -i`. Chỉ cần kiểm tra thêm, đĩa đã đầy theo dung lượng nên không cần thiết để giải thích sự cố |

## 5. Khắc phục tức thời (02:23)

Nguyên tắc: **không đụng `/home`, `/var/lib/mysql`, file session, source hay database của khách.** Chỉ xóa bản backup hỏng do chính script tạo ra, và nén log.

```bash
# 1. Giải phóng nhanh ~16 GB: xóa backup DỞ DANG (script đã báo lỗi lúc 01:32:58, file không dùng được).
#    GIỮ NGUYÊN sites-20261019.tar.gz (bản đầy đủ gần nhất)
ls -lh /var/backups/
rm /var/backups/sites-20261020.tar.gz
df -h /          # kỳ vọng ~73%

# 2. Sửa lỗi cấu hình logrotate và kiểm tra cú pháp trước khi áp dụng
sed -i 's/^\s*dailly/\tdaily/' /etc/logrotate.d/nginx
logrotate -d /etc/logrotate.d/nginx 2>&1 | grep -i error   # không được còn lỗi

# 3. Đổi tên access.log 37 GB (không xóa), cho nginx mở lại file mới, nén nền với ưu tiên thấp
mv /var/log/nginx/access.log /var/log/nginx/access.log.pre-incident
nginx -s reopen
nice -n19 ionice -c3 gzip -1 /var/log/nginx/access.log.pre-incident &   # dự kiến giải phóng thêm ~30 GB

# 4. MySQL tự retry mỗi 60s nên KHÔNG restart (tránh crash-recovery khi đang chờ ghi)
tail -f /var/log/mysql/error.log                      # theo dõi thông báo hồi phục
mysqladmin processlist | grep -c "Waiting for"        # phải giảm về gần 0

# 5. Nếu sau 1-2 phút hàng đợi PHP-FPM vẫn không xả: reload mềm (không kill request đang chạy)
ss -xlp | grep php                                    # Recv-Q phải về ~0
systemctl reload php8.1-fpm

# 6. Giám sát hồi phục (restart zabbix-agent nếu item vfs.fs.size vẫn "not supported")
for h in site42.vn tintuc24h.net noithatmoc.com; do
  curl -s -o /dev/null -w "$h %{http_code} %{time_total}s\n" -H "Host: $h" http://127.0.0.1/; done

# 7. (Tùy chọn) Chặn IP bot đang dò cổng 3306 và crawl sâu
ufw deny from 45.135.232.17
```

Kiểm tra sau cùng: `df -h /`, log LB không còn 5xx, `mysql/error.log` ngừng ghi lỗi ENOSPC. Do backup 01:00 đêm sau sẽ chạy lại, cần xác nhận còn đủ chỗ trống trước khi nó chạy (xem mục 6).

## 6. Phòng ngừa

**Giám sát và cảnh báo (phát hiện sớm, hiện tại mất 48 phút và đĩa tăng 7 ngày không ai biết)**
- `disk-check.sh` chỉ ghi WARN vào syslog (19/10 và 20/10), không gửi đi đâu. Đưa vào Zabbix trigger: >80% cảnh báo, >90% khẩn cấp, thêm trigger dự báo (`timeleft`/`forecast`) để báo trước khi đầy.
- Cảnh báo khi syslog xuất hiện `logrotate: error` và `backup-sites: ERROR`; cảnh báo tỷ lệ 5xx trên LB (ví dụ >5% trong 2 phút).
- Zabbix agent không nên ghi log vào cùng phân vùng dễ đầy. Đặt log agent ra phân vùng riêng hoặc dùng buffer của proxy để không mất dữ liệu khi đầy.
- LB nên có health check **chạm vào đường dẫn động** (script PHP kiểm tra kết nối DB), để tự rút node khỏi pool thay vì tiếp tục đẩy request vào.

**Cấu hình**
- logrotate: sửa lỗi gõ sai, thêm `maxsize 1G` để xoay theo dung lượng. Cấu hình hệ thống nên đi qua config management và chạy `logrotate -d` trước khi áp dụng.
- Backup: ghi ra volume/host khác (NFS, S3, restic), kiểm tra dung lượng trống ≥ 1,3 lần bản trước trước khi chạy, xóa file dở dang khi lỗi, chỉ xóa bản cũ sau khi kiểm tra bản mới hợp lệ.
- Tách phân vùng cho `/var/log`, `/var/lib/mysql`, `/var/backups`, để một thành phần đầy không kéo cả hệ thống. Đặt `binlog_expire_logs_seconds` rõ ràng.
- PHP-FPM: tách pool riêng cho từng site (hiện 40 site dùng chung pool `www` nên một điểm nghẽn kéo sập tất cả). Đặt `request_terminate_timeout` và bật `slowlog`. Đo nhu cầu rồi mới chỉnh `pm.max_children`, không tăng mù quáng.
- Bot: dùng `limit_req` của nginx cho các URL phân trang sâu và chặn IP giả mạo UA.

## Hạn chế của báo cáo

- Tốc độ ghi `access.log` (~6 GB/ngày) cao hơn nhiều so với lưu lượng thấy trong log LB (~185 request/5 phút). Dữ liệu được cấp chưa đủ để giải thích. Cần mở `access.log.pre-incident` sau khi nén để xem vhost/IP nào ghi nhiều nhất.
- Log LB có `rt=60.5s` cho request 5xx đầu tiên ghi lúc 01:33:05, tức request bắt đầu sớm hơn lỗi `tar` khoảng 1 phút. Dữ liệu chưa đủ để giải thích chênh lệch này (độ lệch đồng hồ hoặc đĩa bắt đầu chậm ngay trước khi đầy).
- Các lệnh ở mục 5 được viết theo output trong `snapshot/`, chưa chạy trên máy thật.
