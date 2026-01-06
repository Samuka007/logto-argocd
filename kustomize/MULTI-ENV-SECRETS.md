# 多环境密钥管理

## 核心变更

**数据库凭据由 CNPG 自动生成**，不再需要手动管理 SealedSecret！

仅需为 **Prod 环境**生成 S3 备份凭据（Dev 不启用备份）。

## SealedSecret 原理

SealedSecret 加密时绑定 **namespace + secret name**：

```
加密 = f(controller-key, namespace, secret-name, 明文)
```

**结果：**
- 在 `dev` namespace 加密的密钥只能在 `dev` 解密
- 在 `production` namespace 加密的密钥只能在 `production` 解密
- **相同明文在不同 namespace 的加密结果完全不同**

## CNPG 自动生成的 Secret

CNPG Operator 会自动创建：
- **Secret 名称**: `<cluster-name>-app`
- **我们的集群**: `logto-postgres-app`
- **包含的 keys**:
  - `username` - 数据库用户名
  - `password` - 自动生成的强密码
  - 其他可能的 keys...

Logto 应用直接引用此 Secret，无需手动创建和管理密码。

## 目录结构

```
logto-postgres-cluster/
├── base/                    # 只放非敏感配置
│   ├── cluster.yaml
│   └── ...
│
├── overlays/dev/
│   ├── sealed-secret.yaml        ← Dev 独立密钥
│   └── ...
│
└── overlays/prod/
    ├── sealed-secret.yaml        ← Prod 独立密钥（不同于 dev）
    └── ...
```

## 仅需生成的密钥

### 1. Prod 环境 S3 备份凭据

```bash
# S3 备份凭据（仅 Prod）
kubectl create secret generic logto-backup-storage-credentials \
  --from-literal=ACCESS_KEY_ID="PROD_S3_KEY" \
  --from-literal=ACCESS_SECRET_KEY="PROD_S3_SECRET" \
  --namespace=production \
  --dry-run=client -o yaml | \
  kubeseal --controller-namespace=sealed-secrets --format=yaml \
  > /tmp/prod-backup-sealed.yaml
```

### 2. 更新配置

从生成的文件中复制 `spec.encryptedData` 到：
- `overlays/prod/sealed-secret-backup.yaml`

**Dev 环境**：无需任何手动创建的密钥！

## 验证

```bash
# 查看 CNPG 自动生成的 Secret
kubectl get secret -n dev logto-postgres-app
kubectl get secret -n production logto-postgres-app

# 查看 Secret 内容（需要 decode）
kubectl get secret -n dev logto-postgres-app -o jsonpath='{.data.username}' | base64 -d
```

## 密钥轮转

**数据库密码**：由 CNPG 管理，通常不需要轮转。如需轮转，参考 CNPG 官方文档。

**S3 备份凭据**（仅 Prod）：
1. 生成新的 S3 凭据
2. 更新 `overlays/prod/sealed-secret-backup.yaml`
3. Git commit + push
4. ArgoCD 自动同步

## 故障排查

**SealedSecret 无法解密：**
```bash
kubectl logs -n sealed-secrets -l app.kubernetes.io/name=sealed-secrets
```

常见原因：
- namespace 不匹配
- secret name 不匹配
- sealed-secrets controller 版本不兼容
