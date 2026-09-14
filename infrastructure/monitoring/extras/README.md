# monitoring-extras

Mọi resource Kubernetes bổ sung cho hệ giám sát mà **không thuộc về chart Helm**
`kube-prometheus-stack` — ServiceMonitor cho các dịch vụ tự thêm, và PrometheusRule
cho cảnh báo hiệu năng.

Application Argo CD: `monitoring-extras` (khai báo tại `argocd/monitoring-extras.yaml`).

**Đã đổi tên thư mục** từ `servicemonitors` → `extras`, vì giờ chứa cả ServiceMonitor
lẫn PrometheusRule, không chỉ ServiceMonitor như lúc đầu.

---

## Vì sao cần thư mục này, tách khỏi Application `monitoring`

`kube-prometheus-stack` là một chart Helm lấy từ nguồn ngoài
(`prometheus-community.github.io/helm-charts`), quản lý bởi Application riêng tên
`monitoring`. Chart đó tự tạo Prometheus, Grafana, Alertmanager, và một số
ServiceMonitor/PrometheusRule mặc định cho control plane — nhưng **không biết gì về
ingress-nginx hay backend ecommerce của bạn**, vì đó là thành phần nằm ngoài chart.

`monitoring-extras` là nơi khai báo mọi thứ **riêng của cụm này**: ServiceMonitor để
Prometheus biết đường lấy metric từ ingress-nginx/backend, và luật cảnh báo hiệu năng
dành cho chính ứng dụng ecommerce.

```
Application "monitoring"         → chart Helm ngoài, hạ tầng giám sát chung
Application "monitoring-extras"  → resource riêng của cụm này (thư mục extras/)
```

---

## Cấu trúc file

```
infrastructure/monitoring/extras/
├── kustomization.yaml       danh sách file được apply
├── ingress-nginx.yaml       Service (cổng metrics) + ServiceMonitor cho ingress-nginx
├── backend.yaml             ServiceMonitor cho backend ecommerce
└── performance-rules.yaml   7 luật cảnh báo hiệu năng (Mục 6.4)
```

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ingress-nginx.yaml
  - backend.yaml
  - performance-rules.yaml
```

---

## Điều kiện tiên quyết — đã thoả từ giai đoạn 2

Mặc định Prometheus chỉ nhận ServiceMonitor/PrometheusRule mang nhãn `release` trùng
tên release Helm. Các file trong thư mục này không có nhãn đó, nên cần ba dòng trong
`infrastructure/monitoring/values.yaml`:

```yaml
prometheus:
  prometheusSpec:
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    ruleSelectorNilUsesHelmValues: false
```

Thiếu ba dòng này, ServiceMonitor/PrometheusRule bị bỏ qua **trong im lặng** — không
báo lỗi, không xuất hiện trong danh sách target/rule, chỉ đơn giản không có tác dụng.
Đã cấu hình từ giai đoạn 2, không cần làm lại.

---

## 1. ingress-nginx.yaml — Service + ServiceMonitor

### Vấn đề giải quyết

Controller phát metric ở cổng `10254`, nhưng Service gốc `ingress-nginx-controller`
chỉ mở cổng 80/443. File này tạo Service thứ hai trỏ vào cổng 10254, kèm ServiceMonitor
để Prometheus scrape.

### Hai bộ nhãn — điểm dễ nhầm nhất

| Trường | Dùng để | Giá trị |
|---|---|---|
| `metadata.labels` | ServiceMonitor tìm thấy **Service** | `component: controller-metrics` |
| `spec.selector` | Service tìm thấy **pod** | `component: controller` (sao chép từ Service gốc) |

### Điều kiện bổ sung — `--enable-metrics=true`

Manifest bare-metal chính thức của ingress-nginx đặt `--enable-metrics=false` mặc
định. Đã patch thủ công qua `kubectl patch` (không qua Ansible, vì Ansible apply
thẳng manifest từ URL, không đọc override). Cần patch lại nếu dựng lại cụm từ đầu —
xem `k8s-ansible/README.md` để biết cách tự động hoá việc này sau.

```bash
kubectl -n ingress-nginx get deploy ingress-nginx-controller \
  -o jsonpath='{.spec.template.spec.containers[0].args}' | tr ',' '\n' | tail -1
# phai la "--enable-metrics=true"
```

---

## 2. backend.yaml — ServiceMonitor cho backend ecommerce

### Vấn đề đã gặp — nhãn dùng chung giữa backend và frontend

Service `backend` và `frontend` ban đầu cùng có nhãn `app.kubernetes.io/part-of:
ecommerce` — ServiceMonitor theo nhãn đó sẽ khớp cả hai. Đã thêm nhãn riêng
`app: backend` vào `metadata.labels` của Service (không đổi `spec.selector`).

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: backend
  namespace: ecommerce
spec:
  selector:
    matchLabels:
      app: backend
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
```

### Metric thu được

`http_requests_total`, `http_request_duration_seconds_bucket`,
`http_requests_in_flight` — theo khung RED, code tự viết ở giai đoạn 4. Dùng làm
nguồn cho dashboard "Ứng dụng ecommerce" (Grafana) và 2 luật `HighErrorRate`/
`HighLatency` trong `performance-rules.yaml`.

---

## 3. performance-rules.yaml — 7 luật cảnh báo hiệu năng

Theo Mục 6.4 của tài liệu giám sát, nhãn `category: performance` để Alertmanager định
tuyến riêng khỏi cảnh báo bảo mật.

| Luật | Điều kiện | Mức |
|---|---|---|
| PodCrashLooping | Khởi động lại > 3 lần / 1 giờ | warning |
| PodPendingTooLong | Pending quá 5 phút | warning |
| DeploymentReplicasMismatch | Bản sao không sẵn sàng quá 5 phút | warning |
| NodeNotReady | Node không sẵn sàng quá 1 phút | critical |
| HighErrorRate | Tỷ lệ lỗi 5xx > 1% trong 5 phút | critical |
| HighLatency | p99 > 1 giây trong 10 phút | warning |
| TargetDown | `up == 0` quá 5 phút | critical |

Ngưỡng `HighErrorRate` (1%) và `HighLatency` (1 giây) khớp với ngưỡng đã đặt trong
dashboard Grafana "Ứng dụng ecommerce" — nhất quán giữa nơi xem và nơi cảnh báo.

**TargetDown quan trọng hơn vẻ ngoài:** nếu chính hệ giám sát chết, mọi cảnh báo khác
trở nên vô nghĩa — đây là bài học từ sự cố kube-state-metrics ở giai đoạn 2, khi metric
biến mất đúng lúc cần điều tra nhất mà biểu đồ vẫn trông "bình thường".

### Kênh nhận cảnh báo — Telegram qua Alertmanager

Alertmanager định tuyến `category: performance` và `category: security` tới cùng bot
Telegram đã dùng cho Falco (`alertmanager-telegram-token` Secret, namespace
`monitoring`, tái sử dụng token từ `falco-telegram-secret`). Cấu hình nằm trong
`infrastructure/monitoring/values.yaml`, khối `alertmanager.config` — **không** trong
thư mục `extras` này, vì đó là cấu hình của chính chart Helm `monitoring`.

Token không commit Git — xem
`infrastructure/monitoring/secret-alertmanager-telegram.example.yaml` để biết cách
tạo Secret thật trên cụm.

---

## Áp dụng

```bash
git add infrastructure/monitoring/extras argocd/monitoring-extras.yaml
git commit -m "gom servicemonitors va rules vao thu muc extras, them 7 luat canh bao hieu nang"
git push
```

Argo CD tự sync (`selfHeal: true`), hoặc:

```bash
argocd app sync monitoring-extras
```

---

## Xác minh

### ServiceMonitor

```bash
kubectl -n ecommerce get endpoints backend
kubectl -n ingress-nginx get endpoints ingress-nginx-controller-metrics
# Ca hai phai co IP pod, khong duoc <none>

curl -s http://localhost:30090/api/v1/targets | \
  jq -r '.data.activeTargets[] | select(.labels.job|test("backend|ingress")) | "\(.health) \(.labels.job)"'
```

### PrometheusRule

```bash
kubectl -n monitoring get prometheusrule performance-alerts

curl -s http://localhost:30090/api/v1/rules | \
  jq -r '.data.groups[] | select(.name=="performance") | .rules[].name'
# phai liet ke du 7 ten luat
```

### Cảnh báo tới Telegram

```bash
curl -s -X POST http://192.168.253.111:30093/api/v2/alerts -H "Content-Type: application/json" -d '[{
  "labels": {"alertname": "TestAlert", "category": "performance", "severity": "warning"},
  "annotations": {"summary": "Canh bao thu nghiem"}
}]'
```

Kiểm tra Telegram có tin nhắn mới — đã xác nhận hoạt động thật (ảnh chụp
`Ecommerce bot noti` báo `TestAlert`, `category: performance`).

---

## Truy vấn thường dùng

```promql
# Ty le loi va do tre — nguon cua HighErrorRate / HighLatency
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
histogram_quantile(0.99, sum by (le, route) (rate(http_request_duration_seconds_bucket[5m])))

# Do quet endpoint — tin hieu bao mat, Muc 7.1.1
sum by (ingress) (rate(nginx_ingress_controller_requests{status=~"401|403|404"}[5m]))

# Pod crash loop — nguon cua PodCrashLooping
increase(kube_pod_container_status_restarts_total[1h]) > 3
```

---

## Thêm ServiceMonitor hoặc PrometheusRule mới

1. Tạo file trong `infrastructure/monitoring/extras/`
2. Thêm tên file vào `resources` của `kustomization.yaml`
3. Commit, push — Argo CD tự sync

Hai điều cần nhớ:
- **Port ServiceMonitor tham chiếu theo TÊN, không theo số** — Service phải có `name`
  cho port
- **Kiểm tra Endpoint trước khi debug Prometheus** — phần lớn trường hợp "không có dữ
  liệu" là do Service không tìm thấy pod (nhãn `spec.selector` sai), không phải do
  Prometheus

---

## Liên quan

| Tài liệu | Nội dung |
|---|---|
| `bao-cao-giam-sat-kubernetes.docx` Mục 4.7 | Metric tầng mạng và Ingress |
| `bao-cao-giam-sat-kubernetes.docx` Mục 4.6.1 | Nhóm RED của ứng dụng |
| `bao-cao-giam-sat-kubernetes.docx` Mục 6.3, 6.4 | Bảng điều khiển và cảnh báo hiệu năng |
| `infrastructure/falco/README.md` | Cơ chế Telegram dùng chung, chi tiết Secret |