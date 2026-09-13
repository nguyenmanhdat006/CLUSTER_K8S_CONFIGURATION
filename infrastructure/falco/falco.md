# Falco

Theo dõi hành vi lúc chạy ở cả tầng máy chủ (host) và tầng container, thay cho
`auditd` trong lộ trình Mục 5 của tài liệu giám sát.

Application Argo CD: `falco` (khai báo tại `argocd/falco.yaml`, dùng cấu trúc
multi-source — xem Mục "Cấu trúc file" bên dưới).

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
                              metrics service → Prometheus
```

---

## Vì sao cần khai báo `mounts.volumes` / `mounts.volumeMounts`

Đây là phần hay bị hiểu nhầm: tưởng chỉ cần viết luật trỏ vào một đường dẫn là Falco tự
giám sát được đường dẫn đó trên máy chủ. Thực tế cần một bước trước đó.

### Container có filesystem riêng, tách biệt với máy chủ

Falco chạy **bên trong một container**. Theo đúng bản chất của container, nó có hệ thống
tệp riêng, hoàn toàn tách biệt với máy chủ thật đang chạy nó:

```
Máy chủ thật                    Container Falco
/root/.ssh/authorized_keys      /root/.ssh/   ← thư mục CỦA RIÊNG container,
/var/lib/etcd/                                   trống, không liên quan máy chủ
```

Nếu viết luật trỏ vào `/root/.ssh/authorized_keys` mà không mount gì thêm, Falco đang
nhìn vào thư mục `/root/.ssh` **của chính nó**, không phải của máy chủ. Không ai ghi gì
vào đó cả — luật sẽ không bao giờ kích hoạt, và không có gì báo lỗi để nhận ra vấn đề.
Đây là loại lỗi âm thầm giống hệt trường hợp `--enable-metrics=false` từng gặp ở
ingress-nginx: cấu hình trông đúng, chạy không báo lỗi, nhưng dữ liệu cần thì không bao
giờ xuất hiện.

### `volumes` / `volumeMounts` là "khoan lỗ xuyên tường container"

Khai báo này nói với Kubernetes: lấy một thư mục thật trên máy chủ (`hostPath`), nối nó
vào một đường dẫn bên trong container Falco. Quy ước đặt tiền tố `/host` cho đường dẫn
đích, để không lẫn với thư mục gốc cùng tên của chính container:

```yaml
mounts:
  volumes:
    - name: var-lib-etcd
      hostPath:
        path: /var/lib/etcd        # thư mục THẬT trên máy chủ
  volumeMounts:
    - name: var-lib-etcd
      mountPath: /host/var/lib/etcd  # đường dẫn BÊN TRONG container Falco
      readOnly: true
```

Sau khi mount, luật viết trong Falco phải trỏ vào `/host/var/lib/etcd`, không phải
`/var/lib/etcd` — vì đó mới là nơi Falco thật sự nhìn thấy dữ liệu của máy chủ.

### Hai bước, thứ tự bắt buộc

```
1. Mount đường dẫn (mounts.volumes / mounts.volumeMounts)
     → Falco NHÌN THẤY được thư mục của máy chủ
2. Viết luật (falco.rules)
     → Falco BIẾT khi nào cần báo động cho đường dẫn đó
```

Làm bước 2 mà bỏ qua bước 1 thì luật không có gì để bắt — đúng cú pháp cũng vô dụng.

### Thư mục `/etc` đã có sẵn, phần còn lại phải tự thêm

Chart Falco mặc định đã mount sẵn `/host/etc` (chỉ đọc), phủ được **9 trong 11** đối
tượng cần theo dõi ở Mục 5.3.5 của tài liệu — vì chúng đều nằm dưới `/etc`. Hai đối
tượng còn lại nằm ngoài `/etc`, phải tự khai báo thêm mount riêng:

| Đường dẫn | Đã có mount sẵn? |
|---|---|
| Mọi thứ dưới `/etc/...` (9 đối tượng) | Có, qua `/host/etc` |
| `/var/lib/etcd` | Không — đã bổ sung |
| `/root/.ssh/authorized_keys` | Không — đã bổ sung |
| `/home/nguyendat/.ssh/authorized_keys` | Không — đã bổ sung |

Ba mục cuối được thêm vì cụm dùng cả tài khoản `root` lẫn `nguyendat` (có sudo) để đăng
nhập, cả hai `authorized_keys` đều có khoá thật đang hoạt động — cần theo dõi cả hai,
không chỉ một.

---

## Chi phí tài nguyên — đọc trước khi bật thêm rule

| Hạng mục | Mức |
|---|---|
| RAM mỗi node | 300–768 MB (đã nâng sau sự cố OOMKilled, xem bên dưới) |
| CPU | Liên tục, tăng khi container hoạt động nhiều |
| Yêu cầu kernel | eBPF hiện đại cần kernel ≥ 5.8 |

Cụm này từng gặp sự cố nghiêm trọng vì thiếu giới hạn bộ nhớ cho Prometheus (xem nhật ký
giai đoạn 2 và 3). Vì vậy `resources.limits.memory` trong cấu hình Falco là bắt buộc,
không phải tuỳ chọn.

**Sự cố thực tế đã gặp:** cấu hình ban đầu đặt `limits.memory: 400Mi`, hai trong ba pod
bị `OOMKilled` (mã thoát 137) ngay khi khởi tạo vùng đệm cho driver `modern_ebpf` — quá
trình này cần nhiều bộ nhớ hơn mức chạy ổn định sau đó. Đã nâng lên `768Mi` và bổ sung
`hostPID: true` (bắt buộc để driver eBPF bám đúng không gian định danh tiến trình gốc
của máy chủ). Sau khi sửa, cả ba pod chạy ổn định `2/2 Running`.

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
| ≥ 5.8 | `modern_ebpf` (đang dùng) |
| < 5.8 | `ebpf` — sửa `driver.kind` trong `infrastructure/falco/values.yaml` |

---

## Cấu trúc file

Dùng Argo CD multi-source: một nguồn là chart Helm từ kho công khai, một nguồn là file
values riêng lấy từ chính Git repo — vì `repoURL` của chart nằm ngoài repo hạ tầng, Argo
CD không tự ghép được file values cục bộ với chart nguồn ngoài nếu không khai báo theo
cách này.

```
argocd/falco.yaml                    Application, cấu trúc "sources" (nhiều nguồn)
infrastructure/falco/values.yaml     Toàn bộ giá trị cấu hình Helm
```

`argocd/falco.yaml` trỏ tới chart Falco và tham chiếu `values.yaml` qua `$values/...`,
`infrastructure/falco/values.yaml` chứa nội dung cấu hình thật — driver, tài nguyên, và
phần `mounts` đã giải thích ở trên.

---

## Cài đặt

### 1. Xác nhận phiên bản chart thật trước khi apply

Chart Falco đã lên tới nhánh version 9.x, khác hẳn các bản 3.x/4.x cũ hơn — cấu trúc
values đã đổi giữa các major version. Ví dụ thực tế đã gặp: tên trường mount không phải
`extraVolumes`/`extraVolumeMounts` như các bản cũ, mà là `mounts.volumes` /
`mounts.volumeMounts` — chỉ phát hiện được bằng cách tra trực tiếp, không đoán từ trí
nhớ hay tài liệu cũ.

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts 2>/dev/null
helm repo update
helm search repo falcosecurity/falco --versions | head -5
```

Đối chiếu số trong `targetRevision` với kết quả lệnh trên trước khi apply.

### 2. Kiểm tra values khớp đúng schema chart thật

```bash
helm show values falcosecurity/falco --version <version-vua-tim> > /tmp/falco-values-goc.yaml
grep -n -i "volume" /tmp/falco-values-goc.yaml
grep -n -B2 -A15 "^driver:" /tmp/falco-values-goc.yaml
```

So với nội dung trong `infrastructure/falco/values.yaml`. Nếu tên trường khác, sửa lại
theo đúng schema thật thay vì giữ nguyên tên đoán trước.

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

Phải thấy đúng 3 pod, mỗi node một cái, trạng thái `2/2 Running`, cột `RESTARTS` không
tăng thêm sau vài phút quan sát.

```bash
kubectl -n falco logs -l app.kubernetes.io/name=falco --tail=30
```

Tìm dòng xác nhận driver nạp thành công. Nếu pod crash ngay sau dòng log về kích thước
vùng đệm syscall, gần như chắc chắn là `OOMKilled` — kiểm tra bằng lệnh dưới.

```bash
kubectl -n falco describe pod <ten-pod> | grep -A10 "Last State"
```

### Xác nhận mount đã vào đúng chỗ

```bash
kubectl -n falco get daemonset falco -o jsonpath='{.spec.template.spec.containers[0].volumeMounts}' | jq
```

Phải thấy đủ ba mount mới: `var-lib-etcd`, `root-ssh`, `user-ssh`, cạnh mount `/host/etc`
đã có sẵn từ chart.

```bash
kubectl -n falco exec -it $(kubectl -n falco get pod -o name | head -1) -- ls /host/var/lib/etcd
kubectl -n falco exec -it $(kubectl -n falco get pod -o name | head -1) -- ls -la /host/root/.ssh
```

Cả hai lệnh phải liệt kê được nội dung thật của máy chủ. Nếu trống hoặc báo lỗi, mount
chưa đúng — quay lại kiểm tra `hostPath` trong `values.yaml`.

### RAM sau khi cài

```bash
kubectl -n falco top pod
free -h    # chạy trên cả ba node
```

Đối chiếu với số đo trước khi cài.

---

## Nội dung values.yaml đang dùng

```yaml
driver:
  kind: modern_ebpf   # đổi thành "ebpf" nếu kernel < 5.8

hostPID: true          # bắt buộc cho driver eBPF, thiếu sẽ gây lỗi khi mở syscall probe
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
    memory: 300Mi
  limits:
    memory: 768Mi        # 400Mi từng gây OOMKilled ngay lúc khởi tạo vùng đệm eBPF

mounts:
  volumes:
    - name: var-lib-etcd
      hostPath:
        path: /var/lib/etcd
    - name: root-ssh
      hostPath:
        path: /root/.ssh
    - name: user-ssh
      hostPath:
        path: /home/nguyendat/.ssh
  volumeMounts:
    - name: var-lib-etcd
      mountPath: /host/var/lib/etcd
      readOnly: true
    - name: root-ssh
      mountPath: /host/root/.ssh
      readOnly: true
    - name: user-ssh
      mountPath: /host/home/nguyendat/.ssh
      readOnly: true
```

Không đặt `limits.cpu` — CPU throttling khiến Falco xử lý syscall trễ, có thể bỏ lỡ sự
kiện thay vì chỉ chạy chậm.

---

## Rule mặc định đáng chú ý

Falco đi kèm sẵn một số rule phổ biến, nhắm vào hành vi **bên trong container**:

| Rule | Phát hiện |
|---|---|
| Terminal shell in container | Có shell được mở bên trong container gắn với terminal thật |
| Write below etc | Tiến trình ghi vào `/etc` bên trong container |
| Read sensitive file | Đọc file dạng `/etc/shadow` và tương tự, bên trong container |
| Contact K8S API Server From Container | Container tự gọi thẳng vào API server |

**Chưa có rule nào nhắm vào máy chủ** — đó là lý do cần viết luật riêng cho 11 đối tượng
ở Mục 5.3.5 của tài liệu, việc tiếp theo sau khi mount đã xác nhận hoạt động.

Rà lại rule mặc định sau một tuần chạy để loại bớt báo động giả — chính hệ thống giám
sát (Prometheus, Grafana) có thể tự kích hoạt một vài rule mặc định vì hành vi vận hành
bình thường của chúng.

---

## Việc còn lại

- [ ] Viết luật riêng cho 11 đối tượng tầng máy chủ (Mục 5.3.5), dùng đường dẫn có tiền
      tố `/host` đã mount
- [ ] Thử nghiệm thật: chạm vào một file trong danh sách, xác nhận Falco sinh sự kiện
- [ ] Xác nhận sự kiện tới Prometheus qua `falco_events_total`
- [ ] Bật audit log API server (Mục 5.4) và cấu hình plugin `k8s-audit` cho Falco đọc
      — vẫn là bước rủi ro cao, sửa `kube-apiserver.yaml`, làm cuối cùng khi cụm ổn định
- [ ] Xác nhận `falcosidekick` có cần bật để định tuyến cảnh báo ra Slack/webhook không
- [ ] Bảng `audit_events` trong backend (Mục 5.5) — độc lập với Falco, làm riêng

---

## Liên quan

| Tài liệu | Nội dung |
|---|---|
| `bao-cao-giam-sat-kubernetes.docx` Mục 5.3 | Tầng hạ tầng — nguyên lý audit, 11 đối tượng, nguồn tham khảo |
| `bao-cao-giam-sat-kubernetes.docx` Mục 5.4 | Audit log API server — vẫn cần làm riêng |
| `bao-cao-giam-sat-kubernetes.docx` Mục 9.3 | Hướng mở rộng — theo dõi hành vi lúc chạy |
| `ke-hoach-trien-khai-giam-sat-v5.docx` Mục 3, giai đoạn 5a-ii | Bốn việc chuẩn bị trước khi viết luật |