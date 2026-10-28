# Phần 3 – Đề B: Chặn / rate-limit bot crawl: L4 (nftables) vs L7 (Nginx `limit_req`)

**Giả định:** máy shared hosting ~40 vhost (Nginx + PHP-FPM) nằm sau load balancer, LB truyền IP gốc qua `X-Forwarded-For` hoặc PROXY protocol. Bot crawl là HTTP hợp lệ từ nhiều IP; thứ cạn kiệt là PHP-FPM worker, không phải băng thông.

## 1. So sánh theo tiêu chí

| Tiêu chí | L4 – nftables | L7 – Nginx `limit_req` |
|---|---|---|
| Dữ liệu nhìn thấy | IP, port, conntrack; với HTTPS không thấy Host/URI/UA *(phân tích)* | Host, URI, UA, cookie; key `limit_req_zone` có thể là biến hoặc tổ hợp biến [1] |
| Độ mịn theo khách hàng | Không phân biệt vhost (cùng :443, nội dung mã hóa) *(phân tích)* | Key theo `$binary_remote_addr`, `$server_name`; khai báo được nhiều `limit_req` cùng lúc [1] |
| Cơ chế | `meter`/dynamic set + `limit rate`; `ct count over N` (từ 4.18) [2] | Leaky bucket, `burst`, `nodelay`; trả 503 mặc định, đổi bằng `limit_req_status` [1] |
| Bộ nhớ | Dynamic set, gắn được `timeout`; `ct count` không dùng được với set có `timeout` [2] | State 64 B (32-bit) / 128 B (64-bit): 1 MB ≈ 16.000 / 8.000 state; hết chỗ thì xóa state ít dùng nhất, vẫn không đủ thì request bị lỗi [1] |
| Chặn nhầm | Cao: chặn IP khỏi **mọi** site (CGNAT, IP dùng chung) *(phân tích)* | Thấp hơn: có `burst`, phạm vi theo vhost *(phân tích)* |
| Sau load balancer | Chỉ thấy IP của LB nếu không truyền IP gốc *(phân tích)* | `set_real_ip_from` + `real_ip_header`; cần build `--with-http_realip_module` [3] |
| Thử nghiệm an toàn | Không có dry-run riêng; mô phỏng bằng rule `counter` không `drop` *(phân tích)* | `limit_req_dry_run on` (từ 1.17.1): không giới hạn nhưng vẫn đếm [1] |

Số trong ngoặc vuông là nguồn ở mục 4; *(phân tích)* là suy luận của người viết, chưa đo.

## 2. Khuyến nghị: L7 là lớp chính, L4 là lưới phụ

1. **L7:** mỗi vhost áp `limit_req` theo IP thật (`burst` + `nodelay`, trả 429 để phân biệt với lỗi backend trong log), cộng một zone key `$server_name` ngưỡng cao làm "cầu chì" để một site bị crawl không rút hết PHP-FPM của các site còn lại [1]. Whitelist LB, monitoring, bot tìm kiếm đã xác thực rDNS.
2. **L4:** set có `timeout` 30–60 phút chứa IP vi phạm lặp lại, nạp tự động từ access log có `$limit_req_status` = `REJECTED` (nginx ≥ 1.17.6 [1]; giai đoạn dry-run giá trị là `REJECTED_DRY_RUN`) bằng fail2ban hoặc script, để IP đó bị drop trong kernel, không tốn Nginx nữa.
3. **Lộ trình:** `limit_req_dry_run on` 1–2 ngày xem false positive rồi mới enforce; L4 chạy `counter` trước, `drop` sau.

```nginx
limit_req_zone $binary_remote_addr zone=perip:20m   rate=5r/s;
limit_req_zone $server_name        zone=pervhost:1m rate=100r/s;
limit_req zone=perip burst=20 nodelay;  limit_req zone=pervhost burst=200;
limit_req_status 429;  limit_req_dry_run on;   # ngưỡng chỉ là điểm khởi đầu
```

## 3. Rủi ro và đánh đổi

- **IP dùng chung (NAT/CGNAT):** limit theo IP có thể phạt người dùng thật; giảm nhẹ bằng `burst`, ngưỡng L4 thô, thời gian ban ngắn.
- **Botnet / proxy xoay IP:** limit theo IP kém hiệu quả ở cả hai tầng; cần thêm tín hiệu L7 (UA, path) hoặc WAF/CDN phía trước.
- **Zone cạn:** state ít dùng nhất bị xóa hoặc request lỗi [1] (hệ quả là bộ đếm của IP đó bị reset *(phân tích)*); tính theo số IP duy nhất cao điểm (64-bit: 20 MB ≈ 160.000 state) và theo dõi `error.log`.
- **Kế thừa cấu hình:** `limit_req` chỉ kế thừa khi cấp hiện tại không khai báo `limit_req` nào, nên `limit_req` do khách tự thêm vào `location` sẽ vô hiệu chính sách chung [1].

## 4. Kiểm chứng lab và tài liệu

**Môi trường lab:** VPS 1 vCPU, Nginx 1.31.8 + PHP-FPM 8.2 (`pm.max_children=5`), endpoint `usleep(50000)`. Tải tạo từ máy local bằng `wrk -t4 -c100 -d30s --latency` qua mạng công cộng; P99 bao gồm RTT nên chỉ dùng so sánh tương đối giữa các kịch bản.

| Kịch bản | Requests/sec | Latency Avg | Latency P99 | Non-2xx | Ghi chú |
|---|---|---|---|---|---|
| Baseline (không `limit_req`) | 95.52 | 1.03s | 1.73s | 0 | PHP-FPM nghẽn 5/5 worker, request xếp hàng chờ |
| Dry-run (`limit_req_dry_run on`) | 88.58 | 1.07s | 1.83s | 0 | Log ghi nhận 883/888 request (99.4%) `REJECTED_DRY_RUN` — không chặn thật, chỉ đếm |
| Enforce (trả 429) | 11,758.74 | 26.83ms | 222.32ms | 352,681 / 352,986 (99.9%) | Request vượt ngưỡng bị chặn ngay tại Nginx, không chiếm worker PHP-FPM |

**Nhận xét:** Ở baseline, PHP-FPM bị chiếm hết 5/5 worker, P99 lên tới 1.73s do request phải xếp hàng chờ. Dry-run giữ nguyên throughput và latency so với baseline (88.58 req/s, P99 1.83s), nhưng log ghi nhận 883/888 request (99.4%) bị đánh dấu `REJECTED_DRY_RUN` — xác nhận cơ chế dry-run chỉ đếm mà không can thiệp traffic, phù hợp để chạy 1–2 ngày quan sát false-positive trước khi enforce. Khi enforce, Nginx trả 429 cho 99.9% request vượt ngưỡng ngay tại tầng L7, không gọi PHP-FPM: P99 giảm từ 1.73s xuống 222ms (~8 lần), latency trung bình giảm từ ~1s xuống 27ms. Throughput 11,758 req/s ở đây là **tốc độ từ chối** request, không phải năng lực phục vụ request hữu ích — điểm mấu chốt là Nginx chặn được lượng lớn request mà **không tốn worker PHP-FPM nào**, đúng với khuyến nghị ở mục 2: L7 là lớp chính bảo vệ backend, L4 là lưới phụ.

*Hạn chế:* lab nhỏ (1 vCPU, 1 endpoint, tải từ xa qua Internet), chưa đo tách biệt ảnh hưởng của RTT mạng; số liệu đủ để so sánh xu hướng, không đại diện tuyệt đối cho môi trường 40 vhost thật. Cấu hình `burst=5`, `rate=10r/s` chỉ là điểm khởi đầu, cần tinh chỉnh theo lưu lượng thật và theo dõi false-positive bằng `limit_req_dry_run` trước khi enforce.

1. https://nginx.org/en/docs/http/ngx_http_limit_req_module.html
2. https://wiki.nftables.org/wiki-nftables/index.php/Meters
3. https://nginx.org/en/docs/http/ngx_http_realip_module.html
