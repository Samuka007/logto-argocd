# ArgoCD 配置

## 目录结构

```
argocd/
├── app-of-apps/
│   ├── root.yaml           # Dev 环境 App of Apps
│   └── root-prod.yaml      # Prod 环境 App of Apps
│
└── applications/
    ├── dev/
    │   ├── logto-postgres-cluster.yaml
    │   └── logto-app.yaml
    │
    └── prod/
        ├── logto-postgres-cluster.yaml
        └── logto-app.yaml
```

## 快速部署

### Dev 环境

```bash
kubectl apply -f argocd/app-of-apps/root.yaml
```

### Prod 环境

```bash
kubectl apply -f argocd/app-of-apps/root-prod.yaml
```

### 单独部署 Logto 集群

```bash
# Dev
kubectl apply -f argocd/applications/dev/logto-postgres-cluster.yaml

# Prod
kubectl apply -f argocd/applications/prod/logto-postgres-cluster.yaml
```


## 配置说明

### 数据库凭据

CNPG 自动生成 `logto-postgres-app` Secret，Logto 应用直接引用。

**Dev 环境**: 无需任何手动密钥。

**Prod 环境**: 仅需在 `kustomize/logto-postgres-cluster/overlays/prod/sealed-secret-backup.yaml` 中配置 S3 备份凭据。

### 配置修改

修改 repoURL（所有文件）：

```yaml
repoURL: https://github.com/your-org/your-repo.git  # ← 改成你的仓库
```
