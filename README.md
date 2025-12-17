# ArgoCD Learning Repository

这是一个用于学习 ArgoCD GitOps 的示例 repository。

## 目录结构

```
learn-argo/
└── apps/
    ├── argocd/             # ArgoCD Helm Chart - ArgoCD 管理自己
    │   ├── Chart.yaml
    │   ├── values.yaml
    │   └── templates/
    └── nginx/              # 简单的 nginx 示例应用
        ├── deployment.yaml # Nginx Deployment (3 replicas)
        └── service.yaml    # ClusterIP Service
```

## 如何使用

### 1. 在 ArgoCD 中创建 Application

- **Application Name**: nginx-demo
- **Project**: default
- **Sync Policy**: Manual（或 Automatic）
- **Repository URL**: https://github.com/Ryanfortraining/learn-argo
- **Path**: apps/nginx
- **Cluster**: https://kubernetes.default.svc
- **Namespace**: default

### 2. 同步应用

在 ArgoCD UI 中点击 "Sync" 按钮，观察应用部署到 K8s。

### 3. 体验 GitOps

修改 `apps/nginx/deployment.yaml` 中的 `replicas` 数量：
- 从 2 改成 3
- Commit 并 push 到 GitHub
- ArgoCD 会自动检测变化并同步（如果启用了 Auto-Sync）

## 学习目标

- ✅ 理解 Git 作为唯一真相来源
- ✅ 体验声明式配置管理
- ✅ 观察 ArgoCD 的自动同步和漂移检测
- ✅ 学习应用回滚（通过 Git history）
- ✅ ArgoCD 管理自己（App of Apps 模式）

---

## 进阶：ArgoCD 管理自己

### 为什么要这样做？

- **Infrastructure as Code**：所有配置都在 Git 中
- **可审计**：每次配置变更都有 Git 历史
- **可回滚**：通过 Git revert 回滚配置
- **多环境管理**：同一份配置管理 dev/staging/prod

### 在 ArgoCD 中创建 Application（管理 ArgoCD 自己）

- **Application Name**: argocd-self
- **Project**: default
- **Sync Policy**: Manual（建议）
- **Repository URL**: https://github.com/Ryanfortraining/learn-argo
- **Path**: apps/argocd
- **Cluster**: https://kubernetes.default.svc
- **Namespace**: argocd

⚠️ **注意**：因为这是 Helm Chart，需要手动下载 dependencies：

```bash
cd apps/argocd
helm dependency update
git add charts/
git commit -m "Add argocd dependencies"
git push
```

然后在 ArgoCD UI 中同步即可。

### 修改 ArgoCD 配置示例

想要禁用 TLS？编辑 `apps/argocd/values.yaml`：

```yaml
argo-cd:
  server:
    extraArgs:
      - --insecure  # 添加或删除这行
```

Commit → Push → ArgoCD 自动同步 → ArgoCD 重启并应用新配置！

---

**Linus says**: "Talk is cheap. Show me the code."
