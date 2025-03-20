# Развертывание приложений с Helm

Развертывание приложений - основная функция Helm. В этом разделе мы рассмотрим все аспекты установки, обновления, отката и удаления приложений с помощью Helm.

## Жизненный цикл приложения в Helm

Жизненный цикл приложения, управляемого Helm, включает следующие этапы:

1. **Установка (Install)**: Первоначальное развертывание приложения
2. **Обновление (Upgrade)**: Изменение конфигурации или версии приложения
3. **Откат (Rollback)**: Возврат к предыдущей версии при проблемах
4. **Удаление (Uninstall)**: Удаление приложения из кластера

Каждая установка или обновление чарта создает новый **релиз**. Helm сохраняет историю релизов, что позволяет отслеживать изменения и выполнять откат при необходимости.

## Установка приложения

### Базовая установка

Самый простой способ установить приложение из чарта:

```bash
helm install myrelease bitnami/nginx
```

Где:
- `myrelease` - имя релиза (уникальный идентификатор установки)
- `bitnami/nginx` - имя чарта из репозитория

### Установка с пользовательскими значениями

Используйте файл values для настройки чарта:

```bash
helm install -f myvalues.yaml myrelease bitnami/nginx
```

Или используйте параметр `--set` для переопределения отдельных значений:

```bash
helm install --set replicaCount=3 --set service.type=LoadBalancer myrelease bitnami/nginx
```

### Установка чарта из локальной директории

```bash
helm install myrelease ./mychart
```

### Установка чарта из архива

```bash
helm install myrelease ./mychart-0.1.0.tgz
```

### Установка из URL

```bash
helm install myrelease https://example.com/charts/mychart-0.1.0.tgz
```

### Установка из OCI-репозитория

```bash
helm install myrelease oci://registry.example.com/charts/mychart --version 1.0.0
```

### Дополнительные параметры установки

| Параметр | Описание |
|----------|----------|
| `--create-namespace` | Создает namespace, если он не существует |
| `--namespace` или `-n` | Указывает namespace для установки |
| `--wait` | Ожидает, пока все ресурсы не будут готовы |
| `--timeout` | Устанавливает таймаут ожидания (например, `--timeout 5m`) |
| `--debug` | Включает отладочные сообщения |
| `--dry-run` | Симуляция установки без реального применения ресурсов |
| `--atomic` | Отменяет установку при ошибке |
| `--description` | Добавляет описание для релиза |

### Пример полной команды установки

```bash
helm install myapp ./myapp-chart \
  --namespace my-namespace \
  --create-namespace \
  --set global.environment=production \
  --set replicaCount=3 \
  -f production-values.yaml \
  --wait \
  --timeout 10m \
  --atomic \
  --description "Production deployment of MyApp v1.2.3"
```

## Просмотр установленных релизов

### Список всех релизов

```bash
helm list
```

или сокращенно:

```bash
helm ls
```

### Список релизов в определенном namespace

```bash
helm list -n my-namespace
```

### Список всех релизов во всех namespace

```bash
helm list --all-namespaces
```

или сокращенно:

```bash
helm ls -A
```

### Фильтрация списка релизов

```bash
# По статусу
helm list --deployed    # Только успешно установленные
helm list --failed      # Только с ошибками
helm list --pending     # Ожидающие завершения
helm list --uninstalled # Удаленные, но с сохраненной историей

# По имени (с поддержкой регулярных выражений)
helm list --filter 'nginx-.*'
```

## Получение информации о релизе

### Просмотр состояния релиза

```bash
helm status myrelease
```

### Получение манифестов релиза

```bash
helm get manifest myrelease
```

### Получение значений релиза

```bash
helm get values myrelease       # Только переопределенные значения
helm get values myrelease -a    # Все значения, включая значения по умолчанию
```

### Получение заметок релиза

```bash
helm get notes myrelease
```

### Получение всей информации о релизе

```bash
helm get all myrelease
```

## Обновление приложения

### Обновление с новыми значениями

```bash
helm upgrade myrelease bitnami/nginx -f new-values.yaml
```

### Обновление с дополнительными параметрами

```bash
helm upgrade myrelease bitnami/nginx --set replicaCount=5 --set image.tag=1.22.0
```

### Установка или обновление (если релиз уже существует)

```bash
helm upgrade --install myrelease bitnami/nginx
```

Часто используется сокращение `helm upgrade -i` или в скриптах CI/CD:

```bash
helm upgrade -i myrelease bitnami/nginx -f values.yaml
```

### Обновление с сохранением конкретной истории

По умолчанию Helm хранит 10 последних ревизий. Вы можете изменить это поведение:

```bash
helm upgrade --history-max 15 myrelease bitnami/nginx
```

### Важные флаги для обновления

| Параметр | Описание |
|----------|----------|
| `--install` или `-i` | Устанавливает чарт, если релиз не существует |
| `--force` | Принудительное обновление с перезапуском ресурсов |
| `--wait` | Ожидает готовности ресурсов после обновления |
| `--atomic` | Откатывает изменения при ошибке обновления |
| `--cleanup-on-fail` | Удаляет новые ресурсы при ошибке обновления |
| `--reuse-values` | Повторно использует предыдущие значения с применением новых |

### Просмотр различий при обновлении

Для просмотра изменений перед применением используйте плагин `helm-diff`:

```bash
# Установка плагина
helm plugin install https://github.com/databus23/helm-diff

# Просмотр изменений
helm diff upgrade myrelease bitnami/nginx -f new-values.yaml
```

## Откат релиза

### Просмотр истории релиза

```bash
helm history myrelease
```

### Откат к предыдущей ревизии

```bash
helm rollback myrelease 1    # Откат к ревизии 1
```

### Откат с ожиданием завершения

```bash
helm rollback myrelease 2 --wait --timeout 5m
```

### Откат с дополнительными параметрами

```bash
helm rollback myrelease 3 --force    # Принудительный откат
```

## Удаление приложения

### Базовое удаление

```bash
helm uninstall myrelease
```

или старый синтаксис (до Helm 3):

```bash
helm delete myrelease
```

### Удаление нескольких релизов

```bash
helm uninstall release1 release2 release3
```

### Удаление с сохранением истории

```bash
helm uninstall myrelease --keep-history
```

Это позволит восстановить релиз в будущем:

```bash
helm rollback myrelease 1
```

### Удаление с ожиданием

```bash
helm uninstall myrelease --wait
```

## Стратегии развертывания с Helm

### Blue-Green Deployment

Blue-Green развертывание предполагает одновременное существование двух версий приложения, что позволяет быстро переключаться между ними.

```bash
# Установка "синей" версии
helm install myapp-blue ./myapp -f blue-values.yaml --set ingress.host=myapp-blue.example.com

# Установка "зеленой" версии (новой)
helm install myapp-green ./myapp -f green-values.yaml --set ingress.host=myapp.example.com

# Переключение трафика путем обновления ingress
helm upgrade myapp-blue ./myapp -f blue-values.yaml --set ingress.host=myapp.example.com
helm upgrade myapp-green ./myapp -f green-values.yaml --set ingress.host=myapp-green.example.com

# Удаление старой версии после успешного переключения
helm uninstall myapp-blue
```

### Canary Deployment

Canary-развертывание направляет часть трафика на новую версию приложения для тестирования в production-среде.

```bash
# Установка основной версии
helm install myapp-stable ./myapp -f stable-values.yaml --set replicaCount=9

# Установка canary-версии
helm install myapp-canary ./myapp -f canary-values.yaml --set replicaCount=1

# При успешном тестировании - обновление основной версии и удаление canary
helm upgrade myapp-stable ./myapp -f canary-values.yaml --set replicaCount=10
helm uninstall myapp-canary
```

### Rolling Update

Стандартное плавное обновление Kubernetes, контролируемое параметрами `strategy` в Deployment:

```yaml
# values.yaml
deployment:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

```bash
helm upgrade myrelease ./mychart -f values.yaml
```

## Проверки и тестирование при деплое

### Проверка чарта перед установкой

```bash
# Проверка синтаксиса и структуры чарта
helm lint ./mychart

# Рендеринг шаблонов без установки
helm template ./mychart

# Симуляция установки
helm install --dry-run --debug myrelease ./mychart
```

### Встроенные тесты Helm

Helm поддерживает автоматические тесты через специальные Pod-ресурсы:

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "mychart.fullname" . }}-test-connection"
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args: ['{{ include "mychart.fullname" . }}:{{ .Values.service.port }}']
  restartPolicy: Never
```

Запуск тестов:

```bash
helm test myrelease
```

## Использование хуков Helm

Хуки позволяют выполнять дополнительные операции в определенные моменты жизненного цикла релиза.

### Типы хуков

| Хук | Описание |
|-----|----------|
| `pre-install` | Выполняется перед установкой релиза |
| `post-install` | Выполняется после установки релиза |
| `pre-upgrade` | Выполняется перед обновлением релиза |
| `post-upgrade` | Выполняется после обновления релиза |
| `pre-delete` | Выполняется перед удалением релиза |
| `post-delete` | Выполняется после удаления релиза |
| `pre-rollback` | Выполняется перед откатом релиза |
| `post-rollback` | Выполняется после отката релиза |
| `test` | Выполняется при запуске `helm test` |

### Пример хука для миграции базы данных

```yaml
# templates/hooks/db-migrate.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-db-migrate
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "1"        # Порядок выполнения
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded   # Политика удаления
spec:
  template:
    spec:
      containers:
        - name: db-migrate
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
          command: ["./migrate.sh"]
      restartPolicy: Never
```

## Отладка проблем развертывания

### Проверка ресурсов Kubernetes

```bash
# Проверка подов
kubectl get pods -n <namespace>

# Просмотр логов
kubectl logs -n <namespace> <pod-name>

# Проверка событий
kubectl get events -n <namespace>

# Описание ресурса
kubectl describe pod -n <namespace> <pod-name>
```

### Диагностические команды Helm

```bash
# Проверка статуса релиза
helm status myrelease

# История релиза
helm history myrelease

# Просмотр манифестов
helm get manifest myrelease

# Просмотр примененных значений
helm get values myrelease -a
```

### Отладка с --debug

```bash
helm install --debug --dry-run myrelease ./mychart
```

## Настройка производительности и ресурсов

### Установка лимитов ресурсов

```yaml
# values.yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

### Настройка горизонтального автомасштабирования

```yaml
# values.yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 80
```

### Настройка Affinity и Node Selector

```yaml
# values.yaml
nodeSelector:
  kubernetes.io/role: worker
  
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app.kubernetes.io/name: {{ include "mychart.name" . }}
        topologyKey: "kubernetes.io/hostname"
```

## Шпаргалка по деплойменту

| Операция | Команда |
|----------|---------|
| Установка чарта | `helm install myrelease bitnami/nginx` |
| Установка с values | `helm install -f values.yaml myrelease bitnami/nginx` |
| Обновление релиза | `helm upgrade myrelease bitnami/nginx` |
| Установка или обновление | `helm upgrade -i myrelease bitnami/nginx` |
| Откат релиза | `helm rollback myrelease 1` |
| Удаление релиза | `helm uninstall myrelease` |
| Список релизов | `helm list` |
| Статус релиза | `helm status myrelease` |
| История релиза | `helm history myrelease` |
| Тестирование релиза | `helm test myrelease` |
| Проверка без установки | `helm install --dry-run --debug myrelease ./mychart` |

## Заключение

Эффективное управление развертыванием приложений с помощью Helm требует понимания основных концепций и инструментов. Использование правильных стратегий развертывания, тщательное тестирование и мониторинг помогут обеспечить надежность и стабильность ваших приложений в Kubernetes. 