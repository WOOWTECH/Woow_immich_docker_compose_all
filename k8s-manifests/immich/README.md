# Immich K3s/Kubernetes 部署指南

[English](#english) | [中文](#中文)

---

## English

### Overview

Self-hosted photo and video backup solution with AI-powered search, face recognition, and automatic organization. Immich is a Google Photos alternative that keeps your memories private and under your control. This deployment includes PostgreSQL with pgvecto-rs for vector similarity search, Redis for job queuing, and a dedicated machine learning container for AI processing (CLIP embeddings, face detection).

> **GitHub Repo (Podman/Docker):** [Woow_immich_docker_compose_all](https://github.com/WOOWTECH/Woow_immich_docker_compose_all)

### Architecture

```
                           Immich K3s Architecture
 ============================================================================

   External Access                K3s Cluster (namespace: immich)
  +----------------+       +--------------------------------------------------+
  |                |       |                                                  |
  |  Browser /     |       |   +------------------------------------------+   |
  |  Mobile App    |       |   |  Service: immich-server                  |   |
  |  :30283  ------+--NodePort->|  ClusterIP :2283                        |   |
  |                |       |   +-------------------+----------------------+   |
  +----------------+       |                       |                          |
                           |                       v                          |
                           |   +------------------------------------------+   |
                           |   |  Pod: immich-server (Deployment)         |   |
                           |   |  Image: immich-app/immich-server         |   |
                           |   |  Port: 2283                              |   |
                           |   |  Volume: /usr/src/app/upload (50Gi)      |   |
                           |   +--------+-----------+---------------------+   |
                           |            |           |                         |
                           |            v           v                         |
                           |   +----------------+  +------------------------+ |
                           |   | Pod: postgres  |  | Pod: immich-redis      | |
                           |   | (StatefulSet)  |  | (Deployment)           | |
                           |   | Image:         |  | Image: redis:7-alpine  | |
                           |   | pgvecto-rs     |  | Port: 6379             | |
                           |   | :pg16-v0.3.0   |  +------------------------+ |
                           |   | Port: 5432     |                             |
                           |   | Volume: 10Gi   |  +------------------------+ |
                           |   +----------------+  | Pod: immich-ml         | |
                           |                       | (Deployment)           | |
                           |   immich-server ------>| Image: immich-         | |
                           |   sends ML jobs       |   machine-learning     | |
                           |                       | Port: 3003             | |
                           |                       | Volume: /cache (10Gi)  | |
                           |                       +------------------------+ |
                           |                                                  |
                           +--------------------------------------------------+
```

### Features

- AI-powered photo search using CLIP embeddings
- Facial recognition and automatic grouping
- Automatic photo/video backup from iOS and Android
- Timeline view with map integration
- Shared albums and external link sharing
- Dedicated ML container for AI model inference
- PostgreSQL with pgvecto-rs for vector similarity search
- Redis-backed job queue for background processing

### Quick Start

```bash
# 1. Update secrets before deploying
nano k8s-manifests/immich/secret.yaml

# 2. Deploy all Immich components
kubectl apply -k k8s-manifests/immich/

# 3. Verify pods are running
kubectl -n immich get pods

# 4. Watch server logs for initial setup
kubectl -n immich logs deploy/immich-server -f
```

### Configuration

#### Environment Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `DB_HOSTNAME` | PostgreSQL hostname | `immich-postgres` | Yes |
| `REDIS_HOSTNAME` | Redis hostname | `immich-redis` | Yes |
| `TZ` | Timezone | `Asia/Taipei` | No |

#### Secrets

Edit `secret.yaml` before deploying. The following secrets must be configured:

| Secret Key | Description | Default (change me!) |
|------------|-------------|----------------------|
| `DB_PASSWORD` | PostgreSQL password | `changeme-please` |
| `DB_USERNAME` | PostgreSQL username | `immich` |
| `DB_DATABASE_NAME` | PostgreSQL database name | `immich` |

```bash
# Edit the secret file
nano k8s-manifests/immich/secret.yaml
```

### Accessing the Service

| Endpoint | URL | Protocol |
|----------|-----|----------|
| Immich Web UI | `http://<node-ip>:30283` | HTTP (NodePort) |
| Internal (cluster) | `http://immich-server.immich.svc.cluster.local:2283` | HTTP |

On first access, create an admin account through the web interface. Then install the Immich mobile app (iOS/Android) and configure it to connect to your server URL.

### Data Persistence

| PVC Name | Mount Path | Size | Purpose |
|----------|------------|------|---------|
| `immich-postgres-data` | `/var/lib/postgresql/data` | 10Gi | PostgreSQL database (metadata, vectors, face data) |
| `immich-upload-data` | `/usr/src/app/upload` | 50Gi | Original photos and video files |
| `immich-model-cache` | `/cache` | 10Gi | Downloaded ML models (CLIP, facial recognition) |

All PVCs use the `local-path` storage class (k3s default).

### Backup & Restore

#### Backup

```bash
# 1. Backup PostgreSQL database (metadata, face data, vector embeddings)
kubectl -n immich exec sts/immich-postgres -- pg_dump -U immich immich > immich-db-backup.sql

# 2. Backup uploaded media files
kubectl -n immich exec deploy/immich-server -- tar czf /tmp/upload-backup.tar.gz /usr/src/app/upload
kubectl -n immich cp immich/<server-pod>:/tmp/upload-backup.tar.gz ./upload-backup.tar.gz
```

#### Restore

```bash
# 1. Restore PostgreSQL database
kubectl -n immich exec -i sts/immich-postgres -- psql -U immich immich < immich-db-backup.sql

# 2. Restore uploaded media
kubectl -n immich cp ./upload-backup.tar.gz immich/<server-pod>:/tmp/upload-backup.tar.gz
kubectl -n immich exec deploy/immich-server -- tar xzf /tmp/upload-backup.tar.gz -C /

# 3. Restart to reindex
kubectl -n immich rollout restart deploy/immich-server
```

### Useful Commands

```bash
# Check all pod status
kubectl -n immich get pods

# View server logs
kubectl -n immich logs deploy/immich-server -f

# View ML processing logs
kubectl -n immich logs deploy/immich-machine-learning -f

# View PostgreSQL logs
kubectl -n immich logs sts/immich-postgres -f

# Check Redis connectivity
kubectl -n immich exec deploy/immich-redis -- redis-cli ping

# Check upload storage usage
kubectl -n immich exec deploy/immich-server -- df -h /usr/src/app/upload

# Check ML pod resource usage
kubectl -n immich top pod -l component=machine-learning

# Restart server
kubectl -n immich rollout restart deploy/immich-server
```

### Troubleshooting

#### Machine learning pod OOMKilled

The ML container downloads and loads AI models that require significant memory. Increase the memory limit in `machine-learning-deployment.yaml`:

```yaml
resources:
  limits:
    memory: 6Gi  # Increase from 4Gi
```

```bash
kubectl apply -k k8s-manifests/immich/
```

#### PostgreSQL fails to start with pgvecto-rs

The PostgreSQL container uses custom startup arguments for the pgvecto-rs extension. Verify the pod logs:

```bash
kubectl -n immich logs sts/immich-postgres
```

Common issues:
- Corrupt data directory: Delete the PVC and re-create
- Incompatible pgvecto-rs version: Check the image tag matches the data directory version

#### Slow face recognition or CLIP search

```bash
# Check ML pod resource usage
kubectl -n immich top pod -l component=machine-learning

# View ML processing logs
kubectl -n immich logs deploy/immich-machine-learning -f
```

#### Redis connection issues

```bash
kubectl -n immich get pods -l component=redis
kubectl -n immich exec deploy/immich-redis -- redis-cli ping
```

#### Upload fails or times out

If uploads fail for large files, check the Immich server logs and verify the PVC has sufficient space:

```bash
kubectl -n immich exec deploy/immich-server -- df -h /usr/src/app/upload
```

### File Structure

```
immich/
├── configmap.yaml                    # Environment variables (DB host, Redis host, timezone)
├── kustomization.yaml                # Kustomize resource list
├── machine-learning-deployment.yaml  # Deployment for ML container (CLIP, face detection)
├── machine-learning-service.yaml     # ClusterIP service for ML (3003)
├── namespace.yaml                    # Namespace: immich
├── postgres-service.yaml             # ClusterIP service for PostgreSQL (5432)
├── postgres-statefulset.yaml         # StatefulSet for PostgreSQL + pgvecto-rs
├── pvc.yaml                          # PersistentVolumeClaims (DB, uploads, model cache)
├── README.md                         # This file
├── redis-deployment.yaml             # Deployment for Redis job queue
├── redis-service.yaml                # ClusterIP service for Redis (6379)
├── secret.yaml                       # Database credentials (change before deploy!)
├── server-deployment.yaml            # Deployment for Immich server
└── server-service.yaml               # NodePort service (30283 -> 2283)
```

---

## 中文

### 概述

Immich 是自架的照片與影片備份解決方案，具備 AI 驅動搜尋、人臉辨識及自動整理功能。作為 Google Photos 的替代方案，讓您的回憶保持私密且完全由您掌控。本部署包含 PostgreSQL（含 pgvecto-rs 向量相似度搜尋）、Redis（任務佇列）及專用機器學習容器（CLIP 嵌入、人臉偵測）。

> **GitHub Repo (Podman/Docker):** [Woow_immich_docker_compose_all](https://github.com/WOOWTECH/Woow_immich_docker_compose_all)

### 架構圖

```
                           Immich K3s 架構
 ============================================================================

   外部存取                     K3s 叢集 (namespace: immich)
  +----------------+       +--------------------------------------------------+
  |                |       |                                                  |
  |  瀏覽器 /      |       |   +------------------------------------------+   |
  |  行動應用程式   |       |   |  Service: immich-server                  |   |
  |  :30283  ------+--NodePort->|  ClusterIP :2283                        |   |
  |                |       |   +-------------------+----------------------+   |
  +----------------+       |                       |                          |
                           |                       v                          |
                           |   +------------------------------------------+   |
                           |   |  Pod: immich-server (Deployment)         |   |
                           |   |  映像: immich-app/immich-server          |   |
                           |   |  埠號: 2283                              |   |
                           |   |  磁碟區: /usr/src/app/upload (50Gi)      |   |
                           |   +--------+-----------+---------------------+   |
                           |            |           |                         |
                           |            v           v                         |
                           |   +----------------+  +------------------------+ |
                           |   | Pod: postgres  |  | Pod: immich-redis      | |
                           |   | (StatefulSet)  |  | (Deployment)           | |
                           |   | 映像:          |  | 映像: redis:7-alpine   | |
                           |   | pgvecto-rs     |  | 埠號: 6379             | |
                           |   | :pg16-v0.3.0   |  +------------------------+ |
                           |   | 埠號: 5432     |                             |
                           |   | 磁碟區: 10Gi   |  +------------------------+ |
                           |   +----------------+  | Pod: immich-ml         | |
                           |                       | (Deployment)           | |
                           |   immich-server ------>| 映像: immich-          | |
                           |   傳送 ML 任務        |   machine-learning     | |
                           |                       | 埠號: 3003             | |
                           |                       | 磁碟區: /cache (10Gi)  | |
                           |                       +------------------------+ |
                           |                                                  |
                           +--------------------------------------------------+
```

### 功能特色

- AI 驅動照片搜尋，使用 CLIP 嵌入
- 人臉辨識與自動分組
- 自動從 iOS 和 Android 備份照片/影片
- 時間軸檢視與地圖整合
- 共享相簿與外部連結分享
- 專用 ML 容器進行 AI 模型推論
- PostgreSQL 搭配 pgvecto-rs 實現向量相似度搜尋
- Redis 支援的背景任務佇列

### 快速開始

```bash
# 1. 部署前先更新密鑰
nano k8s-manifests/immich/secret.yaml

# 2. 部署所有 Immich 元件
kubectl apply -k k8s-manifests/immich/

# 3. 確認 Pod 運作中
kubectl -n immich get pods

# 4. 查看伺服器啟動日誌
kubectl -n immich logs deploy/immich-server -f
```

### 設定

#### 環境變數

| 變數 | 說明 | 預設值 | 必填 |
|------|------|--------|------|
| `DB_HOSTNAME` | PostgreSQL 主機名稱 | `immich-postgres` | 是 |
| `REDIS_HOSTNAME` | Redis 主機名稱 | `immich-redis` | 是 |
| `TZ` | 時區 | `Asia/Taipei` | 否 |

#### Secrets（密鑰）

部署前請編輯 `secret.yaml`，需設定以下密鑰：

| Secret Key | 說明 | 預設值（請變更！） |
|------------|------|-------------------|
| `DB_PASSWORD` | PostgreSQL 密碼 | `changeme-please` |
| `DB_USERNAME` | PostgreSQL 使用者名稱 | `immich` |
| `DB_DATABASE_NAME` | PostgreSQL 資料庫名稱 | `immich` |

```bash
# 編輯密鑰檔案
nano k8s-manifests/immich/secret.yaml
```

### 存取服務

| 端點 | URL | 協定 |
|------|-----|------|
| Immich Web UI | `http://<node-ip>:30283` | HTTP (NodePort) |
| 叢集內部 | `http://immich-server.immich.svc.cluster.local:2283` | HTTP |

首次存取時，透過 Web 介面建立管理員帳號。然後安裝 Immich 行動應用程式（iOS/Android），設定連線到您的伺服器 URL。

### 資料持久化

| PVC 名稱 | 掛載路徑 | 大小 | 用途 |
|----------|----------|------|------|
| `immich-postgres-data` | `/var/lib/postgresql/data` | 10Gi | PostgreSQL 資料庫（中繼資料、向量、人臉資料） |
| `immich-upload-data` | `/usr/src/app/upload` | 50Gi | 原始照片與影片檔案 |
| `immich-model-cache` | `/cache` | 10Gi | 已下載的 ML 模型（CLIP、人臉辨識） |

所有 PVC 使用 `local-path` 儲存類別（k3s 預設）。

### 備份與還原

#### 備份

```bash
# 1. 備份 PostgreSQL 資料庫（中繼資料、人臉資料、向量嵌入）
kubectl -n immich exec sts/immich-postgres -- pg_dump -U immich immich > immich-db-backup.sql

# 2. 備份上傳的媒體檔案
kubectl -n immich exec deploy/immich-server -- tar czf /tmp/upload-backup.tar.gz /usr/src/app/upload
kubectl -n immich cp immich/<server-pod>:/tmp/upload-backup.tar.gz ./upload-backup.tar.gz
```

#### 還原

```bash
# 1. 還原 PostgreSQL 資料庫
kubectl -n immich exec -i sts/immich-postgres -- psql -U immich immich < immich-db-backup.sql

# 2. 還原上傳的媒體
kubectl -n immich cp ./upload-backup.tar.gz immich/<server-pod>:/tmp/upload-backup.tar.gz
kubectl -n immich exec deploy/immich-server -- tar xzf /tmp/upload-backup.tar.gz -C /

# 3. 重啟以重新索引
kubectl -n immich rollout restart deploy/immich-server
```

### 實用指令

```bash
# 查看所有 Pod 狀態
kubectl -n immich get pods

# 查看伺服器日誌
kubectl -n immich logs deploy/immich-server -f

# 查看 ML 處理日誌
kubectl -n immich logs deploy/immich-machine-learning -f

# 查看 PostgreSQL 日誌
kubectl -n immich logs sts/immich-postgres -f

# 檢查 Redis 連線
kubectl -n immich exec deploy/immich-redis -- redis-cli ping

# 檢查上傳儲存使用量
kubectl -n immich exec deploy/immich-server -- df -h /usr/src/app/upload

# 檢查 ML Pod 資源使用量
kubectl -n immich top pod -l component=machine-learning

# 重啟伺服器
kubectl -n immich rollout restart deploy/immich-server
```

### 疑難排解

#### 機器學習 Pod OOMKilled

ML 容器會下載並載入需要大量記憶體的 AI 模型。請增加 `machine-learning-deployment.yaml` 中的記憶體限制：

```yaml
resources:
  limits:
    memory: 6Gi  # 從 4Gi 增加
```

```bash
kubectl apply -k k8s-manifests/immich/
```

#### PostgreSQL 無法啟動（pgvecto-rs 問題）

PostgreSQL 容器使用 pgvecto-rs 擴充功能的自訂啟動參數。請檢查 Pod 日誌：

```bash
kubectl -n immich logs sts/immich-postgres
```

常見問題：
- 資料目錄損壞：刪除 PVC 並重新建立
- pgvecto-rs 版本不相容：檢查映像標籤與資料目錄版本是否匹配

#### 人臉辨識或 CLIP 搜尋緩慢

```bash
# 檢查 ML Pod 資源使用量
kubectl -n immich top pod -l component=machine-learning

# 查看 ML 處理日誌
kubectl -n immich logs deploy/immich-machine-learning -f
```

#### Redis 連線問題

```bash
kubectl -n immich get pods -l component=redis
kubectl -n immich exec deploy/immich-redis -- redis-cli ping
```

#### 上傳失敗或逾時

若大檔案上傳失敗，請檢查 Immich 伺服器日誌並確認 PVC 有足夠空間：

```bash
kubectl -n immich exec deploy/immich-server -- df -h /usr/src/app/upload
```

### 檔案結構

```
immich/
├── configmap.yaml                    # 環境變數（DB 主機、Redis 主機、時區）
├── kustomization.yaml                # Kustomize 資源列表
├── machine-learning-deployment.yaml  # ML 容器的 Deployment（CLIP、人臉偵測）
├── machine-learning-service.yaml     # ML 的 ClusterIP 服務 (3003)
├── namespace.yaml                    # 命名空間: immich
├── postgres-service.yaml             # PostgreSQL 的 ClusterIP 服務 (5432)
├── postgres-statefulset.yaml         # PostgreSQL + pgvecto-rs 的 StatefulSet
├── pvc.yaml                          # 持久卷宣告（DB、上傳、模型快取）
├── README.md                         # 本文件
├── redis-deployment.yaml             # Redis 任務佇列的 Deployment
├── redis-service.yaml                # Redis 的 ClusterIP 服務 (6379)
├── secret.yaml                       # 資料庫帳號密碼（部署前請變更！）
├── server-deployment.yaml            # Immich 伺服器的 Deployment
└── server-service.yaml               # NodePort 服務 (30283 -> 2283)
```
