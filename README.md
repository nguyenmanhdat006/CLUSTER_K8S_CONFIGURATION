# GitOps — Ứng dụng Ecommerce (3 môi trường)

Triển khai ứng dụng ecommerce lên Kubernetes theo mô hình **GitOps** với Argo CD, quản lý
ba môi trường **dev / staging / prod** bằng kỹ thuật **Kustomize base/overlays**.

Nguyên tắc cốt lõi: Git là nguồn sự thật duy nhất. Mọi thay đổi được thực hiện qua commit;
Argo CD tự động đồng bộ trạng thái cluster cho khớp với Git.

---

## Kiến trúc thư mục

```
gitops-demo/
├── argocd/                      # Các Argo CD Application (apply thủ công 1 lần)
│   ├── dev.yaml                 #   → theo dõi app/overlays/dev
│   ├── staging.yaml             #   → theo dõi app/overlays/staging
│   ├── prod.yaml                #   → theo dõi app/overlays/prod
│   └── monitoring.yaml          #   → cài kube-prometheus-stack (xem phần "Giám sát")
│
├── infrastructure/              # Công cụ vận hành cluster (không phải app người dùng)
│   └── monitoring/
│       └── values.yaml          #   Values cho chart kube-prometheus-stack
│
├── app/
│   ├── base/                    # Manifest DÙNG CHUNG, trung lập môi trường
│   │   ├── backend.yaml         #   Deployment + Service backend
│   │   ├── frontend.yaml        #   Deployment + Service frontend
│   │   ├── ingress.yaml         #   Định tuyến (host để placeholder)
│   │   └── kustomization.yaml
│   │
│   └── overlays/                # KHÁC BIỆT theo từng môi trường
│       ├── dev/
│       │   ├── kustomization.yaml
│       │   ├── namespace.yaml
│       │   ├── hpa.yaml
│       │   └── backend-sealed-secret.example.yaml
│       ├── staging/
│       │   └── ... (tương tự)
│       └── prod/
│           ├── kustomization.yaml
│           ├── namespace.yaml
│           ├── hpa.yaml
│           ├── resources-patch.yaml
│           └── backend-sealed-secret.yaml     # secret thật (đã seal)
│
└── k8s-ansible/                 # Dựng cụm Kubernetes từ đầu bằng Ansible (tuỳ chọn)
    ├── site.yml                 #   Playbook chính: chuẩn bị node → init master → join worker → add-on
    ├── inventory.ini            #   Danh sách 3 máy (1 master + 2 worker) + user SSH
    ├── group_vars/all.yml       #   Phiên bản K8s, dải IP pod, URL add-on
    └── roles/                   #   common / master / worker / addons
```

## Nguyên lý base/overlays

`base/` chứa những gì **không đổi** giữa các môi trường. Mỗi `overlays/<env>/` chỉ khai
báo **phần khác biệt** của môi trường đó, rồi Kustomize ghép lại:

```
        base/                overlays/<env>/            Kết quả
   ┌──────────────┐        ┌──────────────┐        ┌──────────────┐
   │ backend       │        │ namespace     │        │ manifest      │
   │ frontend      │  +     │ số replica    │   =    │ hoàn chỉnh    │
   │ ingress       │        │ host          │        │ cho môi       │
   │ (không ns,    │        │ tài nguyên    │        │ trường đó     │
   │  host tạm)    │        │ secret        │        │               │
   └──────────────┘        └──────────────┘        └──────────────┘
```

## Khác biệt giữa ba môi trường

| Thuộc tính        | dev                     | staging                     | prod                    |
|-------------------|-------------------------|-----------------------------|-------------------------|
| Namespace         | `ecommerce-dev`         | `ecommerce-staging`         | `ecommerce`             |
| HPA backend       | min 1 / max 3, CPU 80%  | min 2 / max 5, CPU 75%      | min 3 / max 10, CPU 70% |
| HPA frontend      | min 1 / max 2, CPU 80%  | min 2 / max 4, CPU 75%      | min 3 / max 8, CPU 70%  |
| Host              | `dev.k8s.nguyendat.tech`| `staging.k8s.nguyendat.tech`| `k8s.nguyendat.tech`    |
| Tài nguyên backend| mặc định (base)         | mặc định (base)             | cấp cao hơn             |
| Sealed Secret     | cần seal cho namespace  | cần seal cho namespace      | có sẵn (seal cho `ecommerce`) |

Số replica **không** còn khai báo tĩnh trong Git — mỗi Deployment được
`HorizontalPodAutoscaler` (HPA) quản lý dựa trên % CPU sử dụng (`requests.cpu` làm mốc).
Argo CD được cấu hình `ignoreDifferences` trên `/spec/replicas` để không giằng co với HPA
(xem phần "Trải nghiệm vòng lặp GitOps" bên dưới).

Quy ước namespace: môi trường prod dùng namespace gốc `ecommerce`; các môi trường phi-prod
dùng hậu tố `-<env>`. Đây là một quy ước phổ biến giúp phân tách rõ ràng.

---

## Chưa có cluster? Dựng bằng Ansible

Thư mục [`k8s-ansible/`](k8s-ansible/) chứa playbook Ansible dựng **từ đầu** một cụm
Kubernetes 3 node (1 master + 2 worker) trên VMware, kèm sẵn mọi thứ repo GitOps này cần:
Argo CD, metrics-server, ingress-nginx, sealed-secrets. Nếu đã có cluster sẵn đáp ứng phần
"Yêu cầu hệ thống" bên dưới thì bỏ qua bước này.

Tóm tắt cách chạy (chi tiết xem [`k8s-ansible/README.md`](k8s-ansible/README.md)):

```bash
cd k8s-ansible

# 1. Sửa inventory.ini: user SSH + đường dẫn private key (IP 3 máy đã điền sẵn)
# 2. Kiểm tra Ansible vào được cả 3 máy
ansible all -m ping

# 3. Chạy triển khai (mất khoảng 10–20 phút)
ansible-playbook site.yml -K
```

Kết thúc, Ansible in ra **mật khẩu admin Argo CD** — dùng nó để đăng nhập, sau đó quay lại
các bước bên dưới để apply Argo CD Application của repo này. Lưu ý: cluster mới sinh khóa
sealed-secrets mới, nên `backend-sealed-secret.yaml` hiện có trong `app/overlays/` **phải
được seal lại** cho cluster mới trước khi dùng (xem "Lưu ý bảo mật" cuối file).

---

## Yêu cầu hệ thống

- Cluster Kubernetes đã cài **Argo CD**.
- Đã cài **sealed-secrets controller** (để giải mã `backend-secret`).
- **ingress-nginx** đang chạy (để định tuyến qua host; không bắt buộc để quan sát vòng lặp).

## Hướng dẫn triển khai

### Bước 1 — Cấu hình repo
Các file Application trong `argocd/` (`dev.yaml`, `staging.yaml`, `prod.yaml`, `monitoring.yaml`)
đã trỏ sẵn `repoURL` về `https://github.com/nguyenmanhdat006/CLUSTER_K8S_CONFIGURATION`. Nếu
fork sang repo khác, cập nhật lại URL này trong cả bốn file trước khi apply.

### Bước 2 — Đẩy lên Git
Argo CD đọc cấu hình từ Git, không đọc từ máy cục bộ. Commit và push toàn bộ trước.

### Bước 3 — Triển khai môi trường prod (chạy được ngay)
Môi trường prod dùng sealed secret có sẵn (đã seal cho namespace `ecommerce`):

```bash
kubectl apply -f argocd/prod.yaml
```

Kiểm tra:
```bash
kubectl get application ecommerce-prod -n argocd     # trạng thái Synced / Healthy
kubectl get pods -n ecommerce                        # 3 backend + 3 frontend
kubectl get secret backend-secret -n ecommerce       # controller đã giải mã
```

### Bước 4 — Chuẩn bị secret cho dev / staging
Mỗi môi trường cần secret riêng, seal lại cho đúng namespace (SealedSecret bị ràng buộc
với namespace). Tạo cho `ecommerce-dev`:

```bash
kubectl create secret generic backend-secret \
  --from-literal=DATABASE_URL='<gia-tri-cua-ban>' \
  --from-literal=FRONTEND_ORIGIN='<gia-tri-cua-ban>' \
  --from-literal=PORT='8080' \
  -n ecommerce-dev --dry-run=client -o yaml \
| kubeseal --format yaml \
    --controller-name sealed-secrets \
    --controller-namespace <NAMESPACE_CONTROLLER> \
> app/overlays/dev/backend-sealed-secret.yaml
```

Sau đó bỏ comment dòng `backend-sealed-secret.yaml` trong
`app/overlays/dev/kustomization.yaml`, commit, push. Làm tương tự cho staging.

### Bước 5 — Triển khai dev / staging
```bash
kubectl apply -f argocd/dev.yaml
kubectl apply -f argocd/staging.yaml
```

Hoặc triển khai cả ba môi trường cùng lúc:
```bash
kubectl apply -f argocd/
```

---

## Trải nghiệm vòng lặp GitOps

Đây là điểm cốt lõi cần nắm. Thử thay đổi cấu hình môi trường dev:

1. Mở `app/overlays/dev/hpa.yaml`, đổi `maxReplicas: 3` thành `maxReplicas: 5` cho `backend`.
2. Commit và push:
   ```bash
   git add app/overlays/dev/hpa.yaml
   git commit -m "raise dev backend max replicas to 5"
   git push
   ```
3. Không chạy lệnh `kubectl apply`. Chỉ quan sát:
   ```bash
   kubectl get hpa -n ecommerce-dev -w
   ```

Argo CD phát hiện thay đổi trong Git và tự động cập nhật cluster. Quan trọng: thay đổi chỉ
tác động môi trường dev, prod và staging không bị ảnh hưởng — đó là giá trị của việc tách
overlays.

**Vì sao Argo CD không giằng co với HPA:** HPA liên tục ghi `spec.replicas` của Deployment
theo tải CPU thực tế, còn Git không khai báo giá trị này (`replicas` đã bị bỏ khỏi
`app/base/`). Mỗi `argocd/<env>.yaml` có thêm `ignoreDifferences` trên `/spec/replicas`, nên
Argo CD bỏ qua khác biệt ở field đó thay vì tự phục hồi (self-heal) về Git — nếu không có
khối này, Argo sẽ liên tục kéo replicas về giá trị cũ và triệt tiêu tác dụng của HPA.

Kiểm chứng HPA đang hoạt động (cần **metrics-server** trong cluster):
```bash
kubectl top pods -n ecommerce-dev                # xác nhận metrics-server có dữ liệu
kubectl get hpa -n ecommerce-dev -w               # theo dõi HPA scale theo %CPU
```

---

## Vận hành thường ngày

**Thay đổi cấu hình một môi trường:** sửa file trong `overlays/<env>/`, commit, push.

**Thay đổi áp dụng cho mọi môi trường:** sửa file trong `base/`, commit, push. Cả ba môi
trường cùng nhận thay đổi.

**Thêm môi trường mới (ví dụ uat):** sao chép một overlay, sửa namespace / host / replicas,
tạo `argocd/uat.yaml`, apply.

## Giám sát (monitoring)

Prometheus + Grafana + Alertmanager, cài qua Argo CD bằng Helm chart
`kube-prometheus-stack` (multi-source: chart từ internet + `values.yaml` từ repo Git này).
Đây là công cụ vận hành cluster, tách biệt khỏi `app/` — nằm ở `infrastructure/monitoring/`.

### Cài đặt

1. `argocd/monitoring.yaml` đã ghim sẵn `targetRevision: 90.0.0` (bản mới nhất của chart tại
   thời điểm viết). Muốn nâng cấp về sau:
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
   helm search repo prometheus-community/kube-prometheus-stack | head
   ```
   rồi sửa lại số version trong `argocd/monitoring.yaml`.

2. `repoURL` trong `argocd/monitoring.yaml` đã trỏ sẵn về repo này (xem "Bước 1" ở trên).

3. Đổi `adminPassword` trong `infrastructure/monitoring/values.yaml` trước khi push.

4. Commit + push, rồi apply Application (tách biệt với `argocd/<env>.yaml`, không nằm
   trong vòng lặp GitOps tự động của app):
   ```bash
   kubectl apply -f argocd/monitoring.yaml
   ```

### Truy cập (NodePort)

- Grafana:      `http://<node-ip>:30300` (admin / mật khẩu bạn đặt)
- Prometheus:   `http://<node-ip>:30090`
- Alertmanager: `http://<node-ip>:30093`

### Kiểm tra

```bash
kubectl get pods -n monitoring
kubectl get pods -n monitoring -o wide     # xem pod nằm ở node nào
```

## Vai trò từng thành phần

| Thành phần | Vai trò |
|------------|---------|
| `argocd/<env>.yaml` | Argo CD Application: theo dõi một overlay, đồng bộ vào một namespace |
| `argocd/monitoring.yaml` | Argo CD Application: cài `kube-prometheus-stack` từ `infrastructure/monitoring/values.yaml` |
| `app/base/` | Manifest gốc, dùng chung cho mọi môi trường |
| `app/overlays/<env>/kustomization.yaml` | Ghép base + khai báo khác biệt của môi trường |
| `app/overlays/<env>/hpa.yaml` | HorizontalPodAutoscaler cho backend + frontend, ngưỡng theo môi trường |
| `app/overlays/prod/resources-patch.yaml` | Patch tài nguyên (requests/limits) riêng cho prod |
| `app/overlays/<env>/namespace.yaml` | Namespace riêng của môi trường |
| `app/overlays/<env>/backend-sealed-secret.yaml` | Secret đã seal cho namespace tương ứng |
| `infrastructure/monitoring/values.yaml` | Values cho chart `kube-prometheus-stack` (Grafana/Prometheus/Alertmanager) |

## Dọn dẹp

```bash
kubectl delete -f argocd/        # xóa TẤT CẢ Application, kể cả monitoring; prune dọn theo tài nguyên đã tạo
```

Chỉ muốn gỡ riêng monitoring: `kubectl delete -f argocd/monitoring.yaml`.

## Lưu ý bảo mật

Không commit secret dạng plaintext hay private key. Chỉ commit `SealedSecret` (đã mã hóa).
Mỗi SealedSecret gắn với một namespace cụ thể; đổi namespace phải seal lại.
