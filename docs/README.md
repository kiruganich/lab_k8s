# Документация и пояснения к лабораторной работе

## 1. Обход блокировок Docker Hub
В манифестах для образов `postgres` и `minio` используются зеркала `mirror.gcr.io` (например, `mirror.gcr.io/library/postgres:16-alpine`), что позволяет скачивать эти популярные образы даже при наличии блокировок в РФ. Образы самого мессенджера (`mablinov2704/*`) тянутся как обычно, но если с ними возникнут проблемы, вы можете настроить Docker-демон minikube на использование зеркала:
`minikube start --registry-mirror=https://mirror.gcr.io`

## 2. Различия между dev и prod (Kustomize)
- **Количество реплик**: в dev используются единичные реплики для всех компонентов (frontend, bff, user-service, message-service), тогда как в prod масштабирование установлено на 2 реплики для обеспечения отказоустойчивости.
- **Ресурсы**: в dev установлены минимальные requests/limits для экономии ресурсов локального кластера. В prod лимиты и запросы (requests/limits) увеличены.
- **Изоляция окружений**: dev и prod разворачиваются в разные неймспейсы (messager-dev и messager-prod соответственно).
- В `prod` overlay продемонстрирована механика фиксации тегов через блок `images:` (оставлено `latest` для совместимости, но готово к использованию sha-хэшей).

## 3. Настройка Node Affinity
Свойства `nodeAffinity` добавлены в `base` манифесты:
- **Postgres и MinIO**: имеют жесткое требование (`requiredDuringScheduling...`) запускаться на узлах с меткой `workload=system`.
- **Frontend, BFF, User-Service**: имеют жесткое требование запускаться на `workload=app`.
- **Message-Service**: имеет жесткое требование к `workload=app` и дополнительное мягкое предпочтение (`preferredDuringScheduling...` с весом 100) к узлам с меткой `disk=fast`. 

## 4. Загрузка через S3 CSI
В `base/csi-s3.yaml` сконфигурирован `StorageClass` (провайдер `ru.yandex.s3.csi`), использующий GeeseFS.
Предварительно Job `minio-setup` создает бакет `uploads` в MinIO, после чего CSI монтирует его как локальную папку `/uploads` в `message-service`.

## Инструкция по запуску:
1. Запушьте данный каталог в свой GitHub репозиторий.
2. В файлах `argocd/application-dev.yaml` и `argocd/application-prod.yaml` замените `YOUR_USERNAME/YOUR_REPO_NAME` на актуальную ссылку.
3. Примените ArgoCD манифесты в кластере с установленным ArgoCD:
   `kubectl apply -f argocd/`
4. ArgoCD автоматически создаст неймспейсы, синхронизирует и развернет все компоненты.
