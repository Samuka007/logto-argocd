# Logto PostgreSQL 集群

Logto 独享的 PostgreSQL 数据库集群，基于 CloudNativePG。

## 特性

- 独享资源，完全隔离
- 高可用（生产 3 实例）
- 自动备份到 S3
- PgBouncer 连接池
- 多环境支持（dev/prod）

## 集群信息

- **集群名**: `logto-postgres`
- **数据库**: `logto`
- **用户**: `logto`
- **服务端点**:
  - 读写: `logto-postgres-rw:5432`
  - 只读: `logto-postgres-ro:5432`
  - 连接池: `logto-postgres-pooler-rw:5432`

## 快速部署

### 1. 生成备份凭据（仅 Prod）

**CNPG 自动生成数据库凭据**，无需手动创建。只需为 Prod 生成 S3 备份凭据：

```bash
kubectl create secret generic logto-backup-storage-credentials \
  --from-literal=ACCESS_KEY_ID="YOUR_S3_KEY" \
  --from-literal=ACCESS_SECRET_KEY="YOUR_S3_SECRET" \
  --namespace=production --dry-run=client -o yaml | kubeseal > backup-sealed.yaml
```

复制 `encryptedData` 到 `overlays/prod/sealed-secret-backup.yaml`

### 2. 部署

```bash
# Dev
kubectl apply -k overlays/dev

# Prod
kubectl apply -k overlays/prod
```

### 3. 验证

```bash
# 查看集群状态
kubectl get cluster -n <namespace> logto-postgres

# 查看 CNPG 自动生成的 Secret
kubectl get secret -n <namespace> logto-postgres-app

# 连接测试
kubectl exec -n <namespace> -it logto-postgres-1 -- psql -U logto -d logto
```

## 目录结构

```
logto-postgres-cluster/
├── base/
│   ├── cluster.yaml           # 集群配置
│   └── pooler.yaml            # 连接池
│
├── overlays/dev/
│   └── patches/
│       └── cluster-resources.yaml  # 1 instance, 小资源
│
└── overlays/prod/
    ├── sealed-secret-backup.yaml   # S3 凭据（仅 Prod）
    ├── backup-schedule.yaml        # 备份调度（仅 Prod）
    └── patches/
        └── cluster-resources.yaml  # 3 instances, 大资源
```

**注**：数据库凭据由 CNPG 自动生成（Secret: `logto-postgres-app`）

## 环境配置

| 配置 | Dev | Prod |
|------|-----|------|
| Namespace | dev | production |
| 实例数 | 1 | 3 |
| CPU | 250m-500m | 500m-2000m |
| 内存 | 256Mi-512Mi | 512Mi-2Gi |
| 存储 | 10Gi | 20Gi |
| 备份 | ❌ 禁用 | ✅ 每日，保留30天 |

## 常用操作

### 手动备份（仅 Prod）

```bash
# 仅在 prod 环境可用，dev 环境未配置备份
kubectl cnpg backup logto-postgres -n production
```

### 查看备份（仅 Prod）

```bash
kubectl get backup -n production
```

### 扩展集群

修改 `overlays/prod/patches/cluster-resources.yaml`:

```yaml
spec:
  instances: 5  # 改为 5
```

### 查看日志

```bash
kubectl logs -n <namespace> -l cnpg.io/cluster=logto-postgres --tail=100
```

## 故障排查

### 集群未启动

```bash
kubectl describe cluster -n <namespace> logto-postgres
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

### SealedSecret 问题

```bash
kubectl logs -n sealed-secrets -l app.kubernetes.io/name=sealed-secrets
kubectl get sealedsecret -n <namespace>
```

### 连接问题

```bash
# DNS 测试
kubectl run -it --rm debug --image=busybox -n <namespace> -- \
  nslookup logto-postgres-rw

# 连接测试
kubectl run -it --rm debug --image=postgres:16 -n <namespace> -- \
  psql -h logto-postgres-rw -U logto -d logto
```
