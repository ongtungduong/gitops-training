# gitops-training

Lab GitOps trên ArgoCD: **App-of-Apps + ApplicationSet**, multi-source, **1 base Helm chart dùng chung**.
Một repo; mỗi project là 1 branch (branch-based), mỗi env là 1 directory (directory-based).

## Cấu trúc

```
main (branch hạ tầng)
├── root-application.yaml      # App-of-Apps gốc — apply tay 1 lần
├── bootstrap/
│   ├── appprojects/           # AppProject: platform, birdnet-market, mention-mate
│   ├── appsets/               # ApplicationSet: platform, all-projects
│   └── shared-gateway-app.yaml
├── helm-charts/app/           # base chart gốc (canonical) — đồng bộ sang các branch project
├── platform/gateway/          # Gateway dùng chung (shared-gw)
└── scripts/seal.sh            # bọc kubeseal

project-<name> (mỗi project 1 branch)
├── helm-charts/app/           # bản copy của chart (đồng bộ từ main)
└── apps/<app>/overlays/<env>/values.yaml
```

> **Vì sao chart copy sang branch?** ArgoCD multi-source không cho `$values` trỏ tới *revision khác*
> của *cùng 1 repo*. Nên chart và values phải cùng revision = branch project. Sửa chart trên `main`
> rồi đồng bộ sang branch (`git checkout <branch> && git checkout main -- helm-charts/app && commit`).

## Bootstrap (1 lệnh tay duy nhất)

```bash
kubectl apply -f root-application.yaml
```

`root` → `bootstrap/` (recurse), theo sync-wave:

| Wave | Thành phần |
|------|------------|
| -2 | AppProjects |
| -1 | appset `platform` → sealed-secrets, kgateway-crds, kgateway |
| 0  | appset `all-projects` → quét branch project, sinh 1 Application / (app, env) |
| 1  | `shared-gateway` → Gateway `shared-gw` (kgateway-system) |

| Project (branch) | Apps | Application |
|---|---|---|
| birdnet-market | `birdnet-market-frontend`, `birdnet-market-backend` | 2 × 3 env = 6 |
| mention-mate | `mention-mate-app` (backend + worker) | 1 × 3 env = 3 |

→ tổng **9 Application**, mỗi cái = `helm-charts/app` + `values.yaml` (cùng lấy từ branch project đó).

## Base chart `app`

1 chart cho mọi app, render theo map `components`:

- mỗi component → 1 **Deployment** (+ **Service** nếu có `port`, + **ConfigMap** nếu có `config`, + **HTTPRoute** nếu `httpRoute.enabled`)
- 1 component = single-deployment; nhiều component = multi-deployment
- **1 SealedSecret** dùng chung cho cả release (`envFrom` vào mọi component)
- HTTPRoute gắn vào **Gateway dùng chung** `shared-gw` (kgateway-system)

Cấu trúc `components` đầy đủ: xem `helm-charts/app/values.yaml`.

## Overlay env (trên branch project)

```
apps/<app>/overlays/<env>/values.yaml   # components, config, sealedSecret.encryptedData
```

`all-projects` dùng git **directories generator** quét `apps/*/overlays/*` (không cần `config.yaml`).
Application = `{project}-{app}-{env}`, namespace = `{project}-{env}`. Cả chart và values lấy từ **cùng
branch project** (cùng revision); khác biệt app/env nằm hết trong `values.yaml` ⇒ **promotion = sửa `values.yaml`**.

## SealedSecret

`encryptedData` trong repo là **PLACEHOLDER** (không decrypt được). Sinh ciphertext thật bằng kubeseal —
scope `strict` gắn ciphertext theo đúng **name + namespace**, nên tên secret phải khớp `<release>-secret`:

```bash
# release = <project>-<app>-<env> ; secret = <release>-secret ; ns = <project>-<env>
./scripts/seal.sh mention-mate-dev mention-mate-mention-mate-app-dev-secret DB_PASSWORD=... API_KEY=...
# dán block encryptedData trả về vào values.yaml của env đó, commit, push.
```

## Đăng ký repo cho ArgoCD

```bash
argocd repo add https://github.com/ongtungduong/gitops-training.git --username '<user>' --password '<PAT>'
argocd repo add cr.kgateway.dev/kgateway-dev/charts --type helm --enable-oci   # cho kgateway
```

repo-server cần egress tới GitHub, `cr.kgateway.dev`, `bitnami-labs.github.io`. ArgoCD **≥ 3.1** (native OCI Helm); lab pin **3.3.x**.
