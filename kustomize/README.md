# Kustomize 配置

## 目录

```
kustomize/
├── logto-postgres-cluster/     # Logto 独享数据库集群
│   ├── base/                   # 共享配置
│   └── overlays/
│       ├── dev/                # Dev 环境（密钥独立）
│       └── prod/               # Prod 环境（密钥独立）
│
└── logto-app/                  # Logto 应用
    ├── base/
    └── overlays/
        ├── dev/
        └── prod/
```

## 快速开始

### 1. 生成 S3 备份凭据（仅 Prod）

**CNPG 自动生成数据库凭据**，无需手动创建。只需为 Prod 生成备份凭据：

```bash
kubectl create secret generic logto-backup-storage-credentials \
  --from-literal=ACCESS_KEY_ID="YOUR_S3_KEY" \
  --from-literal=ACCESS_SECRET_KEY="YOUR_S3_SECRET" \
  --namespace=production --dry-run=client -o yaml | \
  kubeseal > prod-backup-sealed.yaml
```

复制 `encryptedData` 到：`logto-postgres-cluster/overlays/prod/sealed-secret-backup.yaml`

注：Dev 环境不启用备份，无需任何凭据。

### 2. 验证

```bash
kustomize build kustomize/logto-postgres-cluster/overlays/dev
kustomize build kustomize/logto-postgres-cluster/overlays/prod
```

### 3. 部署

```bash
kubectl apply -k kustomize/logto-postgres-cluster/overlays/dev
kubectl apply -k kustomize/logto-postgres-cluster/overlays/prod
```

或使用 ArgoCD（见 `argocd/README.md`）

## 数据库凭据管理

**CNPG 自动生成**：集群启动后，CNPG Operator 自动创建 `logto-postgres-app` Secret，包含数据库凭据。Logto 应用直接引用此 Secret，无需手动管理密码。

## 环境差异

| 配置 | Dev | Prod |
|------|-----|------|
| Namespace | dev | production |
| 实例数 | 1 | 3 |
| CPU | 250m-500m | 500m-2000m |
| 内存 | 256Mi-512Mi | 512Mi-2Gi |
| 存储 | 10Gi | 20Gi |
| 备份 | ❌ 禁用 | ✅ 每日，保留30天 |


## 故障排查

```bash
# 集群状态
kubectl get cluster -n <namespace> logto-postgres

# Pod 日志
kubectl logs -n <namespace> -l cnpg.io/cluster=logto-postgres

# 连接测试
kubectl exec -n <namespace> -it logto-postgres-1 -- psql -U logto -d logto
```
