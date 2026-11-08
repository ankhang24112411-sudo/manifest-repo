# Kubernetes manifests — shoe-store

Cấu trúc Kustomize: `base/` chứa tài nguyên chung, `overlays/<env>/` gắn namespace + Ingress riêng.

```
k8s/
├── bootstrap/namespaces.yaml   # staging, production, argocd
├── base/                       # configmap, api, web, postgresql
├── overlays/staging/           # namespace staging, host staging.shoe.test
├── overlays/production/        # namespace production, host shoe.test
└── argocd/                     # Application trỏ vào từng overlay
```

## 1. Build & push image

CI của repo app tự build và push 2 image lên GHCR khi push vào `main` (`.github/workflows/ci.yml`):

- `ghcr.io/buinam2k4/teamwork_2:latest` — API (`Dockerfile`)
- `ghcr.io/buinam2k4/teamwork_2-web:latest` — Web tĩnh (`Dockerfile.web`)

Build tay nếu cần:

```bash
echo "$GHCR_TOKEN" | docker login ghcr.io -u buinam2k4 --password-stdin
docker build -t ghcr.io/buinam2k4/teamwork_2:latest .
docker build -f Dockerfile.web -t ghcr.io/buinam2k4/teamwork_2-web:latest .
docker push ghcr.io/buinam2k4/teamwork_2:latest
docker push ghcr.io/buinam2k4/teamwork_2-web:latest
```

Package GHCR mặc định private. Chọn 1 trong 2:
- để package **Public** (Settings → Packages), hoặc
- tạo pull secret ở mỗi namespace rồi gắn `imagePullSecrets` vào Pod:

```bash
kubectl -n staging create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io --docker-username=buinam2k4 --docker-password="$GHCR_TOKEN"
```

Web là nginx phục vụ tĩnh `public/`; điều hướng phía client dùng hash (`#/...`) nên không cần fallback SPA.

## 2. Tạo Secret (không commit plaintext)

`db-secret` phải tồn tại ở cả 2 namespace:

```bash
for ns in staging production; do
  kubectl -n $ns create secret generic db-secret \
    --from-literal=POSTGRES_PASSWORD='<pass-db>' \
    --from-literal=DATABASE_URL='postgres://shoe:<pass-db>@postgres:5432/shoe' \
    --from-literal=SESSION_SECRET="$(openssl rand -hex 32)" \
    --from-literal=ADMIN_PASSWORD='<pass-admin>'
done
```

`SESSION_SECRET` là bắt buộc khi `NODE_ENV=production` (app sẽ từ chối khởi động nếu thiếu).
Có thể thay bằng SealedSecret/External Secret nếu repo công khai.

## 3. Bootstrap namespace + ArgoCD

```bash
kubectl apply -f k8s/bootstrap/namespaces.yaml
kubectl apply -f k8s/argocd/project.yaml
kubectl apply -f k8s/argocd/application-staging.yaml
kubectl apply -f k8s/argocd/application-production.yaml
```

Hoặc chạy tay không qua ArgoCD:

```bash
kubectl apply -k k8s/overlays/staging
kubectl apply -k k8s/overlays/production
```

## 4. Ingress Controller & publish

- Cluster dùng **Traefik** (mặc định của k3s). Ingress khai báo `ingressClassName: traefik`.
- Traefik chạy dạng Service `LoadBalancer` (k3s có ServiceLB) → truy cập qua IP node, hoặc thêm `hosts`:
  - `staging.shoe.test` và `shoe.test` → IP node.
- Kiểm tra: `kubectl -n staging get ingress`, `curl -H 'Host: staging.shoe.test' http://<node-ip>/api/products`.

## 5. Tài nguyên & Pending

- `api`: requests 100m/128Mi, limits 500m/256Mi. `web`: 50m/64Mi → 200m/128Mi. `postgres`: 100m/256Mi → 1CPU/512Mi.
- Pod Pending thường do thiếu CPU/RAM trên node: `kubectl -n staging describe pod <pod>` xem `Insufficient cpu/memory`, rồi giảm requests hoặc thêm node.
- Pod Pending do PVC: `kubectl -n staging get pvc` phải `Bound`; nếu `Pending` kiểm tra StorageClass `local-path` tồn tại (`kubectl get storageclass`).
- `api` giữ 1 replica vì session lưu in-memory (MemoryStore). Tăng replica cần thêm session store dùng chung (Redis/DB), nếu không người dùng sẽ bị đăng xuất ngẫu nhiên. `web` chạy 2 replica.

## 6. Phân quyền tối thiểu

- Không commit kubeconfig vào repo. ArgoCD dùng ServiceAccount trong cluster, không dùng kubeconfig admin.
- Repo Git nên cấp chỉ quyền đọc (deploy key read-only / token read-only).
- `AppProject` trong `k8s/argocd/project.yaml` giới hạn mỗi Application chỉ được đọc đúng repo trên và chỉ ghi vào đúng namespace của môi trường (`staging` hoặc `production`), không dùng `project: default`.

## 7. Kiểm chứng

```bash
kubectl -n production get pods,svc,ingress
kubectl -n production get pvc                          # Bound
kubectl -n production rollout restart sts postgres
kubectl -n production get pod postgres-0               # Ready lại, dữ liệu còn
kubectl -n production rollout restart deploy api
```
