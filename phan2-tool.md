# Proxmox Cluster Load Balancer

Tool phân tích snapshot cluster Proxmox VE, phát hiện node quá tải, và đề xuất
kế hoạch live-migrate VM để cân bằng tài nguyên.

Viết bằng **Python 3.10+**, **không cần dependency ngoài** (chỉ stdlib).

## Cài đặt

```bash
cd phan2-tool
python3 --version   # cần >= 3.10
```

## Cách chạy

```bash
# Output dạng bảng (mặc định)
python3 proxmox_balance.py ../phan2-proxmox/cluster-pve-hcm-c3.json

# Output JSON (để pipe sang tool/script tự động hóa khác)
python3 proxmox_balance.py ../phan2-proxmox/cluster-pve-hcm-c3.json --format json

# Tùy chỉnh ngưỡng
python3 proxmox_balance.py ../phan2-proxmox/cluster-pve-hcm-c3.json --cpu-threshold 85 --mem-threshold 90 --overcommit-threshold 5

# Chạy unit test
python3 -m unittest test_proxmox_balance -v
```

## Ngưỡng đánh giá & giả định

| Ngưỡng | Mặc định | Lý do |
|---|---|---|
| CPU thực dùng | **80%** | Trên 80% dễ gây timeout, ảnh hưởng VM khác trên cùng node |
| RAM thực dùng | **85%** | Buffer cho kernel cache & spike đột ngột |
| Overcommit vCPU | **4.0x** | Tỷ lệ phổ biến cho workload hosting; vượt 4x dễ gây tranh chấp CPU |

Tất cả ngưỡng đều **override được qua CLI**.

### Giả định xử lý dữ liệu

Đề bài không nói rõ một số trường hợp biên. Tool áp dụng các giả định sau
(đều được log warning rõ ràng, không crash):

1. **`cpu_usage` ngoài khoảng [0, 1]:**
   - Nếu `1 < cpu_usage <= 100`: giả định đơn vị là **phần trăm** -> chia 100.
   - Nếu `> 100`: clamp về `1.0`.
   - *Lý do:* data thực tế có `31.0`, `98.0`, `20.8`.

2. **`mem_used=None` hoặc thiếu:** bỏ qua VM đó khi tính tải, log warning.

3. **`mem_used > mem_max`:** clamp `mem_used` về `mem_max`, log warning.

4. **`vcpus <= 0`:** bỏ qua VM, log warning.

5. **VM `stopped`:** không tính vào tải thực (CPU/RAM) và không đề xuất migrate.
   `vcpu_allocated` cũng chỉ tính VM running.

6. **Node `offline`:** không nhận migrate, không tính vào phân tích quá tải.

7. **Node có `mem_total < 1GB` hoặc `cpu_cores <= 0`:** coi là **dữ liệu nghi
   hỏng** (ví dụ `hcm-c3-n06` có `mem_total=262144` ~256KB). Đánh dấu
   `is_eligible_target=False`, hiển thị `n/a`, không đưa vào kế hoạch migrate.

8. **VM trỏ tới node không có trong danh sách `nodes[]`:** log warning (orphan node).

9. **Ngưỡng an toàn khi migrate:** VM chỉ được chuyển sang node đích nếu sau khi
   nhận, node đích vẫn thỏa **cả 3 ngưỡng** (CPU, RAM, overcommit). Nếu không
   tìm được node đích phù hợp, VM **không được migrate** và node nguồn vẫn giữ
   trạng thái quá tải (có warning rõ).
10. **Node mem_total nghi sai đơn vị:** nếu `mem_total < 1GB` và là bội số của
1024, tool cảnh báo thêm giả thuyết "nếu đơn vị là MiB thì X MiB = Y GiB"
(ví dụ `hcm-c3-n06`: 262144 MiB = 256 GiB).
## Ví dụ output thực tế

### 1. Output dạng bảng (ngưỡng mặc định 80/85/4)

Lệnh chạy:

```bash
python3 proxmox_balance.py ../phan2-proxmox/cluster-pve-hcm-c3.json
```

Output:

```text
=== TẢI TỪNG NODE (TRƯỚC MIGRATE) ===
NODE           STATUS         CPU    CPU%                 MEM    MEM%  vCPU    OC  TÌNH TRẠNG
---------------------------------------------------------------------------------------------
hcm-c3-n01     online     55.0/64   85.9%   511.27GB/549.76GB   93.0%   110 1.72x  QUÁ TẢI      CPU 85.9% > 80% RAM 93.0% > 85%
hcm-c3-n02     online     27.9/64   43.6%   328.00GB/549.76GB   59.7%    92 1.44x  ok           
hcm-c3-n03     online     18.2/48   38.0%   193.79GB/412.32GB   47.0%    80 1.67x  ok           
hcm-c3-n04     offline     0.0/48    0.0%     0.00GB/412.32GB    0.0%     0 0.00x  offline      
hcm-c3-n05     online     44.8/64   70.0%   395.07GB/549.76GB   71.9%    60 0.94x  ok           
hcm-c3-n06     online         n/a     n/a                 n/a     n/a    94   n/a  DỮ LIỆU HỎNG mem_total=262144 bytes quá nhỏ (<1GB) - dữ liệu nghi hỏng (nếu đơn vị là MiB thì 262144 MiB = 256 GiB -> có thể exporter xuất sai đơn vị)

=== CẢNH BÁO DỮ LIỆU ===
  [!] node hcm-c3-n04: node offline - không thể nhận VM
  [!] node hcm-c3-n06: mem_total=262144 bytes quá nhỏ (<1GB) - dữ liệu nghi hỏng (nếu đơn vị là MiB thì 262144 MiB = 256 GiB -> có thể exporter xuất sai đơn vị)
  [!] vmid=2067 (wp-host-992): cpu_usage=31.0 ngoài [0,1] -> giả định đơn vị %, chia 100
  [!] vmid=2073 (vps-nvme-430): cpu_usage=98.0 ngoài [0,1] -> giả định đơn vị %, chia 100
  [!] vmid=2065 (cloud-std-682): cpu_usage=20.8 ngoài [0,1] -> giả định đơn vị %, chia 100
  [!] vmid=2199 (wp-host-377): mem_used=None, bỏ qua
  [!] vmid=2390 (vps-nvme-512): node='hcm-c3-n07' không có trong danh sách nodes

=== KẾ HOẠCH MIGRATE ===
  Đề xuất 1 migration:
  - vmid=2054 (cloud-std-657) hcm-c3-n01 -> hcm-c3-n03 [8vCPU, 66.53GB] - giải phóng 66.5GB khỏi node quá tải

=== TẢI SAU MIGRATE (dự kiến) ===
  hcm-c3-n01: CPU 74.1% | RAM 80.9% | OC 1.59x
  hcm-c3-n02: CPU 43.6% | RAM 59.7% | OC 1.44x
  hcm-c3-n03: CPU 53.8% | RAM 63.1% | OC 1.83x
  hcm-c3-n05: CPU 70.0% | RAM 71.9% | OC 0.94x
  hcm-c3-n06: n/a (dữ liệu nghi hỏng, cần kiểm tra thủ công)

```

### 2. Output dạng bảng (ngưỡng chặt 70/70/2)

Lệnh chạy:

```bash
python3 proxmox_balance.py ../phan2-proxmox/cluster-pve-hcm-c3.json \
  --cpu-threshold 70 --mem-threshold 70 --overcommit-threshold 2
```

Output:

```text
=== TẢI TỪNG NODE (TRƯỚC MIGRATE) ===
NODE           STATUS         CPU    CPU%                 MEM    MEM%  vCPU    OC  TÌNH TRẠNG
---------------------------------------------------------------------------------------------
hcm-c3-n01     online     55.0/64   85.9%   511.27GB/549.76GB   93.0%   110 1.72x  QUÁ TẢI      CPU 85.9% > 70% RAM 93.0% > 70%
hcm-c3-n02     online     27.9/64   43.6%   328.00GB/549.76GB   59.7%    92 1.44x  ok           
hcm-c3-n03     online     18.2/48   38.0%   193.79GB/412.32GB   47.0%    80 1.67x  ok           
hcm-c3-n04     offline     0.0/48    0.0%     0.00GB/412.32GB    0.0%     0 0.00x  offline      
hcm-c3-n05     online     44.8/64   70.0%   395.07GB/549.76GB   71.9%    60 0.94x  QUÁ TẢI      RAM 71.9% > 70%
hcm-c3-n06     online         n/a     n/a                 n/a     n/a    94   n/a  DỮ LIỆU HỎNG mem_total=262144 bytes quá nhỏ (<1GB) - dữ liệu nghi hỏng (nếu đơn vị là MiB thì 262144 MiB = 256 GiB -> có thể exporter xuất sai đơn vị)

=== CẢNH BÁO DỮ LIỆU ===
  [!] node hcm-c3-n04: node offline - không thể nhận VM
  [!] node hcm-c3-n06: mem_total=262144 bytes quá nhỏ (<1GB) - dữ liệu nghi hỏng (nếu đơn vị là MiB thì 262144 MiB = 256 GiB -> có thể exporter xuất sai đơn vị)
  [!] vmid=2067 (wp-host-992): cpu_usage=31.0 ngoài [0,1] -> giả định đơn vị %, chia 100
  [!] vmid=2073 (vps-nvme-430): cpu_usage=98.0 ngoài [0,1] -> giả định đơn vị %, chia 100
  [!] vmid=2065 (cloud-std-682): cpu_usage=20.8 ngoài [0,1] -> giả định đơn vị %, chia 100
  [!] vmid=2199 (wp-host-377): mem_used=None, bỏ qua
  [!] vmid=2390 (vps-nvme-512): node='hcm-c3-n07' không có trong danh sách nodes
  [!] node hcm-c3-n05: quá tải (RAM 71.9% > 70%) nhưng không tìm được node đích phù hợp để migrate

=== KẾ HOẠCH MIGRATE ===
  Đề xuất 3 migration:
  - vmid=2054 (cloud-std-657) hcm-c3-n01 -> hcm-c3-n03 [8vCPU, 66.53GB] - giải phóng 66.5GB khỏi node quá tải
  - vmid=2027 (s3-gw-616) hcm-c3-n01 -> hcm-c3-n02 [2vCPU, 45.80GB] - giải phóng 45.8GB khỏi node quá tải
  - vmid=2021 (cloud-std-520) hcm-c3-n01 -> hcm-c3-n03 [4vCPU, 27.48GB] - giải phóng 27.5GB khỏi node quá tải

=== TẢI SAU MIGRATE (dự kiến) ===
  hcm-c3-n01: CPU 66.1% | RAM 67.6% | OC 1.50x
  hcm-c3-n02: CPU 46.7% | RAM 68.0% | OC 1.47x
  hcm-c3-n03: CPU 60.3% | RAM 69.8% | OC 1.92x
  hcm-c3-n05: CPU 70.0% | RAM 71.9% | OC 0.94x
  hcm-c3-n06: n/a (dữ liệu nghi hỏng, cần kiểm tra thủ công)


```

### 3. Output dạng JSON

Lệnh chạy:

```bash
python3 proxmox_balance.py ../phan2-proxmox/cluster-pve-hcm-c3.json --format json
```

Output (rút gọn — node đầu tiên và migrations):

```{
  "summary": {
    "total_nodes": 6,
    "online_nodes": 5,
    "overloaded_nodes": [
      "hcm-c3-n01"
    ],
    "migration_count": 1,
    "warning_count": 8
  },
  "nodes_before": [
    {
      "name": "hcm-c3-n01",
      "status": "online",
      "cpu_cores": 64,
      "mem_total": 549755813888,
      "cpu_used_cores": 55.0,
      "cpu_usage_pct": 85.94,
      "mem_used": 511267456789,
      "mem_usage_pct": 93.0,
      "vcpu_allocated": 110,
      "overcommit_ratio": 1.719,
      "is_overloaded": true,
      "is_eligible_target": true,
      "overload_reasons": [
        "CPU 85.9% > 80%",
        "RAM 93.0% > 85%"
      ],
      "warnings": []
    }
  ],
  "nodes_after": [
    {
      "name": "hcm-c3-n01",
      "status": "online",
      "cpu_cores": 64,
      "mem_total": 549755813888,
      "cpu_used_cores": 47.416,
      "cpu_usage_pct": 74.09,
      "mem_used": 444743779097,
      "mem_usage_pct": 80.9,
      "vcpu_allocated": 102,
      "overcommit_ratio": 1.594,
      "is_overloaded": false,
      "is_eligible_target": true,
      "overload_reasons": [],
      "warnings": []
    }
  ],
  "migrations": [
    {
      "vmid": 2054,
      "name": "cloud-std-657",
      "from": "hcm-c3-n01",
      "to": "hcm-c3-n03",
      "vcpus": 8,
      "mem_used": 66529127812,
      "reason": "giải phóng 66.5GB khỏi node quá tải"
    }
  ],
  "warnings": [
    "node hcm-c3-n04: node offline - không thể nhận VM",
    "node hcm-c3-n06: mem_total=262144 bytes quá nhỏ (<1GB) - dữ liệu nghi hỏng (nếu đơn vị là MiB thì 262144 MiB = 256 GiB -> có thể exporter xuất sai đơn vị)",
    "vmid=2067 (wp-host-992): cpu_usage=31.0 ngoài [0,1] -> giả định đơn vị %, chia 100",
    "vmid=2073 (vps-nvme-430): cpu_usage=98.0 ngoài [0,1] -> giả định đơn vị %, chia 100",
    "vmid=2065 (cloud-std-682): cpu_usage=20.8 ngoài [0,1] -> giả định đơn vị %, chia 100",
    "vmid=2199 (wp-host-377): mem_used=None, bỏ qua",
    "vmid=2390 (vps-nvme-512): node='hcm-c3-n07' không có trong danh sách nodes"
  ]
}
```

## Unit test

Lệnh chạy:

```bash
python3 -m unittest test_proxmox_balance -v
```

Output:

```text
test_high_cpu_marks_overloaded (test_proxmox_balance.TestAnalyze.test_high_cpu_marks_overloaded) ... ok
test_high_mem_marks_overloaded (test_proxmox_balance.TestAnalyze.test_high_mem_marks_overloaded) ... ok
test_overcommit_marks_overloaded (test_proxmox_balance.TestAnalyze.test_overcommit_marks_overloaded) ... ok
test_tiny_mem_node_not_analyzed (test_proxmox_balance.TestAnalyze.test_tiny_mem_node_not_analyzed)
Node dữ liệu hỏng không bị đánh dấu quá tải (tránh nhiễu). ... ok
test_json_structure (test_proxmox_balance.TestJsonOutput.test_json_structure) ... ok
test_cpu_usage_above_100_clamped (test_proxmox_balance.TestLoadSnapshot.test_cpu_usage_above_100_clamped) ... ok
test_cpu_usage_percent_converted (test_proxmox_balance.TestLoadSnapshot.test_cpu_usage_percent_converted) ... ok
test_invalid_json_raises (test_proxmox_balance.TestLoadSnapshot.test_invalid_json_raises) ... ok
test_load_valid (test_proxmox_balance.TestLoadSnapshot.test_load_valid) ... ok
test_mem_used_clamped_to_max (test_proxmox_balance.TestLoadSnapshot.test_mem_used_clamped_to_max) ... ok
test_missing_file_raises (test_proxmox_balance.TestLoadSnapshot.test_missing_file_raises) ... ok
test_offline_node_not_eligible (test_proxmox_balance.TestLoadSnapshot.test_offline_node_not_eligible) ... ok
test_orphan_node_warning (test_proxmox_balance.TestLoadSnapshot.test_orphan_node_warning) ... ok
test_skip_vm_invalid_vcpus (test_proxmox_balance.TestLoadSnapshot.test_skip_vm_invalid_vcpus) ... ok
test_skip_vm_missing_mem_used (test_proxmox_balance.TestLoadSnapshot.test_skip_vm_missing_mem_used) ... ok
test_stopped_vm_not_counted_in_load (test_proxmox_balance.TestLoadSnapshot.test_stopped_vm_not_counted_in_load) ... ok
test_tiny_mem_node_not_eligible (test_proxmox_balance.TestLoadSnapshot.test_tiny_mem_node_not_eligible)
Node có mem_total < 1GB (dữ liệu hỏng) không được làm target. ... ok
test_migration_does_not_overload_target (test_proxmox_balance.TestPlanMigrations.test_migration_does_not_overload_target) ... ok
test_migration_success (test_proxmox_balance.TestPlanMigrations.test_migration_success) ... ok
test_offline_node_never_target (test_proxmox_balance.TestPlanMigrations.test_offline_node_never_target) ... ok
test_overload_reasons_kept_when_no_target (test_proxmox_balance.TestPlanMigrations.test_overload_reasons_kept_when_no_target)
Nếu không migrate được, reasons của node quá tải phải giữ nguyên. ... ok
test_overload_reasons_recomputed (test_proxmox_balance.TestPlanMigrations.test_overload_reasons_recomputed)
Sau migrate thành công, reasons của node nguồn phải rỗng. ... ok
test_tiny_mem_node_never_target (test_proxmox_balance.TestPlanMigrations.test_tiny_mem_node_never_target) ... ok
test_tiny_mem_node_not_in_migration_source (test_proxmox_balance.TestPlanMigrations.test_tiny_mem_node_not_in_migration_source)
Node hỏng không được sinh migration (dù có VM). ... ok
test_accepts_within_thresholds (test_proxmox_balance.TestWouldStayHealthy.test_accepts_within_thresholds) ... ok
test_rejects_when_mem_over (test_proxmox_balance.TestWouldStayHealthy.test_rejects_when_mem_over) ... ok

----------------------------------------------------------------------
Ran 26 tests in 0.011s

OK
```

26 test case cover:

- Parse JSON hợp lệ / hỏng / thiếu file
- Bỏ qua VM thiếu `mem_used`, `vcpus=0`, thiếu `vmid`
- Chuyển đổi `cpu_usage` % -> tỷ lệ, clamp `> 100`
- Clamp `mem_used > mem_max`
- Phát hiện orphan node (VM trỏ tới node không tồn tại)
- VM `stopped` không tính vào tải
- Node `mem_total < 1GB` / `offline` không eligible
- Phát hiện quá tải theo CPU / RAM / overcommit
- Migrate thành công, không đẩy node đích vào quá tải
- `overload_reasons` được recalc sau migrate (cả trường hợp migrate được
  và không migrate được)
- Cấu trúc JSON output (null cho node không eligible)

### Test xử lý ngoại lệ an toàn

**Case 1 — File không tồn tại:**

```bash
python3 proxmox_balance.py /tmp/khong-ton-tai.json; echo "exit code: $?"
```

Output:

```text
LỖI: không tìm thấy file /tmp/khong-ton-tai.json
exit code: 1
```

**Case 2 — File JSON hỏng:**

```bash
echo "{not valid json" > /tmp/bad.json
python3 proxmox_balance.py /tmp/bad.json; echo "exit code: $?"
```

Output:

```text
LỖI: file JSON không hợp lệ (/tmp/bad.json): Expecting property name enclosed in double quotes: line 1 column 2 (char 1)
exit code: 1
```

**Case 3 — JSON thiếu trường (không crash, có warning):**

```bash
python3 proxmox_balance.py /tmp/partial.json; echo "exit code: $?"
```

Output:

```text
=== TẢI TỪNG NODE (TRƯỚC MIGRATE) ===
NODE           STATUS         CPU    CPU%                 MEM    MEM%  vCPU    OC  TÌNH TRẠNG
---------------------------------------------------------------------------------------------
n1             online       1.0/8   12.5%      2.00GB/16.00GB   12.5%     2 0.25x  ok           

=== CẢNH BÁO DỮ LIỆU ===
  [!] vmid=2 (no_mem_used): mem_used=None, bỏ qua
  [!] vmid=3 (bad_vcpus): vcpus=0 không hợp lệ, bỏ qua
  [!] vms[3]: thiếu/không hợp lệ 'vmid', bỏ qua

=== KẾ HOẠCH MIGRATE ===
  Không cần migrate - cluster đã cân bằng.

=== TẢI SAU MIGRATE (dự kiến) ===
  n1: CPU 12.5% | RAM 12.5% | OC 0.25x
exit code: 0
```

## Thuật toán migrate

Tool dùng **greedy worst-fit**:

1. Sắp xếp node quá tải theo `mem_usage_pct` giảm dần (nặng nhất trước).
2. Với mỗi node quá tải, sắp xếp VM theo `mem_used` giảm dần.
3. Với từng VM, tìm node đích thỏa **tất cả**:
   - Đang online, `is_eligible_target=True`
   - Không quá tải hiện tại
   - Sau khi nhận VM vẫn thỏa cả 3 ngưỡng (`_would_stay_healthy`)
4. Chọn node đích có `mem_usage_pct` **thấp nhất** (worst-fit - dàn đều tải).
5. Lặp đến khi node nguồn hết quá tải hoặc không còn VM nào chuyển được.

**Hạn chế đã biết:** greedy không đảm bảo tối ưu toàn cục. Với cluster nhỏ
(<= 10 node) như bài toán này, greedy cho kết quả tốt và dễ hiểu. Nếu cần tối ưu
chặt hơn, có thể thay bằng ILP / min-cost flow.

## Cấu trúc thư mục

```
phan2-tool/
├── proxmox_balance.py        # tool chính
├── test_proxmox_balance.py   # 26 unit test
└── README.md                 
```

## Hướng phát triển (ngoài phạm vi bài test)

- Đọc trực tiếp từ Proxmox API (`/api2/json/cluster/resources`) thay vì file JSON.
- Chạy định kỳ qua cron/systemd timer, gửi alert qua Prometheus Alertmanager.
- Thêm ràng buộc anti-affinity (không đặt 2 VM cùng service trên 1 node).
- Tối ưu bằng ILP cho cluster lớn (> 10 node).
