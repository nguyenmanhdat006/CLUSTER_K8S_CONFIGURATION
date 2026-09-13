# Falco

Theo dõi hành vi lúc chạy ở cả tầng máy chủ (host) và tầng container, thay cho
`auditd` trong lộ trình Mục 5 của tài liệu giám sát.

Application Argo CD: `falco` (khai báo tại `argocd/falco.yaml`).

---

## Falco là gì, khác auditd ở đâu

`auditd` và audit log của API server ghi lại **lệnh gọi** — ai gọi API nào, ai sửa file
nào. Falco quan sát ở tầng thấp hơn: **mọi syscall** mà bất kỳ tiến trình nào thực hiện,
kể cả bên trong container. Đây là lớp mà audit log Kubernetes không chạm tới, vì audit
log chỉ thấy "có người `kubectl exec` vào pod", còn Falco thấy "bên trong pod đó, một
shell vừa được mở".

| | auditd / audit log K8s | Falco |
|---|---|---|
| Nhìn vào | Log tĩnh, đọc sau | Luồng syscall, thời gian thực |
| Biết gì | Ai gọi API nào | Tiến trình nào làm gì bên trong container |
| Ví dụ | Ai đã `kubectl exec` vào pod | Bên trong pod đó, ai vừa mở `/bin/sh` |

**Quan trọng:** Falco không xoá bỏ việc phải bật audit log API server (Mục 5.4 của tài
liệu). Nếu muốn Falco giám sát cả tầng cụm K8s qua plugin `k8s-audit`, plugin đó vẫn cần
đọc dữ liệu từ audit log API server — bước rủi ro cao nhất trong Mục 5 không biến mất,
chỉ đổi nơi tiêu thụ dữ liệu.

---

## Ba tầng Mục 5, Falco đứng ở đâu

```
Tầng host     → Falco (thay auditd)
Tầng cụm K8s  → vẫn phải bật audit log API server → Falco đọc qua plugin k8s-audit
Tầng ứng dụng → bảng audit_events trong database, không đổi, không liên quan Falco
```

Falco không biết gì về logic nghiệp vụ ứng dụng — không thấy "ai vừa đổi giá sản phẩm".
Tầng ứng dụng vẫn phải làm riêng theo Mục 5.5.

---

## Kiến trúc

Chạy dưới dạng **DaemonSet** — một pod Falco trên mỗi node. Bắt buộc phải vậy vì syscall
là chuyện của từng máy, không gom về một chỗ được.

```
Node 1: kernel/eBPF → Falco daemon → so luật → xuất event
Node 2: kernel/eBPF → Falco daemon → so luật → xuất event
Node 3: kernel/eBPF → Falco daemon → so luật → xuất event
                                          ↓
                            falco-exporter → metric Prometheus
```

---

## Chi phí tài nguyên — đọc trước khi bật thêm rule

| Hạng mục | Mức |
|---|---|
| RAM mỗi node | 200–400 MB |
| CPU | Liên tục, tăng khi container hoạt động nhiều |
| Yêu cầu kernel | eBPF hiện đại cần kernel ≥ 5.8 |

Cụm này từng gặp sự cố nghiêm trọng vì thiếu giới hạn bộ nhớ cho Prometheus (xem nhật ký
giai đoạn 2 và 3). Vì vậy `resources.limits.memory` trong cấu hình Falco là bắt buộc,
không phải tuỳ chọn — chặn cứng để Falco không lặp lại kiểu sự cố đó.

Kiểm tra RAM khả dụng trên cả ba node **trước** khi sync Application:

```bash
for h in 192.168.253.111 192.168.253.112 192.168.253.113; do
  echo "=== $h ==="
  ssh nguyendat@$h 'free -h'
done
```

---

## Kiểm tra kernel trước khi chọn driver

```bash
uname -r
```

| Kết quả | Driver dùng |
|---|---|
| ≥ 5.8 | `modern_ebpf` (mặc định trong values) |
| < 5.8 | `ebpf` — sửa `driver.kind` trong `argocd/falco.yaml` |

---

## Cài đặt

### 1. Xác nhận phiên bản chart thật trước khi apply

Chart Falco đã lên tới nhánh version 9.x, khác hẳn các bản 3.x/4.x cũ hơn — cấu trúc
`values.yaml` có thể đã đổi giữa các major version (xem `BREAKING-CHANGES.md` của repo
chart nếu nâng cấp sau này).

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts 2>/dev/null
helm repo update
helm search repo falcosecurity/falco --versions | head -5
```

Đối chiếu số trong `targetRevision` của `argocd/falco.yaml` với kết quả lệnh trên trước
khi apply — không lấy nguyên số cũ nếu đã có bản mới hơn.

### 2. Kiểm tra values khớp đúng schema chart thật

```bash
helm show values falcosecurity/falco --version <version-vua-tim> > /tmp/falco-values-goc.yaml
grep -A5 "^driver:" /tmp/falco-values-goc.yaml
grep -A10 "^falco:" /tmp/falco-values-goc.yaml | head -20
```

So với khối `helm.values` trong `argocd/falco.yaml`. Nếu tên trường khác, sửa lại theo
đúng schema thật thay vì giữ nguyên tên đoán trước.

### 3. Apply

```bash
kubectl apply -f argocd/falco.yaml
```

---

## Xác minh

```bash
kubectl -n argocd get app falco
kubectl -n falco get pod -o wide
```

Phải thấy đúng 3 pod, mỗi node một cái, trạng thái `Running`.

```bash
kubectl -n falco logs -l app.kubernetes.io/name=falco --tail=30
```

Tìm dòng xác nhận driver nạp thành công. Lỗi thường gặp ở bước này là kernel không hỗ
trợ driver đã chọn — quay lại bước kiểm tra `uname -r`.

### RAM sau khi cài

```bash
kubectl -n falco top pod
free -h    # chạy trên cả ba node
```

Đối chiếu với số đo trước khi cài. Nếu vượt quá dự kiến đáng kể, xem lại
`resources.limits` và cân nhắc tắt bớt rule mặc định.

---

## Cấu hình đang dùng

```yaml
driver:
  kind: modern_ebpf   # đổi thành "ebpf" nếu kernel < 5.8

tty: true

falco:
  json_output: true
  json_include_output_property: true

falcosidekick:
  enabled: false        # chưa định tuyến cảnh báo ra ngoài (Slack...), bật sau nếu cần

metrics:
  enabled: true          # để Prometheus scrape được

resources:
  requests:
    cpu: 100m
    memory: 200Mi
  limits:
    memory: 400Mi        # chặn cứng, không đặt limit cho cpu vì throttling làm Falco trễ
```

Không đặt `limits.cpu` — CPU throttling khiến Falco xử lý syscall trễ, có thể bỏ lỡ sự
kiện thay vì chỉ chạy chậm.

---

## Rule mặc định đáng chú ý

Falco đi kèm sẵn một số rule phổ biến, không cần viết thêm ngay:

| Rule | Phát hiện |
|---|---|
| Terminal shell in container | Có shell được mở bên trong container gắn với terminal thật |
| Write below etc | Tiến trình ghi vào `/etc` bên trong container |
| Read sensitive file | Đọc file dạng `/etc/shadow` và tương tự |
| Contact K8S API Server From Container | Container tự gọi thẳng vào API server, dấu hiệu bất thường |

Rà lại các rule này sau một tuần chạy để loại bớt báo động giả — chính hệ thống giám sát
(Prometheus, Grafana) có thể tự kích hoạt một vài rule mặc định vì hành vi vận hành bình
thường của chúng.

---

## Việc còn lại

- [ ] Bật audit log API server (Mục 5.4) và cấu hình plugin `k8s-audit` cho Falco đọc
      — vẫn là bước rủi ro cao, sửa `kube-apiserver.yaml`, làm cuối cùng khi cụm ổn định
- [ ] Xác nhận `falcosidekick` có cần bật để định tuyến cảnh báo ra Slack/webhook không
- [ ] Sau một tuần chạy, rà lại rule mặc định, tắt bớt rule gây báo động giả
- [ ] Bảng `audit_events` trong backend (Mục 5.5) — độc lập với Falco, làm riêng

---

## Liên quan

| Tài liệu | Nội dung |
|---|---|
| `bao-cao-giam-sat-kubernetes.docx` Mục 5.3 | Tầng hạ tầng — auditd (Falco thay thế) |
| `bao-cao-giam-sat-kubernetes.docx` Mục 5.4 | Audit log API server — vẫn cần làm riêng |
| `bao-cao-giam-sat-kubernetes.docx` Mục 9.3 | Hướng mở rộng — theo dõi hành vi lúc chạy |
| `ke-hoach-trien-khai-giam-sat-v3.docx` Mục 3 | Kế hoạch chi tiết Mục 5 |