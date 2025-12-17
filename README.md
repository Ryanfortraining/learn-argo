# ArgoCD 學習專案

這是一個完整的 ArgoCD GitOps 學習範例，展示如何在 EKS 上部署 ArgoCD 並實現基礎設施即程式碼（Infrastructure as Code）。

## 目錄結構

```
learn-argo/
└── apps/
    ├── argocd/             # ArgoCD Helm Chart - ArgoCD 管理自己
    │   ├── Chart.yaml      # Chart 元資料與依賴定義
    │   ├── values.yaml     # ArgoCD 配置（LoadBalancer, HTTP mode）
    │   ├── charts/         # 官方 argo-cd chart 依賴
    │   └── templates/      # Helm 模板與說明
    └── nginx/              # 簡單的 nginx 示例應用
        ├── deployment.yaml # Nginx Deployment (3 replicas)
        └── service.yaml    # ClusterIP Service
```

---

## 前置需求

- ✅ AWS EKS 集群（或其他 Kubernetes 集群）
- ✅ `kubectl` 已配置並可存取集群
- ✅ `helm` 3.x 已安裝
- ✅ `git` 已安裝
- ✅ GitHub 帳號（或其他 Git 服務）

---

## 完整部署步驟

### 步驟 1：初始部署 ArgoCD

#### 1.1 克隆此專案

```bash
git clone https://github.com/Ryanfortraining/learn-argo.git
cd learn-argo
```

#### 1.2 新增 ArgoCD Helm Repository

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

#### 1.3 下載 Chart 依賴

```bash
cd apps/argocd
helm dependency update
cd ../..
```

#### 1.4 建立 Namespace 並安裝 ArgoCD

```bash
# 建立 argocd namespace
kubectl create namespace argocd

# 使用 Helm 安裝 ArgoCD
helm install argocd apps/argocd --namespace argocd
```

#### 1.5 等待 ArgoCD 啟動

```bash
# 查看 Pod 狀態（等待所有 Pod 變成 Running）
kubectl get pods -n argocd --watch
```

#### 1.6 取得 LoadBalancer 位址

```bash
# 取得 LoadBalancer URL（大約需要 2 分鐘）
kubectl get svc -n argocd argocd-server

# 取得外部 IP/域名
export ARGOCD_SERVER=$(kubectl get svc -n argocd argocd-server -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "ArgoCD URL: http://$ARGOCD_SERVER"
```

#### 1.7 取得初始 Admin 密碼

```bash
kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
echo
```

#### 1.8 登入 ArgoCD UI

在瀏覽器開啟：`http://<LoadBalancer-IP>`

- **使用者名稱**：`admin`
- **密碼**：上一步取得的密碼

---

### 步驟 2：配置 ArgoCD 管理自己（GitOps 最佳實踐）

#### 2.1 移除 Helm 管理權

讓 ArgoCD 從 Git 接管自己的配置：

```bash
# 移除 Helm release 的所有權標籤（保留所有資源）
kubectl delete secret -n argocd -l owner=helm,name=argocd
```

#### 2.2 在 ArgoCD UI 中建立 Application

1. 點擊左側 **"+ NEW APP"**
2. 填寫以下資訊：

**GENERAL**
```
Application Name: argocd
Project Name: default
Sync Policy: Manual
```

**SOURCE**
```
Repository URL: https://github.com/Ryanfortraining/learn-argo
Revision: main
Path: apps/argocd
```

**HELM**（重要！）
```
Release Name: argocd
```

**DESTINATION**
```
Cluster URL: https://kubernetes.default.svc
Namespace: argocd
```

3. 點擊 **"CREATE"**

#### 2.3 同步 Application

1. 點擊 `argocd` Application
2. 點擊右上角 **"SYNC"**
3. 勾選：
   - ✅ **PRUNE RESOURCES**
   - ✅ **APPLY OUT OF SYNC ONLY**
4. 點擊 **"SYNCHRONIZE"**

✅ **完成！ArgoCD 現在通過 Git 管理自己的配置。**

---

### 步驟 3：部署示例應用（nginx）

#### 3.1 在 ArgoCD UI 中建立 nginx Application

1. 點擊 **"+ NEW APP"**
2. 填寫資訊：

**GENERAL**
```
Application Name: nginx-demo
Project Name: default
Sync Policy: Automatic（啟用自動同步）
```

**SOURCE**
```
Repository URL: https://github.com/Ryanfortraining/learn-argo
Revision: main
Path: apps/nginx
```

**DESTINATION**
```
Cluster URL: https://kubernetes.default.svc
Namespace: default
```

3. 點擊 **"CREATE"**

#### 3.2 驗證部署

```bash
# 查看 nginx Pods
kubectl get pods -l app=nginx

# 查看 Service
kubectl get svc nginx-demo
```

你應該看到 3 個 nginx Pod 正在運行。

---

## GitOps 工作流示範

### 情境 1：擴展 nginx 應用

#### 修改配置

編輯 `apps/nginx/deployment.yaml`：

```yaml
spec:
  replicas: 5  # 從 3 改成 5
```

#### 提交並推送

```bash
git add apps/nginx/deployment.yaml
git commit -m "Scale nginx to 5 replicas"
git push
```

#### 觀察自動同步

- 等待 3 分鐘（ArgoCD 預設輪詢間隔）
- 或在 UI 中手動點擊 **"REFRESH"**
- ArgoCD 會自動檢測變更並部署
- 觀察新的 Pod 被建立

```bash
kubectl get pods -l app=nginx --watch
```

---

### 情境 2：修改 ArgoCD 配置

#### 修改配置

編輯 `apps/argocd/values.yaml`：

```yaml
argo-cd:
  global:
    domain: argocd.local

  server:
    replicas: 2  # 新增這行，啟動 2 個 server replica
    extraArgs:
      - --insecure

    service:
      type: LoadBalancer
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
        service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
```

#### 提交並推送

```bash
git add apps/argocd/values.yaml
git commit -m "Scale ArgoCD server to 2 replicas"
git push
```

#### 手動同步

1. 在 ArgoCD UI 中打開 `argocd` Application
2. 等待狀態變為 **OutOfSync**
3. 點擊 **"SYNC"** → **"SYNCHRONIZE"**
4. 觀察第 2 個 argocd-server Pod 啟動

```bash
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server
```

---

## 學習目標總結

通過這個專案，你將學會：

- ✅ 使用 Helm Chart 部署 ArgoCD 到 Kubernetes
- ✅ 配置 LoadBalancer 與 HTTP 模式存取
- ✅ 實現 **App of Apps** 模式（ArgoCD 管理自己）
- ✅ 理解 **Git 是唯一真相來源**的概念
- ✅ 體驗完整的 GitOps 工作流程
- ✅ 掌握聲明式配置管理
- ✅ 觀察自動同步與漂移檢測
- ✅ 透過 Git 歷史實現配置回滾

---

## 常見問題與故障排查

### 問題 1：LoadBalancer 一直處於 Pending 狀態

**原因**：集群沒有 LoadBalancer Controller

**解決方案（EKS）**：
```bash
# 安裝 AWS Load Balancer Controller
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"
```

---

### 問題 2：無法存取 ArgoCD UI

**檢查步驟**：

```bash
# 1. 確認 Pod 都在運行
kubectl get pods -n argocd

# 2. 確認 Service 狀態
kubectl get svc -n argocd argocd-server

# 3. 查看 argocd-server 日誌
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server

# 4. 測試連線
curl -I http://<LoadBalancer-IP>
```

---

### 問題 3：Application 顯示 OutOfSync 但看起來一樣

**原因**：可能是 Labels 或 Annotations 差異

**解決方案**：
1. 點擊 **"APP DIFF"** 查看具體差異
2. 如果差異可接受，點擊 **"SYNC"**
3. 或修改 Git 中的配置使其一致

---

### 問題 4：同步後 Pod 重啟

**原因**：ConfigMap 或 Secret 變更會觸發 Pod 重啟

**這是正常的**，Kubernetes 會：
1. 滾動更新 Deployment
2. 逐步替換舊 Pod
3. 確保服務不中斷

---

### 問題 5：如何回滾配置

**透過 Git 回滾**：

```bash
# 查看歷史提交
git log --oneline

# 回滾到特定版本
git revert <commit-hash>
git push

# ArgoCD 會自動同步回滾後的配置
```

---

## 進階學習資源

- 📖 [ArgoCD 官方文件](https://argo-cd.readthedocs.io/)
- 📖 [GitOps 最佳實踐](https://opengitops.dev/)
- 📖 [Helm Chart 開發指南](https://helm.sh/docs/chart_template_guide/)
- 🎥 [ArgoCD 教學影片](https://www.youtube.com/c/ArgoProjIO)

---

## 下一步

完成基礎學習後，你可以探索：

1. **Sync Waves**：控制資源部署順序
2. **Sync Hooks**：在部署前後執行任務
3. **Multi-cluster 管理**：管理多個 Kubernetes 集群
4. **ApplicationSet**：自動化 Application 建立
5. **Notifications**：配置 Slack/Email 通知
6. **Image Updater**：自動更新容器映像

---

## 貢獻與授權

這是一個學習專案，歡迎提交 Issue 或 Pull Request。

**Linus says**: "Talk is cheap. Show me the code."

---

## 聯絡方式

- GitHub: [Ryanfortraining](https://github.com/Ryanfortraining)
- 專案連結: [learn-argo](https://github.com/Ryanfortraining/learn-argo)

🤖 Generated with [Claude Code](https://claude.com/claude-code)
