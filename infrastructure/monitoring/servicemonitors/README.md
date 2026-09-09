# monitoring-extras

ServiceMonitor cho các thành phần không thuộc chart `kube-prometheus-stack`.

Chart cài sẵn ServiceMonitor cho control plane, node-exporter, kube-state-metrics và
CoreDNS. Những thành phần còn lại — ingress-nginx, ứng dụng ecommerce — cần khai báo
riêng, và thư mục này là nơi chứa chúng.

Application Argo CD: `monitoring-extras` (khai báo tại `argocd/monitoring-extras.yaml`).

---

## Cấu trúc

```
infrastructure/monitoring/servicemonitors/
├── kustomization.yaml     danh sách file được apply
├── ingress-nginx.yaml     Service + ServiceMonitor cho ingress-nginx
└── README.md              file này
```

---

## Luồng hoạt động

```
Git repo
  └── servicemonitors/
        ↑
   Argo CD Application "monitoring-extras" đọc và apply
        ↓
   Cụm: Service mới xuất hiện
        ↓
   Prometheus Operator phát hiện ServiceMonitor → sinh cấu hình scrape
        ↓
   Prometheus scrape endpoint mỗi 30 giây
```

Prometheus của kube-prometheus-stack không đọc file cấu hình scrape tĩnh. Nó tìm các
object `ServiceMonitor` và `PodMonitor` trong cụm rồi tự sinh cấu hình. Vì vậy thêm một
scrape target nghĩa là tạo một object, không phải sửa file cấu hình.

---

## Điều kiện tiên quyết

Mặc định Prometheus chỉ nhận ServiceMonitor mang nhãn `release` trùng tên release Helm.
ServiceMonitor trong thư mục này không có nhãn đó, nên cần nới selector trong
`infrastructure/monitoring/values.yaml`:

```yaml
prometheus:
  prometheusSpec:
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    ruleSelectorNilUsesHelmValues: false
```

Thiếu ba dòng này thì ServiceMonitor bị bỏ qua **trong im lặng** — không báo lỗi, không
xuất hiện trong danh sách target, chỉ đơn giản là không có dữ liệu. Đây là nguyên nhân
gây mất thời gian nhiều nhất khi tự viết ServiceMonitor.

Đã áp dụng ở giai đoạn 2 của kế hoạch triển khai.

---

## ingress-nginx

### Vấn đề cần giải quyết

Controller phát ra metric ở cổng `10254`, nhưng Service `ingress-nginx-controller` chỉ
mở cổng 80 và 443. Prometheus chạy ở pod khác nên không có đường vào.

Cổng 10254 không được khai báo trong `containerPort` của Deployment. Điều đó không ảnh
hưởng: `containerPort` chỉ mang tính tài liệu, không mở hay đóng cổng nào. Tiến trình vẫn
lắng nghe trên mọi interface — đã xác nhận bằng cách gọi trực tiếp vào IP pod từ node
khác.

### Cách giải quyết

Tạo một Service ClusterIP thứ hai trỏ vào cổng 10254 của cùng pod, kèm ServiceMonitor
tương ứng. Service phục vụ traffic (80/443) giữ nguyên, không đụng tới.

### Hai bộ nhãn — điểm dễ nhầm nhất

Service có hai bộ nhãn với vai trò ngược nhau:

| Trường | Dùng để | Giá trị |
|---|---|---|
| `metadata.labels` | ServiceMonitor tìm thấy **Service** | `component: controller-metrics` |
| `spec.selector` | Service tìm thấy **pod** | `component: controller` |

`spec.selector` phải sao chép nguyên từ Service đang chạy — nó đã hoạt động nên chắc
chắn khớp:

```bash
kubectl -n ingress-nginx get svc ingress-nginx-controller \
  -o jsonpath='{.spec.selector}'; echo
```

`metadata.labels` cố ý dùng `controller-metrics` thay vì `controller`, để ServiceMonitor
chọn đúng Service mới chứ không vớ phải Service 80/443.

Sai `spec.selector` thì Service vẫn được tạo bình thường nhưng không có Endpoint nào, và
target sẽ hỏng theo cách khác hẳn lỗi kết nối thông thường — dễ chẩn đoán nhầm.

### Metric thu được

| Metric | Khung | Dùng ở đâu |
|---|---|---|
| `nginx_ingress_controller_requests` | RED — Requests, Errors | Mục 4.7, 6.2, 7.1.1 |
| `nginx_ingress_controller_request_duration_seconds` | RED — Duration | Mục 4.7, 6.2 |

Đây là RED tại điểm vào, có được mà **không cần sửa mã nguồn ứng dụng**. Metric ở tầng
ứng dụng (`http_requests_total`) chính xác hơn nhưng phải chờ giai đoạn 4.

---

## Triển khai

### Kiểm tra cú pháp trước

```bash
python3 -c "import yaml; list(yaml.safe_load_all(open('ingress-nginx.yaml')))" \
  && echo "YAML hop le"
```

### Apply tay để thử

Tách lỗi YAML khỏi lỗi GitOps:

```bash
kubectl apply -f ingress-nginx.yaml
```

### Đưa vào Git

```bash
git add infrastructure/monitoring/servicemonitors argocd/monitoring-extras.yaml
git commit -m "monitoring: ServiceMonitor cho ingress-nginx"
git push
kubectl apply -f argocd/monitoring-extras.yaml
```

Bước này cần thiết kể cả khi đã apply tay. Object nằm ngoài Git là object nằm ngoài tầm
kiểm soát của Argo CD — chính là hiện tượng trôi cấu hình.

---

## Xác minh, theo thứ tự

Ba bước, mỗi bước loại bớt một nhóm nguyên nhân.

### 1. Service có tìm thấy pod không

```bash
kubectl -n ingress-nginx get endpoints ingress-nginx-controller-metrics
```

Cột `ENDPOINTS` phải có IP pod dạng `10.244.x.x:10254`.
Nếu là `<none>` → `spec.selector` sai.

### 2. ServiceMonitor có được Prometheus nhận không

```bash
kubectl -n ingress-nginx get servicemonitor
```

Rồi mở `http://<node-ip>:30090/targets`, tìm job `ingress-nginx`.

- Không thấy xuất hiện gì → selector của Prometheus chưa nới, xem phần điều kiện tiên quyết
- Thấy nhưng DOWN → xem `lastError` để biết lý do

### 3. Metric có dữ liệu không

```promql
nginx_ingress_controller_requests
```

Rỗng có thể chỉ vì chưa có traffic. Tạo ít traffic rồi thử lại:

```bash
for i in $(seq 1 20); do curl -s -o /dev/null http://<node-ip>:31156/; done
```

---

## Truy vấn thường dùng

```promql
# Ty le loi 5xx toan he thong
sum(rate(nginx_ingress_controller_requests{status=~"5.."}[5m]))
  / sum(rate(nginx_ingress_controller_requests[5m]))

# Do tre p99 theo tung ingress
histogram_quantile(0.99, sum by (le, ingress) (
  rate(nginx_ingress_controller_request_duration_seconds_bucket[5m])))

# Tan suat request
sum by (ingress) (rate(nginx_ingress_controller_requests[5m]))

# Tin hieu bao mat: do quet endpoint
sum by (ingress) (
  rate(nginx_ingress_controller_requests{status=~"401|403|404"}[5m]))
```

Truy vấn cuối phục vụ Mục 7.1.1 của tài liệu đặc tả — phát hiện dò quét từ bên ngoài mà
không cần thu thập access log.

---

## Cấu hình liên quan trong ConfigMap của controller

Nằm ngoài thư mục này vì ingress-nginx được cài qua Ansible, không qua Argo CD. Sửa
trực tiếp:

```bash
kubectl -n ingress-nginx edit configmap ingress-nginx-controller
```

```yaml
data:
  generate-request-id: "true"
  log-format-escape-json: "true"
  log-format-upstream: '{"time":"$time_iso8601","request_id":"$req_id","remote_addr":"$remote_addr","method":"$request_method","uri":"$uri","status":$status,"duration":$request_time,"upstream_time":"$upstream_response_time","namespace":"$namespace","ingress":"$ingress_name","service":"$service_name","user_agent":"$http_user_agent"}'
```

`generate-request-id` yêu cầu controller sinh một mã duy nhất cho mỗi request và gắn vào
tiêu đề `X-Request-ID` khi chuyển xuống backend. Đây là mắt xích đầu tiên của chuỗi truy
vết ở Mục 8 — backend đọc mã này và ghi vào bảng `audit_events`.

Bỏ qua bước này thì mọi bản ghi audit về sau đều không nối được với lưu lượng vào, và
không sửa ngược lại được cho dữ liệu đã tích luỹ.

`log-format-upstream` phải nằm trên **một dòng duy nhất**. Xuống dòng giữa chừng làm
nginx không nạp được cấu hình — controller sẽ không reload, nhưng cụm vẫn chạy bình
thường.

Kiểm tra:

```bash
# Ma dinh danh da duoc sinh
curl -s -D- -o /dev/null http://<node-ip>:31156/ | grep -i request-id

# Access log da ra JSON
kubectl -n ingress-nginx logs deploy/ingress-nginx-controller --tail=3
```

---

## Thêm ServiceMonitor mới

Ví dụ cho backend ở giai đoạn 4:

1. Tạo file `backend.yaml` trong thư mục này
2. Thêm `- backend.yaml` vào `resources` của `kustomization.yaml`
3. Commit và push — Argo CD tự sync

Hai điều cần nhớ khi viết:

- **Port tham chiếu theo TÊN, không theo số.** Service phải có `name` cho port thì
  ServiceMonitor mới trỏ tới được.
- **Kiểm tra Endpoint trước khi debug Prometheus.** Phần lớn trường hợp "không có dữ
  liệu" là do Service không tìm thấy pod, không phải do Prometheus.

---

## Liên quan

| Tài liệu | Nội dung |
|---|---|
| `bao-cao-giam-sat-kubernetes.docx` Mục 4.7 | Metric tầng mạng và Ingress |
| `bao-cao-giam-sat-kubernetes.docx` Mục 5.6 | Access log tại Ingress |
| `bao-cao-giam-sat-kubernetes.docx` Mục 7.1.1 | Tín hiệu bảo mật từ xác thực và phân quyền |
| `ke-hoach-trien-khai-giam-sat.docx` Giai đoạn 3 | Kế hoạch và cách xác minh |