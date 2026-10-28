# Phần 3 – Đề B: Chặn / rate-limit bot crawl ở L4 (nftables) vs L7 (Nginx `limit_req`)

## 1. Bối cảnh và giả định

- Máy shared hosting chạy ~40 vhost (Nginx + PHP-FPM), nằm sau load balancer (như `lb-hcm-01` ở Phần 1).
- Bot crawl thường là HTTP hợp lệ, tốc độ vừa phải nhưng đến từ nhiều IP, nên **thứ cạn kiệt là PHP-FPM worker**, không phải băng thông. Đây không phải DDoS volumetric.
- **Giả định:** LB chuyển tiếp IP gốc qua `X-Forwarded-For` hoặc PROXY protocol; TLS kết thúc tại Nginx trên máy hosting. Nếu LB đã terminate TLS thì các nhận định về "L4 không thấy nội dung" vẫn đúng với kết nối LB → máy.

## 2. So sánh theo tiêu chí

| Tiêu chí | L4 – nftables | L7 – Nginx `limit_req` |
|---|---|---|
| Dữ liệu nhìn thấy | IP, port, trạng thái conntrack. Với HTTPS không thấy Host/URI/UA | Host, URI, method, UA, cookie, mã trạng thái |
| Chi phí xử lý | Drop trong kernel, trước khi Nginx/PHP phải làm gì | Sau accept + TLS handshake, nhưng từ chối rẻ hơn nhiều so với chạy PHP |
| Độ mịn theo khách hàng | Không phân biệt vhost (cùng :443) | Key theo IP, `$server_name`, URI; mỗi vhost một chính sách |
| Cơ chế | `meter`/dynamic set `limit rate`, `ct count over N` | Leaky bucket, `burst`, `nodelay`; trả 503 mặc định (đổi được qua `limit_req_status`) |
| Bộ nhớ | Set động, có thể gắn `timeout` | 1 MB zone ≈ 16.000 state 64 B hoặc 8.000 state 128 B; hết zone thì request bị từ chối hoặc limit mất tác dụng |
| Nguy cơ chặn nhầm | Cao: chặn cả IP với **mọi** site (CGNAT, văn phòng dùng chung IP) | Thấp hơn: có `burst`, giới hạn trong từng vhost, whitelist theo UA/path |
| Sau load balancer | Chỉ thấy IP của LB nếu LB proxy → vô dụng nếu không có PROXY protocol | Dùng `realip` (`set_real_ip_from` + `real_ip_header`) lấy IP khách thật |
| Thử nghiệm an toàn | Khó dry-run, sai là cắt kết nối | Có `limit_req_dry_run on` chỉ ghi log, không từ chối |

Dẫn chứng: tài liệu `ngx_http_limit_req_module` (nginx.org/r/limit_req_zone; `limit_req_dry_run`, kích thước state, `burst`/`nodelay`); nftables wiki – Meters (dynamic set, `limit rate`, `ct count`; lưu ý `ct count` không dùng được cùng set có `timeout`).

## 3. Khuyến nghị: mô hình phân lớp, **L7 là chính, L4 là lưới an toàn**

1. **L7 (chính):** mỗi vhost áp `limit_req` theo IP thật, đặt `burst` + `nodelay`, trả 429 thay vì 503 để bot tôn trọng backoff. Thêm một zone key `$server_name` với ngưỡng cao làm "cầu chì" để một site bị crawl không rút hết PHP-FPM của 39 site còn lại. Whitelist LB, monitoring và bot tìm kiếm đã xác thực rDNS.
2. **L4 (phụ):** (a) ngưỡng thô cho kết nối mới/IP (`ct count` hoặc `limit rate over`) để cắt các IP rõ ràng bất thường; (b) một set có `timeout` (30–60 phút) chứa IP vi phạm lặp lại ở L7, nạp tự động từ log `limiting requests, excess` (fail2ban hoặc script), để các IP này bị drop trong kernel và không tốn Nginx nữa.
3. **Lộ trình:** `limit_req_dry_run on` 1–2 ngày → xem log tìm false positive → bật thật. Với L4, bắt đầu bằng `counter` (không drop) rồi mới chuyển sang `drop`.

Cấu hình tham khảo (ngưỡng là điểm khởi đầu, cần chỉnh theo traffic thực):

```nginx
# http {}
set_real_ip_from <IP_LB>;  real_ip_header X-Forwarded-For;  real_ip_recursive on;
limit_req_zone $binary_remote_addr zone=perip:20m  rate=5r/s;
limit_req_zone $server_name        zone=pervhost:1m rate=100r/s;
limit_req_status 429;
# server {}
limit_req zone=perip   burst=20  nodelay;
limit_req zone=pervhost burst=200;
limit_req_dry_run on;   # bỏ dòng này khi chuyển sang enforce
```

```nft
table inet filter {
  set abuse_v4 { type ipv4_addr; flags timeout; timeout 30m; }   # nạp từ L7
  chain input {
    ip saddr @abuse_v4 tcp dport { 80, 443 } counter drop
  }
}
```

## 4. Rủi ro và đánh đổi

- **IP dùng chung (NAT/CGNAT):** giới hạn theo IP có thể phạt nhiều người dùng thật. Giảm nhẹ bằng `burst` hợp lý, ngưỡng L4 thô, thời gian ban ngắn.
- **Bot phân tán nhiều IP (botnet, proxy xoay):** rate-limit theo IP kém hiệu quả; cần thêm tín hiệu L7 (UA, path như `/?s=`, `/wp-json`, cookie) hoặc WAF/CDN phía trước. Đây là giới hạn của cả hai tầng, không phải của riêng tầng nào.
- **Zone cạn bộ nhớ:** nếu zone đầy, limit mất tác dụng đúng lúc cần nhất → cần giám sát số lần lỗi zone trong `error.log`, tính dung lượng theo số IP duy nhất cao điểm (ví dụ zone 20 MB ≈ 160.000–320.000 state theo tài liệu).
- **Chi phí vận hành:** L7 đặt ở từng vhost (sinh cấu hình bằng template), L4 là chính sách chung toàn máy; sai ở L4 ảnh hưởng toàn bộ khách, nên thay đổi phải qua review và rollback nhanh.

## 5. Phạm vi và hạn chế

Chưa có benchmark lab cá nhân; các nhận định về hiệu năng dựa trên tài liệu chính thức và kiến trúc xử lý gói, **chưa đo thực tế**. Nếu có thêm thời gian: dựng lab 1 VM Nginx + PHP-FPM, dùng `wrk`/`hey` giả lập crawl (nhiều IP bằng network namespace), so sánh số PHP-FPM worker bận, p95 latency và CPU giữa 3 cấu hình: không giới hạn / chỉ L7 / L7 + L4.
