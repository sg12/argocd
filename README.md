# ArgoCD Configuration Repository

Этот репозиторий содержит манифесты Kubernetes для развертывания приложений через ArgoCD.

## Структура репозитория

```
argocd/
├── apps/                           # ArgoCD Application манифесты
│   ├── django-app.yaml            # Application для Django приложения
│   ├── monitoring.yaml            # Application для monitoring stack
│   ├── osrm.yaml                  # Application для OSRM сервиса
│   └── my-django-app/             # Helm chart для Django приложения
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
├── monitoring/                     # Манифесты мониторинга
│   ├── prometheus.yaml
│   ├── prometheus-rbac.yaml
│   ├── grafana.yaml
│   └── loki.yaml
└── osrm/                          # Манифесты OSRM сервиса
    ├── osrm-monolith-deployment.yaml
    └── osrm-monolith-service.yaml
```

## Быстрый старт

### 1. Установка ArgoCD Applications

```bash
# Применить все ArgoCD Applications
kubectl apply -f apps/django-app.yaml
kubectl apply -f apps/monitoring.yaml
kubectl apply -f apps/osrm.yaml
```

### 2. Проверка статуса приложений

```bash
# Через ArgoCD CLI
argocd app list
argocd app get my-django-app
argocd app get monitoring-stack
argocd app get osrm-service

# Через kubectl
kubectl get applications -n argocd
```

### 3. Синхронизация приложений

```bash
# Ручная синхронизация (если автосинхронизация отключена)
argocd app sync my-django-app
argocd app sync monitoring-stack
argocd app sync osrm-service
```

## Конфигурация

### Django Application (Helm Chart)

Основные параметры настраиваются в `apps/my-django-app/values.yaml`:

- **djangoApp.image.repository**: Образ Django приложения
- **djangoApp.image.tag**: Тег образа
- **djangoApp.replicaCount**: Количество реплик
- **ingress.hostname**: Доменное имя для Ingress
- **postgres/rabbitmq/redis/elasticsearch.enabled**: Включение/отключение зависимостей

#### Зависимости Django приложения:
- **PostgreSQL**: База данных (порт 5432)
- **RabbitMQ**: Очередь сообщений (порты 5672, 15672)
- **Redis**: Кеширование и Celery backend (порт 6379)
- **Elasticsearch**: Поиск и индексация (порты 9200, 9300)

### Monitoring Stack

Включает:
- **Prometheus**: Сбор метрик (порт 9090)
- **Grafana**: Визуализация (NodePort 32000)
- **Loki**: Агрегация логов (порт 3100)

### OSRM Service

Сервис маршрутизации на базе OpenStreetMap:
- Образ: `gghotdog/osrm-generator:siberia-v1`
- Ресурсы: 4-6Gi памяти, 1-2 CPU
- Доступен внутри кластера на порту 80

## Важные замечания

### Перед развертыванием

1. **Обновите URL репозитория** во всех Application манифестах (`apps/*.yaml`):
   ```yaml
   source:
     repoURL: https://github.com/YOUR_USERNAME/argocd.git
   ```

2. **Создайте необходимые secrets** для Django приложения:
   ```bash
   kubectl create secret generic app-secrets \
     --from-literal=SECRET_KEY=your-secret-key \
     --from-literal=DATABASE_URL=postgres://user:password@postgres-service:5432/dbname
   ```

3. **Настройте namespace** для каждого приложения в соответствующих Application манифестах.

### Автосинхронизация

Все приложения настроены с автоматической синхронизацией:
- `prune: true` - удаление ресурсов, которых нет в Git
- `selfHeal: true` - автоматическое восстановление при ручных изменениях

Для отключения автосинхронизации удалите секцию `syncPolicy.automated` из манифестов.

## Полезные команды

```bash
# Просмотр логов ArgoCD Application Controller
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller

# Проверка статуса синхронизации
argocd app sync my-django-app --dry-run

# Откат к предыдущей версии
argocd app rollback my-django-app

# Удаление приложения из ArgoCD (ресурсы будут удалены)
argocd app delete my-django-app --cascade

# Удаление приложения из ArgoCD (ресурсы останутся)
argocd app delete my-django-app --cascade=false
```

## Разработка

При внесении изменений:

1. Измените манифесты в соответствующей папке
2. Закоммитьте и запушьте в Git
3. ArgoCD автоматически обнаружит изменения и применит их (если включена автосинхронизация)
4. Или выполните ручную синхронизацию: `argocd app sync <app-name>`
