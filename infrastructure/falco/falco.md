# Falco

Theo dõi hành vi lúc chạy ở cả tầng máy chủ (host) và tầng container, thay cho
`auditd` trong lộ trình Mục 5 của tài liệu giám sát. Gồm phát hiện (Falco) và thông
báo tức thời qua Telegram (falcosidekick).

Application Argo CD: `falco` (khai báo tại `argocd/falco.yaml`, dùng cấu trúc
multi-source — values tách riêng tại `infrastructure/falco/values.yaml`).

**Trạng thái: hoạt động đầy đủ.** Đã xác nhận bằng thử nghiệm thật — sửa file trên máy
chủ, nhận được tin nhắn Telegram trong vài giây.

---

## Falco là gì, khác auditd ở đâu

`auditd` và audit log của API server ghi lại **lệnh gọi** — ai gọi API nào, ai sửa file
nào. Falco quan sát ở tầng thấp hơn: **mọi syscall** mà bất kỳ tiến trình nào thực hiện,
kể cả bên trong container.

| | auditd / audit log K8s | Falco |
|---|---|---|
| Nhìn vào | Log tĩnh, đọc sau | Luồng syscall, thời gian thực |
| Biết gì | Ai gọi API nào | Tiến trình nào làm gì, kể cả trên máy chủ |

**Falco không xoá bỏ việc phải bật audit log API server (Mục 5.4).** Nếu muốn Falco
giám sát cả tầng cụm K8s qua plugin `k8s-audit`, plugin đó vẫn cần đọc dữ liệu từ audit
log API server — bước rủi ro cao nhất trong Mục 5 chưa làm, để sau khi cụm ổn định lâu
dài.

---

## Kiến trúc thật đang chạy

```
Node 1 (master):     Falco pod → 10 luật host → phát hiện
Node 2 (worker-1):   Falco pod → 10 luật host → phát hiện
Node 3 (worker-2):   Falco pod → 10 luật host → phát hiện
                              ↓
                     falcosidekick (đọc Secret Telegram)
                              ↓
                     Bot Telegram → tin nhắn tức thời
```

Ba pod, một trên mỗi node, driver `modern_ebpf`. Không dùng Prometheus/Loki cho luồng
này — thông báo đi thẳng, không qua bước lưu trữ trung gian.

---

## Driver — vấn đề tương thích kernel đã gặp và đã giải quyết

### Sự cố thực tế

Hai trong ba node (`k8s-master-1`, `k8s-worker-1`) chạy kernel `7.0.0-31-generic` — một
bản kernel có số hiệu bất thường so với dải chuẩn Ubuntu 24.04. Driver `modern_ebpf`
crash liên tục trên hai node này với lỗi:

```
could not parse param 15 (flags) for event ... type 223 (clone):
expected length 4, found 494
```

### Đây là lỗi đã biết, không phải lỗi cấu hình

Xác nhận qua GitHub Issue chính thức của Falco (`falcosecurity/falco#3955`): nhiều
người dùng độc lập gặp đúng lỗi này trên kernel dòng `7.0.x`, ở cả Falco 0.43.1 và
0.44.1. Bản thân kernel này trả về cấu trúc dữ liệu syscall không đúng định dạng mà
thư viện lõi của Falco (`libs`) kỳ vọng.

### Các driver đã thử

| Driver | Kết quả |
|---|---|
| `ebpf` (cổ điển) | **Không dùng được** — đã bị chart bản 9.x xoá hoàn toàn khỏi danh sách lựa chọn hợp lệ |
| `modern_ebpf` | Crash trên kernel `7.0.0-31`, chạy ổn trên kernel `6.17.0-35` (worker-2) |
| `kmod` | Build thất bại — xung đột phiên bản glibc giữa công cụ `objtool` (build kèm header kernel) và container build của Falco |

### Giải pháp cuối cùng: giữ `modern_ebpf`, chấp nhận rủi ro đã biết

Sau khi loại trừ `kmod` (ngõ cụt do glibc) và `ebpf` cổ điển (không tồn tại ở bản chart
này), quay lại `modern_ebpf` và theo dõi thực tế: **qua nhiều giờ vận hành, không thấy
crash lặp lại** kể từ khi ổn định cấu hình cuối cùng. Có thể do khối lượng syscall trên
cụm lab thấp hơn nhiều so với môi trường được báo cáo trong issue gốc, nên ít có cơ hội
gặp đúng loại sự kiện gây lỗi.

**Rủi ro còn tồn tại, chưa biến mất hoàn toàn.** Nếu `RESTARTS` của pod Falco trên
`k8s-master-1` hoặc `k8s-worker-1` tăng bất thường, đây là nghi ngờ đầu tiên cần kiểm
tra:

```bash
kubectl -n falco get pod -o wide
kubectl -n falco logs <ten-pod> -c falco --previous --tail=30 | grep -i "could not parse"
```

Hướng khắc phục triệt để nếu lỗi tái diễn nhiều: chờ bản vá từ Falco (theo dõi issue
#3955), hoặc giới hạn Falco chỉ chạy trên node có kernel ổn định qua `nodeSelector`
(đánh đổi: mất giám sát trên node đó).

---

## Vì sao cần khai báo `mounts.volumes` / `mounts.volumeMounts`

Falco chạy **bên trong một container**, có filesystem riêng tách biệt với máy chủ thật.
Luật viết trỏ vào một đường dẫn không tự động "nhìn thấy" đường dẫn đó trên máy chủ nếu
không mount trước — luật vẫn nạp được, không báo lỗi, nhưng không bao giờ kích hoạt vì
không có gì để theo dõi.

Chart mặc định đã mount sẵn `/host/etc` (chỉ đọc), phủ được **9 trong 11** đối tượng cần
theo dõi ở Mục 5.3.5, vì chúng đều nằm dưới `/etc`. Hai đối tượng ngoài `/etc` phải tự
thêm mount:

```yaml
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

Ba mount thêm vì cụm dùng cả `root` lẫn `nguyendat` (có sudo) để đăng nhập, cả hai
`authorized_keys` đều có khoá thật.

---

## Bài học quan trọng nhất: đường dẫn trong ĐIỀU KIỆN luật KHÔNG dùng tiền tố `/host`

Đây là nhầm lẫn tốn nhiều thời gian debug nhất trong toàn bộ quá trình, cần ghi nhớ rõ
để không lặp lại.

### Hai loại đường dẫn, dễ nhầm vì cùng trỏ một file vật lý

| Ngữ cảnh | Có tiền tố `/host` không | Ví dụ |
|---|---|---|
| **Tự đọc file từ trong container Falco** (`kubectl exec ... ls /host/...`) | **Có** | `/host/etc/kubernetes/manifests` |
| **Điều kiện luật** (`fd.name` trong `condition`/`output`) | **Không** | `/etc/kubernetes/manifests` |

### Vì sao khác nhau

`fd.name` mà Falco ghi nhận trong sự kiện syscall là đường dẫn **theo góc nhìn của
kernel** — kernel không biết gì về việc container Falco tự mount `/host/etc`, nó chỉ
thấy đường dẫn thật duy nhất tồn tại trên toàn hệ thống: `/etc/kubernetes/manifests`,
không có tiền tố nào cả.

Xác nhận bằng luật debug tạm thời (bắt mọi `open_write`, in ra `fd.name` thô):

```
file=/dev/null process=flanneld
file=/proc/self/loginuid process=cron
```

Không dòng nào có tiền tố `/host`, dù sự kiện tới từ cả tiến trình host lẫn container.

### Hệ quả

10 luật ban đầu viết `condition: open_write and fd.name startswith /host/etc/...` —
**không bao giờ khớp**, dù mount đúng, dù cú pháp đúng, dù chạy không báo lỗi. Sau khi
sửa bỏ tiền tố `/host` khỏi mọi điều kiện, cả 10 luật hoạt động ngay.

---

## Danh sách 10 luật đang chạy

Ứng với 11 đối tượng ở Mục 5.3.5 (2 đối tượng `sudoers`/`sudoers.d` gộp một luật).

| Luật | Đường dẫn theo dõi | Hành động | Mức |
|---|---|---|---|
| Sửa cấu hình control plane trên máy chủ | `/etc/kubernetes/manifests` | ghi | CRITICAL |
| Đụng chứng chỉ cụm trên máy chủ | `/etc/kubernetes/pki` | ghi | CRITICAL |
| Đọc kubeconfig quản trị trên máy chủ | `/etc/kubernetes/admin.conf` | đọc | CRITICAL |
| Đụng dữ liệu etcd trên máy chủ | `/var/lib/etcd` | ghi | CRITICAL |
| Sửa quyền sudo trên máy chủ | `/etc/sudoers`, `/etc/sudoers.d` | ghi | CRITICAL |
| Tạo hoặc xoá tài khoản trên máy chủ | `/etc/passwd` | ghi | CRITICAL |
| Đổi mật khẩu trên máy chủ | `/etc/shadow` | ghi | CRITICAL |
| Đổi nhóm quyền trên máy chủ | `/etc/group` | ghi | WARNING |
| Cài cắm khoá SSH trên máy chủ | `/root/.ssh`, `/home/nguyendat/.ssh` | ghi | CRITICAL |
| Nới lỏng chính sách SSH trên máy chủ | `/etc/ssh/sshd_config` | ghi | WARNING |

Luật đọc `admin.conf` có loại trừ `kubelet`, `kube-apiserver`, `etcd` để tránh báo động
giả — các tiến trình hệ thống đọc file này liên tục khi vận hành bình thường.

### Đã thử nghiệm thật, xác nhận bằng log

```bash
sudo sh -c 'echo test >> /etc/kubernetes/manifests/.test-falco'
sudo cat /etc/kubernetes/admin.conf > /dev/null
sudo sh -c 'echo "" >> /etc/passwd'
sudo sh -c 'echo test >> /root/.ssh/.test-falco'
```

Cả bốn đều sinh đúng sự kiện, đúng luật, đúng nội dung `output`.

---

## Thông báo tức thời qua Telegram (falcosidekick)

### Kiến trúc

`falcosidekick` là một container phụ trong cùng chart Falco, đọc log của Falco và đẩy
đi các kênh thông báo khác nhau. Không cần Prometheus, không cần lưu trữ trung gian —
Critical xảy ra là tin nhắn tới ngay.

### Bảo mật: token không nằm trong Git

Token bot Telegram là thông tin nhạy cảm. Repo này công khai trên GitHub, nên **không**
đặt token trực tiếp trong `values.yaml`. Dùng Kubernetes Secret, tạo thủ công trên cụm,
không commit.

**File mẫu commit được** — `infrastructure/falco/secret-telegram.example.yaml`, không
chứa giá trị thật:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: falco-telegram-secret
  namespace: falco
type: Opaque
stringData:
  TELEGRAM_TOKEN: "DIEN_BOT_TOKEN_THAT_VAO_DAY"
  TELEGRAM_CHATID: "DIEN_CHAT_ID_THAT_VAO_DAY"
```

**Tên key bắt buộc viết đúng** `TELEGRAM_TOKEN`, `TELEGRAM_CHATID` — Secret được nạp
thẳng vào biến môi trường của container `falcosidekick` qua `envFrom`, tên key trong
Secret chính là tên biến môi trường mà mã nguồn đọc. Xác nhận từ mã nguồn chart
(`templates/deployment.yaml`, khối `envFrom.secretRef`) và từ báo cáo người dùng thật
trên GitHub Issue #1283 của `falcosidekick`.

### Quy trình tạo Secret thật

```bash
# Chan file that khoi Git
echo "infrastructure/falco/secret-telegram.yaml" >> .gitignore
git add .gitignore && git commit -m "chan file secret telegram that" && git push

# Tao ban that tu file mau
cp infrastructure/falco/secret-telegram.example.yaml \
   infrastructure/falco/secret-telegram.yaml
vi infrastructure/falco/secret-telegram.yaml   # dien token va chatid that

# Ap dung truc tiep, KHONG qua Argo CD
kubectl apply -f infrastructure/falco/secret-telegram.yaml
```

### Lấy token và chat ID

1. Telegram, tìm **@BotFather**, gửi `/newbot`, làm theo hướng dẫn → nhận `TELEGRAM_TOKEN`
2. Gửi tin bất kỳ cho bot vừa tạo (bấm Start trước)
3. Mở trình duyệt: `https://api.telegram.org/bot<TOKEN>/getUpdates`
4. Tìm `"chat":{"id": ...}` trong kết quả → đó là `TELEGRAM_CHATID`

### Cấu hình trong `values.yaml`

```yaml
falcosidekick:
  enabled: true
  config:
    existingSecret: "falco-telegram-secret"
    telegram:
      minimumpriority: "critical"
```

`token`/`chatid` **không** khai báo trực tiếp trong values — chúng tới từ Secret.
`minimumpriority` không nhạy cảm nên để thẳng trong values, tách khỏi Secret.

### Xác nhận

```bash
kubectl -n falco get pod | grep sidekick
POD_SIDEKICK=$(kubectl -n falco get pod -o name | grep sidekick)
kubectl -n falco exec -it $POD_SIDEKICK -- printenv | grep TELEGRAM
```

Phải thấy `TELEGRAM_TOKEN=...` và `TELEGRAM_CHATID=...` có giá trị, không rỗng.

### Đã xác nhận hoạt động thật

Test bằng cách sửa `/etc/kubernetes/manifests/.test-alert` trên `k8s-master-1`, nhận
được tin nhắn Telegram trong vài giây, đầy đủ thông tin: thời gian, host, tên luật, file
bị đụng, tiến trình, người dùng.

---

## Chi phí tài nguyên

| Hạng mục | Mức |
|---|---|
| RAM mỗi pod Falco | 300–768 MB (giới hạn cứng, `limits.memory: 768Mi`) |
| RAM falcosidekick | Nhỏ, không giới hạn riêng — theo dõi thêm nếu cần |
| CPU | Liên tục, tăng khi container hoạt động nhiều — không đặt `limits.cpu`, throttling làm Falco xử lý trễ |
| Yêu cầu kernel | `modern_ebpf` cần kernel ≥ 5.8, nhưng có kernel cụ thể không tương thích dù thoả điều kiện version (xem phần Driver ở trên) |

### Sự cố OOMKilled đã gặp và đã sửa

Cấu hình ban đầu `limits.memory: 400Mi` khiến 2/3 pod bị `OOMKilled` (mã thoát 137) ngay
lúc khởi tạo vùng đệm cho `modern_ebpf`. Đã nâng lên `768Mi`, ổn định từ đó.

Kiểm tra RAM khả dụng trên cả ba node trước khi sync:

```bash
for h in 192.168.253.111 192.168.253.112 192.168.253.113; do
  echo "=== $h ==="
  ssh nguyendat@$h 'free -h'
done
```

---

## Cấu trúc file trong repo

```
argocd/falco.yaml                          Application, multi-source
infrastructure/falco/values.yaml            Toàn bộ cấu hình Helm, bao gồm 10 luật
infrastructure/falco/secret-telegram.example.yaml   Mẫu Secret, an toàn commit
infrastructure/falco/secret-telegram.yaml   Secret thật, BỊ CHẶN bởi .gitignore
```

Multi-source dùng vì `repoURL` của chart (`falcosecurity.github.io/charts`) nằm ngoài
repo Git của dự án — Argo CD cần khai báo hai nguồn tách biệt (chart + values) và nối
bằng `$values`/`ref`.

---

## Cài đặt từ đầu (nếu dựng lại cụm)

### 1. Xác nhận phiên bản chart và schema thật — không đoán

Chart Falco đã lên version 9.x, khác hẳn 3.x/4.x cũ — nhiều tên trường đã đổi
(`extraVolumes` → `mounts.volumes`, driver `ebpf` cổ điển bị xoá hoàn toàn).

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts 2>/dev/null
helm repo update
helm search repo falcosecurity/falco --versions | head -5
```

### 2. Kiểm tra kernel trước khi chọn driver

```bash
uname -r    # chay tren ca 3 node
```

`< 5.8` → không dùng `modern_ebpf` được, cân nhắc `kmod` (cần header kernel khớp chính
xác, kiểm tra `/usr/src/` trước).

### 3. Apply

```bash
kubectl apply -f argocd/falco.yaml
```

### 4. Tạo Secret Telegram (xem mục ở trên)

### 5. Xác minh toàn diện

```bash
kubectl -n falco get pod -o wide
# Ca 4 pod (3 falco + 1 sidekick, hoac gop chung tuy cau hinh) phai Running on định

kubectl -n falco get daemonset falco -o jsonpath='{.spec.template.spec.containers[0].volumeMounts}' | jq
# Phai co du 3 mount moi: var-lib-etcd, root-ssh, user-ssh

sudo sh -c 'echo test >> /etc/kubernetes/manifests/.verify'
sleep 3
kubectl -n falco logs $(kubectl -n falco get pod -o wide | grep k8s-master-1 | awk '{print $1}') --tail=20 | grep -i CRITICAL
sudo rm -f /etc/kubernetes/manifests/.verify
# Phai thay dung rule kich hoat, va tin nhan Telegram toi trong vai giay
```

---

## Việc còn lại

- [ ] Test 6 luật còn lại chưa xác nhận trực tiếp (pki, etcd, sudoers, shadow, group,
      sshd_config) — cùng cấu trúc với 4 luật đã test, khả năng cao đều hoạt động đúng
- [ ] Theo dõi `RESTARTS` của Falco trên master/worker-1 qua nhiều ngày — xác nhận lỗi
      kernel #3955 không tái diễn dưới tải thực tế cao hơn
- [ ] Bật audit log API server (Mục 5.4) — rủi ro cao, sửa `kube-apiserver.yaml`, làm
      khi cụm ổn định lâu dài, không vội
- [ ] Bảng `audit_events` trong backend (Mục 5.5) — độc lập với Falco, việc riêng
- [ ] Cân nhắc `minimumpriority: "warning"` nếu muốn nhận cả 2 luật mức WARNING
- [ ] Lưu trữ log lâu dài (Loki) — đã thử, hoãn lại vì chart Loki bản mới (7.x/app 3.x)
      bắt buộc object storage (S3/MinIO), không phù hợp tài nguyên lab hiện tại (đĩa
      5.1GB trống, RAM 1GB free). Cân nhắc lại khi có thêm tài nguyên hoặc tìm được
      chart version cũ hơn còn hỗ trợ `filesystem` thuần.

---

## Liên quan

| Tài liệu | Nội dung |
|---|---|
| `bao-cao-giam-sat-kubernetes.docx` Mục 5.3 | Tầng hạ tầng — nguyên lý audit, 11 đối tượng, nguồn tham khảo |
| `bao-cao-giam-sat-kubernetes.docx` Mục 5.4 | Audit log API server — chưa làm |
| `bao-cao-giam-sat-kubernetes.docx` Mục 9.3 | Hướng mở rộng — theo dõi hành vi lúc chạy |
| Falco Issue #3955 | Lỗi tương thích kernel 7.0.x với driver modern_ebpf |
| falcosidekick Issue #1283 | Xác nhận tên biến môi trường Telegram qua envFrom |# Falco

Theo dõi hành vi lúc chạy ở cả tầng máy chủ (host) và tầng container, thay cho
`auditd` trong lộ trình Mục 5 của tài liệu giám sát. Gồm phát hiện (Falco) và thông
báo tức thời qua Telegram (falcosidekick).

Application Argo CD: `falco` (khai báo tại `argocd/falco.yaml`, dùng cấu trúc
multi-source — values tách riêng tại `infrastructure/falco/values.yaml`).

**Trạng thái: hoạt động đầy đủ.** Đã xác nhận bằng thử nghiệm thật — sửa file trên máy
chủ, nhận được tin nhắn Telegram trong vài giây.

---

## Falco là gì, khác auditd ở đâu

`auditd` và audit log của API server ghi lại **lệnh gọi** — ai gọi API nào, ai sửa file
nào. Falco quan sát ở tầng thấp hơn: **mọi syscall** mà bất kỳ tiến trình nào thực hiện,
kể cả bên trong container.

| | auditd / audit log K8s | Falco |
|---|---|---|
| Nhìn vào | Log tĩnh, đọc sau | Luồng syscall, thời gian thực |
| Biết gì | Ai gọi API nào | Tiến trình nào làm gì, kể cả trên máy chủ |

**Falco không xoá bỏ việc phải bật audit log API server (Mục 5.4).** Nếu muốn Falco
giám sát cả tầng cụm K8s qua plugin `k8s-audit`, plugin đó vẫn cần đọc dữ liệu từ audit
log API server — bước rủi ro cao nhất trong Mục 5 chưa làm, để sau khi cụm ổn định lâu
dài.

---

## Kiến trúc thật đang chạy

```
Node 1 (master):     Falco pod → 10 luật host → phát hiện
Node 2 (worker-1):   Falco pod → 10 luật host → phát hiện
Node 3 (worker-2):   Falco pod → 10 luật host → phát hiện
                              ↓
                     falcosidekick (đọc Secret Telegram)
                              ↓
                     Bot Telegram → tin nhắn tức thời
```

Ba pod, một trên mỗi node, driver `modern_ebpf`. Không dùng Prometheus/Loki cho luồng
này — thông báo đi thẳng, không qua bước lưu trữ trung gian.

---

## Driver — vấn đề tương thích kernel đã gặp và đã giải quyết

### Sự cố thực tế

Hai trong ba node (`k8s-master-1`, `k8s-worker-1`) chạy kernel `7.0.0-31-generic` — một
bản kernel có số hiệu bất thường so với dải chuẩn Ubuntu 24.04. Driver `modern_ebpf`
crash liên tục trên hai node này với lỗi:

```
could not parse param 15 (flags) for event ... type 223 (clone):
expected length 4, found 494
```

### Đây là lỗi đã biết, không phải lỗi cấu hình

Xác nhận qua GitHub Issue chính thức của Falco (`falcosecurity/falco#3955`): nhiều
người dùng độc lập gặp đúng lỗi này trên kernel dòng `7.0.x`, ở cả Falco 0.43.1 và
0.44.1. Bản thân kernel này trả về cấu trúc dữ liệu syscall không đúng định dạng mà
thư viện lõi của Falco (`libs`) kỳ vọng.

### Các driver đã thử

| Driver | Kết quả |
|---|---|
| `ebpf` (cổ điển) | **Không dùng được** — đã bị chart bản 9.x xoá hoàn toàn khỏi danh sách lựa chọn hợp lệ |
| `modern_ebpf` | Crash trên kernel `7.0.0-31`, chạy ổn trên kernel `6.17.0-35` (worker-2) |
| `kmod` | Build thất bại — xung đột phiên bản glibc giữa công cụ `objtool` (build kèm header kernel) và container build của Falco |

### Giải pháp cuối cùng: giữ `modern_ebpf`, chấp nhận rủi ro đã biết

Sau khi loại trừ `kmod` (ngõ cụt do glibc) và `ebpf` cổ điển (không tồn tại ở bản chart
này), quay lại `modern_ebpf` và theo dõi thực tế: **qua nhiều giờ vận hành, không thấy
crash lặp lại** kể từ khi ổn định cấu hình cuối cùng. Có thể do khối lượng syscall trên
cụm lab thấp hơn nhiều so với môi trường được báo cáo trong issue gốc, nên ít có cơ hội
gặp đúng loại sự kiện gây lỗi.

**Rủi ro còn tồn tại, chưa biến mất hoàn toàn.** Nếu `RESTARTS` của pod Falco trên
`k8s-master-1` hoặc `k8s-worker-1` tăng bất thường, đây là nghi ngờ đầu tiên cần kiểm
tra:

```bash
kubectl -n falco get pod -o wide
kubectl -n falco logs <ten-pod> -c falco --previous --tail=30 | grep -i "could not parse"
```

Hướng khắc phục triệt để nếu lỗi tái diễn nhiều: chờ bản vá từ Falco (theo dõi issue
#3955), hoặc giới hạn Falco chỉ chạy trên node có kernel ổn định qua `nodeSelector`
(đánh đổi: mất giám sát trên node đó).

---

## Vì sao cần khai báo `mounts.volumes` / `mounts.volumeMounts`

Falco chạy **bên trong một container**, có filesystem riêng tách biệt với máy chủ thật.
Luật viết trỏ vào một đường dẫn không tự động "nhìn thấy" đường dẫn đó trên máy chủ nếu
không mount trước — luật vẫn nạp được, không báo lỗi, nhưng không bao giờ kích hoạt vì
không có gì để theo dõi.

Chart mặc định đã mount sẵn `/host/etc` (chỉ đọc), phủ được **9 trong 11** đối tượng cần
theo dõi ở Mục 5.3.5, vì chúng đều nằm dưới `/etc`. Hai đối tượng ngoài `/etc` phải tự
thêm mount:

```yaml
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

Ba mount thêm vì cụm dùng cả `root` lẫn `nguyendat` (có sudo) để đăng nhập, cả hai
`authorized_keys` đều có khoá thật.

---

## Bài học quan trọng nhất: đường dẫn trong ĐIỀU KIỆN luật KHÔNG dùng tiền tố `/host`

Đây là nhầm lẫn tốn nhiều thời gian debug nhất trong toàn bộ quá trình, cần ghi nhớ rõ
để không lặp lại.

### Hai loại đường dẫn, dễ nhầm vì cùng trỏ một file vật lý

| Ngữ cảnh | Có tiền tố `/host` không | Ví dụ |
|---|---|---|
| **Tự đọc file từ trong container Falco** (`kubectl exec ... ls /host/...`) | **Có** | `/host/etc/kubernetes/manifests` |
| **Điều kiện luật** (`fd.name` trong `condition`/`output`) | **Không** | `/etc/kubernetes/manifests` |

### Vì sao khác nhau

`fd.name` mà Falco ghi nhận trong sự kiện syscall là đường dẫn **theo góc nhìn của
kernel** — kernel không biết gì về việc container Falco tự mount `/host/etc`, nó chỉ
thấy đường dẫn thật duy nhất tồn tại trên toàn hệ thống: `/etc/kubernetes/manifests`,
không có tiền tố nào cả.

Xác nhận bằng luật debug tạm thời (bắt mọi `open_write`, in ra `fd.name` thô):

```
file=/dev/null process=flanneld
file=/proc/self/loginuid process=cron
```

Không dòng nào có tiền tố `/host`, dù sự kiện tới từ cả tiến trình host lẫn container.

### Hệ quả

10 luật ban đầu viết `condition: open_write and fd.name startswith /host/etc/...` —
**không bao giờ khớp**, dù mount đúng, dù cú pháp đúng, dù chạy không báo lỗi. Sau khi
sửa bỏ tiền tố `/host` khỏi mọi điều kiện, cả 10 luật hoạt động ngay.

---

## Danh sách 10 luật đang chạy

Ứng với 11 đối tượng ở Mục 5.3.5 (2 đối tượng `sudoers`/`sudoers.d` gộp một luật).

| Luật | Đường dẫn theo dõi | Hành động | Mức |
|---|---|---|---|
| Sửa cấu hình control plane trên máy chủ | `/etc/kubernetes/manifests` | ghi | CRITICAL |
| Đụng chứng chỉ cụm trên máy chủ | `/etc/kubernetes/pki` | ghi | CRITICAL |
| Đọc kubeconfig quản trị trên máy chủ | `/etc/kubernetes/admin.conf` | đọc | CRITICAL |
| Đụng dữ liệu etcd trên máy chủ | `/var/lib/etcd` | ghi | CRITICAL |
| Sửa quyền sudo trên máy chủ | `/etc/sudoers`, `/etc/sudoers.d` | ghi | CRITICAL |
| Tạo hoặc xoá tài khoản trên máy chủ | `/etc/passwd` | ghi | CRITICAL |
| Đổi mật khẩu trên máy chủ | `/etc/shadow` | ghi | CRITICAL |
| Đổi nhóm quyền trên máy chủ | `/etc/group` | ghi | WARNING |
| Cài cắm khoá SSH trên máy chủ | `/root/.ssh`, `/home/nguyendat/.ssh` | ghi | CRITICAL |
| Nới lỏng chính sách SSH trên máy chủ | `/etc/ssh/sshd_config` | ghi | WARNING |

Luật đọc `admin.conf` có loại trừ `kubelet`, `kube-apiserver`, `etcd` để tránh báo động
giả — các tiến trình hệ thống đọc file này liên tục khi vận hành bình thường.

### Đã thử nghiệm thật, xác nhận bằng log

```bash
sudo sh -c 'echo test >> /etc/kubernetes/manifests/.test-falco'
sudo cat /etc/kubernetes/admin.conf > /dev/null
sudo sh -c 'echo "" >> /etc/passwd'
sudo sh -c 'echo test >> /root/.ssh/.test-falco'
```

Cả bốn đều sinh đúng sự kiện, đúng luật, đúng nội dung `output`.

---

## Thông báo tức thời qua Telegram (falcosidekick)

### Kiến trúc

`falcosidekick` là một container phụ trong cùng chart Falco, đọc log của Falco và đẩy
đi các kênh thông báo khác nhau. Không cần Prometheus, không cần lưu trữ trung gian —
Critical xảy ra là tin nhắn tới ngay.

### Bảo mật: token không nằm trong Git

Token bot Telegram là thông tin nhạy cảm. Repo này công khai trên GitHub, nên **không**
đặt token trực tiếp trong `values.yaml`. Dùng Kubernetes Secret, tạo thủ công trên cụm,
không commit.

**File mẫu commit được** — `infrastructure/falco/secret-telegram.example.yaml`, không
chứa giá trị thật:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: falco-telegram-secret
  namespace: falco
type: Opaque
stringData:
  TELEGRAM_TOKEN: "DIEN_BOT_TOKEN_THAT_VAO_DAY"
  TELEGRAM_CHATID: "DIEN_CHAT_ID_THAT_VAO_DAY"
```

**Tên key bắt buộc viết đúng** `TELEGRAM_TOKEN`, `TELEGRAM_CHATID` — Secret được nạp
thẳng vào biến môi trường của container `falcosidekick` qua `envFrom`, tên key trong
Secret chính là tên biến môi trường mà mã nguồn đọc. Xác nhận từ mã nguồn chart
(`templates/deployment.yaml`, khối `envFrom.secretRef`) và từ báo cáo người dùng thật
trên GitHub Issue #1283 của `falcosidekick`.

### Quy trình tạo Secret thật

```bash
# Chan file that khoi Git
echo "infrastructure/falco/secret-telegram.yaml" >> .gitignore
git add .gitignore && git commit -m "chan file secret telegram that" && git push

# Tao ban that tu file mau
cp infrastructure/falco/secret-telegram.example.yaml \
   infrastructure/falco/secret-telegram.yaml
vi infrastructure/falco/secret-telegram.yaml   # dien token va chatid that

# Ap dung truc tiep, KHONG qua Argo CD
kubectl apply -f infrastructure/falco/secret-telegram.yaml
```

### Lấy token và chat ID

1. Telegram, tìm **@BotFather**, gửi `/newbot`, làm theo hướng dẫn → nhận `TELEGRAM_TOKEN`
2. Gửi tin bất kỳ cho bot vừa tạo (bấm Start trước)
3. Mở trình duyệt: `https://api.telegram.org/bot<TOKEN>/getUpdates`
4. Tìm `"chat":{"id": ...}` trong kết quả → đó là `TELEGRAM_CHATID`

### Cấu hình trong `values.yaml`

```yaml
falcosidekick:
  enabled: true
  config:
    existingSecret: "falco-telegram-secret"
    telegram:
      minimumpriority: "critical"
```

`token`/`chatid` **không** khai báo trực tiếp trong values — chúng tới từ Secret.
`minimumpriority` không nhạy cảm nên để thẳng trong values, tách khỏi Secret.

### Xác nhận

```bash
kubectl -n falco get pod | grep sidekick
POD_SIDEKICK=$(kubectl -n falco get pod -o name | grep sidekick)
kubectl -n falco exec -it $POD_SIDEKICK -- printenv | grep TELEGRAM
```

Phải thấy `TELEGRAM_TOKEN=...` và `TELEGRAM_CHATID=...` có giá trị, không rỗng.

### Đã xác nhận hoạt động thật

Test bằng cách sửa `/etc/kubernetes/manifests/.test-alert` trên `k8s-master-1`, nhận
được tin nhắn Telegram trong vài giây, đầy đủ thông tin: thời gian, host, tên luật, file
bị đụng, tiến trình, người dùng.

---

## Chi phí tài nguyên

| Hạng mục | Mức |
|---|---|
| RAM mỗi pod Falco | 300–768 MB (giới hạn cứng, `limits.memory: 768Mi`) |
| RAM falcosidekick | Nhỏ, không giới hạn riêng — theo dõi thêm nếu cần |
| CPU | Liên tục, tăng khi container hoạt động nhiều — không đặt `limits.cpu`, throttling làm Falco xử lý trễ |
| Yêu cầu kernel | `modern_ebpf` cần kernel ≥ 5.8, nhưng có kernel cụ thể không tương thích dù thoả điều kiện version (xem phần Driver ở trên) |

### Sự cố OOMKilled đã gặp và đã sửa

Cấu hình ban đầu `limits.memory: 400Mi` khiến 2/3 pod bị `OOMKilled` (mã thoát 137) ngay
lúc khởi tạo vùng đệm cho `modern_ebpf`. Đã nâng lên `768Mi`, ổn định từ đó.

Kiểm tra RAM khả dụng trên cả ba node trước khi sync:

```bash
for h in 192.168.253.111 192.168.253.112 192.168.253.113; do
  echo "=== $h ==="
  ssh nguyendat@$h 'free -h'
done
```

---

## Cấu trúc file trong repo

```
argocd/falco.yaml                          Application, multi-source
infrastructure/falco/values.yaml            Toàn bộ cấu hình Helm, bao gồm 10 luật
infrastructure/falco/secret-telegram.example.yaml   Mẫu Secret, an toàn commit
infrastructure/falco/secret-telegram.yaml   Secret thật, BỊ CHẶN bởi .gitignore
```

Multi-source dùng vì `repoURL` của chart (`falcosecurity.github.io/charts`) nằm ngoài
repo Git của dự án — Argo CD cần khai báo hai nguồn tách biệt (chart + values) và nối
bằng `$values`/`ref`.

---

## Cài đặt từ đầu (nếu dựng lại cụm)

### 1. Xác nhận phiên bản chart và schema thật — không đoán

Chart Falco đã lên version 9.x, khác hẳn 3.x/4.x cũ — nhiều tên trường đã đổi
(`extraVolumes` → `mounts.volumes`, driver `ebpf` cổ điển bị xoá hoàn toàn).

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts 2>/dev/null
helm repo update
helm search repo falcosecurity/falco --versions | head -5
```

### 2. Kiểm tra kernel trước khi chọn driver

```bash
uname -r    # chay tren ca 3 node
```

`< 5.8` → không dùng `modern_ebpf` được, cân nhắc `kmod` (cần header kernel khớp chính
xác, kiểm tra `/usr/src/` trước).

### 3. Apply

```bash
kubectl apply -f argocd/falco.yaml
```

### 4. Tạo Secret Telegram (xem mục ở trên)

### 5. Xác minh toàn diện

```bash
kubectl -n falco get pod -o wide
# Ca 4 pod (3 falco + 1 sidekick, hoac gop chung tuy cau hinh) phai Running on định

kubectl -n falco get daemonset falco -o jsonpath='{.spec.template.spec.containers[0].volumeMounts}' | jq
# Phai co du 3 mount moi: var-lib-etcd, root-ssh, user-ssh

sudo sh -c 'echo test >> /etc/kubernetes/manifests/.verify'
sleep 3
kubectl -n falco logs $(kubectl -n falco get pod -o wide | grep k8s-master-1 | awk '{print $1}') --tail=20 | grep -i CRITICAL
sudo rm -f /etc/kubernetes/manifests/.verify
# Phai thay dung rule kich hoat, va tin nhan Telegram toi trong vai giay
```

---

## Việc còn lại

- [ ] Test 6 luật còn lại chưa xác nhận trực tiếp (pki, etcd, sudoers, shadow, group,
      sshd_config) — cùng cấu trúc với 4 luật đã test, khả năng cao đều hoạt động đúng
- [ ] Theo dõi `RESTARTS` của Falco trên master/worker-1 qua nhiều ngày — xác nhận lỗi
      kernel #3955 không tái diễn dưới tải thực tế cao hơn
- [ ] Bật audit log API server (Mục 5.4) — rủi ro cao, sửa `kube-apiserver.yaml`, làm
      khi cụm ổn định lâu dài, không vội
- [ ] Bảng `audit_events` trong backend (Mục 5.5) — độc lập với Falco, việc riêng
- [ ] Cân nhắc `minimumpriority: "warning"` nếu muốn nhận cả 2 luật mức WARNING
- [ ] Lưu trữ log lâu dài (Loki) — đã thử, hoãn lại vì chart Loki bản mới (7.x/app 3.x)
      bắt buộc object storage (S3/MinIO), không phù hợp tài nguyên lab hiện tại (đĩa
      5.1GB trống, RAM 1GB free). Cân nhắc lại khi có thêm tài nguyên hoặc tìm được
      chart version cũ hơn còn hỗ trợ `filesystem` thuần.

---

## Liên quan

| Tài liệu | Nội dung |
|---|---|
| `bao-cao-giam-sat-kubernetes.docx` Mục 5.3 | Tầng hạ tầng — nguyên lý audit, 11 đối tượng, nguồn tham khảo |
| `bao-cao-giam-sat-kubernetes.docx` Mục 5.4 | Audit log API server — chưa làm |
| `bao-cao-giam-sat-kubernetes.docx` Mục 9.3 | Hướng mở rộng — theo dõi hành vi lúc chạy |
| Falco Issue #3955 | Lỗi tương thích kernel 7.0.x với driver modern_ebpf |
| falcosidekick Issue #1283 | Xác nhận tên biến môi trường Telegram qua envFrom |