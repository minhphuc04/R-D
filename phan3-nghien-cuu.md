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

## 4. Hạn chế và tài liệu

Chưa có benchmark lab; các nhận định hiệu năng chưa đo. Nếu có thêm thời gian: lab Nginx + PHP-FPM, giả lập crawl bằng `wrk`/`hey`, so sánh PHP-FPM worker bận, p95 latency, CPU giữa: không giới hạn / chỉ L7 / L7 + L4.

1. https://nginx.org/en/docs/http/ngx_http_limit_req_module.html
2. https://wiki.nftables.org/wiki-nftables/index.php/Meters
3. https://nginx.org/en/docs/http/ngx_http_realip_module.html

(Tra ngày 28/10/2026; cần đối chiếu phiên bản nginx/kernel thực tế trên máy hosting.)
