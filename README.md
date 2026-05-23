# Лабораторная работа - Микросервисный мессенджер в Kubernetes (GitOps / ArgoCD)

Проект представляет собой развертывание микросервисного приложения (Messager) в Kubernetes с использованием современных практик: GitOps (ArgoCD), шаблонизации Kustomize (Dev/Prod окружения), правил NodeAffinity и подключения S3-хранилища через Yandex CSI.

## Требования к окружению
Для локального заупска:
* Docker Desktop с включенным Kubernetes (или Minikube/Kind)
* Установленный `kubectl`
* Установленный ArgoCD в кластере

## Инструкция по запуску

### 1. Подготовка узлов кластера
Для корректной работы баз данных и маршрутизации микросервисов необходимо разметить узлы кластера.
*Для одноузлового кластера (например, Docker Desktop) выполните команды для одного и того же узла:*
```bash
kubectl label node <имя_узла> workload=system --overwrite
kubectl label node <имя_узла> workload=app --overwrite
kubectl label node <имя_узла> disk=fast --overwrite
```

### 2. Установка ArgoCD и драйвера Yandex CSI
Установите ArgoCD и драйвер CSI для S3 (базовые компоненты):
```bash
# ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Патч для обхода блокировок DockerHub в РФ
kubectl patch deployment argocd-redis -n argocd --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value": "mirror.gcr.io/redis:7.2.4-alpine"}]'

# Yandex CSI Driver
kubectl create namespace csi-s3
kubectl apply -f https://raw.githubusercontent.com/yandex-cloud/k8s-csi-s3/master/deploy/kubernetes/provisioner.yaml
kubectl apply -f https://raw.githubusercontent.com/yandex-cloud/k8s-csi-s3/master/deploy/kubernetes/csi-s3.yaml
```

### 3. Развертывание приложения
Развертывание происходит автоматически через ArgoCD, который подтягивает манифесты из этого репозитория.
```bash
kubectl apply -f argocd/
```
Приложения `messager-dev` и `messager-prod` будут созданы, и ArgoCD автоматически начнет синхронизацию.

### 4. Проверка и доступ
Проверьте статус подов:
```bash
kubectl get pods -n messager-prod
```
Когда все поды перейдут в статус `Running`, пробросьте порты для UI и API:
```bash
# API (BFF)
kubectl port-forward pod/<имя_пода_bff> 8080:8080 -n messager-prod
# Frontend
kubectl port-forward pod/<имя_пода_frontend> 3000:80 -n messager-prod
```
Приложение будет доступно в браузере по адресу: `http://localhost:3000`.

Подробный отчет о выполнении работы и решении проблем см. в [docs/REPORT.md](docs/REPORT.md).
