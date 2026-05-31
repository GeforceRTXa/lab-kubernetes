# Messenger in Kubernetes — Runbook

## Структура репозитория

```
k8s/
├── base/                          ← общие манифесты для всех окружений
│   ├── kustomization.yaml
│   ├── namespace.yaml             ← namespace: messager
│   ├── configmap.yaml             ← несекретные переменные (порты, URL)
│   ├── secret.yaml                ← пароли БД, ключи MinIO/S3
│   ├── postgres-pvc.yaml          ← хранилище для postgres
│   ├── postgres.yaml              ← БД + init-скрипт создания messager_users/messager_messages
│   ├── minio.yaml                 ← локальный S3 (affinity: workload=system)
│   ├── message-uploads-pv-pvc.yaml ← PV/PVC через CSI S3-driver
│   ├── migrate-users-job.yaml     ← goose миграции для users DB
│   ├── migrate-messages-job.yaml  ← goose миграции для messages DB
│   ├── user-service.yaml          ← порт 8081, affinity: workload=app
│   ├── message-service.yaml       ← порт 8082, affinity: workload=app (hard) + disk=fast (soft)
│   ├── bff.yaml                   ← порт 8080, affinity: workload=app
│   ├── frontend.yaml              ← порт 80, affinity: workload=app
│   └── ingress.yaml               ← nginx ingress
├── overlays/
│   ├── dev/
│   │   ├── kustomization.yaml     ← 1 реплика, dev.messager.local, latest теги
│   │   └── patches/
│   │       └── frontend-resources.yaml
│   └── prod/
│       ├── kustomization.yaml     ← 2 реплики, prod хост, теги v1.0.0
│       └── patches/
│           └── resources-prod.yaml
argocd/
├── application-dev.yaml           ← Argo CD app → k8s/overlays/dev
└── application-prod.yaml          ← Argo CD app → k8s/overlays/prod
docs/
└── README.md                      ← этот файл
```

---

## Различия dev vs prod

| Параметр         | dev                    | prod                      |
|------------------|------------------------|---------------------------|
| Реплики          | 1                      | 2 для всех сервисов       |
| Теги образов     | latest                 | 1.0.0 (фиксированный)     |
| Ingress host     | dev.messager.local     | prod.messager.example.com |
| CPU limit bff    | 300m                   | 500m                      |
| Memory limit bff | 384Mi                  | 512Mi                     |
| Ресурсы frontend | 50m/64Mi → 100m/128Mi  | 150m/192Mi → 300m/384Mi   |

---

## Порты сервисов

| Сервис          | Порт |
|-----------------|------|
| frontend        | 80   |
| bff             | 8080 |
| user-service    | 8081 |
| message-service | 8082 |
| postgres        | 5432 |
| minio (S3 API)  | 9000 |
| minio (console) | 9001 |

---

## nodeAffinity — где живут поды

| Сервис          | Тип ноды       | Правило               |
|-----------------|----------------|-----------------------|
| postgres        | workload=system| hard (required)       |
| minio           | workload=system| hard (required)       |
| frontend        | workload=app   | hard (required)       |
| bff             | workload=app   | hard (required)       |
| user-service    | workload=app   | hard (required)       |
| message-service | workload=app   | hard (required)       |
| message-service | disk=fast      | soft (preferred w=100)|

---

## Пошаговый запуск

### 1. Установить инструменты (в WSL Ubuntu)

```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# проверка
kubectl version --client
minikube version
```

### 2. Запустить кластер

```bash
minikube start --driver=docker --cpus=4 --memory=6g

# Пометить единственную ноду обоими лейблами (в minikube нода одна)
kubectl label node minikube workload=app
kubectl label node minikube workload=system
kubectl label node minikube disk=fast

# Включить ingress
minikube addons enable ingress
```

### 3. Установить CSI S3 драйвер

```bash
kubectl apply -k "github.com/ctrox/csi-s3/deploy/kubernetes?ref=master"
# Подождать пока поднимется
kubectl wait --for=condition=ready pod -l app=csi-s3 -n kube-system --timeout=120s
```

### 4. Создать bucket в MinIO

MinIO нужно сначала задеплоить, потом создать bucket:

```bash
# Применить только minio сначала
kubectl apply -f k8s/base/namespace.yaml
kubectl apply -f k8s/base/secret.yaml
kubectl apply -f k8s/base/minio.yaml

# Пробросить порт и создать bucket через mc (minio client)
kubectl port-forward -n messager svc/minio 9000:9000 &

# Установить mc
curl -LO https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc && sudo mv mc /usr/local/bin/

mc alias set local http://localhost:9000 minioadmin minioadmin123
mc mb local/uploads
```

### 5. Запушить в GitHub и задеплоить через Argo CD

```bash
# В argocd/application-dev.yaml замени:
# YOUR_GITHUB_USERNAME — твой логин
# YOUR_REPO_NAME — имя репозитория

# Установить Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Подождать
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s

# Получить пароль
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo

# Открыть UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Браузер: https://localhost:8080  login: admin

# Применить Application
kubectl apply -f argocd/application-dev.yaml
```

### 6. Добавить хост в /etc/hosts

```bash
# Узнать IP minikube
minikube ip  # например 192.168.49.2

# Добавить хосты
echo "192.168.49.2 dev.messager.local" | sudo tee -a /etc/hosts
```

Открыть в браузере: http://dev.messager.local

---

## Проверка перед сдачей

```bash
# 1. Проверить сборку overlays
kubectl kustomize k8s/overlays/dev
kubectl kustomize k8s/overlays/prod

# 2. Статус подов
kubectl get pods -n messager

# 3. nodeAffinity реально работает
kubectl get pods -o wide -n messager
kubectl describe pod <имя-пода-message-service> -n messager | grep -A 10 Affinity

# 4. Проверить монтирование S3
kubectl exec -n messager deployment/message-service -- df -h /app/uploads

# 5. Статус Argo CD
kubectl get application -n argocd
```

Ожидаемые статусы подов:
- postgres, minio, frontend, bff, user-service, message-service → `Running`
- migrate-users, migrate-messages → `Completed`
- Argo CD → `Synced / Healthy`

---

## Troubleshooting

```bash
# Pod не стартует — смотрим события
kubectl describe pod -n messager <pod-name>

# Логи
kubectl logs -n messager deployment/message-service

# nodeAffinity не работает — проверить лейблы
kubectl get node minikube --show-labels

# Argo CD не синхронизирует — ручная синхронизация
kubectl -n argocd exec deployment/argocd-server -- argocd app sync messager-dev --insecure
```
