# Основные концепции Helm

В этом документе рассматриваются фундаментальные концепции Helm, которые необходимо понять перед началом работы с инструментом.

## Содержание
- [Чарты (Charts)](#чарты-charts)
- [Релизы (Releases)](#релизы-releases)
- [Репозитории (Repositories)](#репозитории-repositories)
- [Шаблоны (Templates)](#шаблоны-templates)
- [Values](#values)
- [Hooks](#hooks)
- [Зависимости (Dependencies)](#зависимости-dependencies)
- [Версионирование](#версионирование)

## Чарты (Charts)

Чарт (Chart) — основная единица в экосистеме Helm. Это пакет, содержащий все необходимые ресурсы для развертывания приложения в Kubernetes.

### Анатомия чарта

```
mychart/
├── .helmignore    # Файлы, которые нужно игнорировать при упаковке чарта
├── Chart.yaml     # Метаданные чарта (версия, описание и т.д.)
├── values.yaml    # Значения по умолчанию для шаблонов
├── charts/        # Директория для зависимых чартов (подчартов)
├── templates/     # Директория с шаблонами
│   ├── NOTES.txt  # Информация, отображаемая после установки
│   ├── _helpers.tpl # Вспомогательные шаблоны
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ...
└── crds/         # Директория для Custom Resource Definitions (опционально)
```

### Типы чартов

1. **Приложение**: Чарт, который развертывает конкретное приложение (например, nginx, postgresql)
2. **Библиотека**: Чарт, содержащий общие шаблоны и функции для использования другими чартами
3. **Umbrella chart**: Чарт, объединяющий несколько приложений в одном пакете

### Жизненный цикл чарта

```
       ┌───────────┐
       │ Создание  │
       │   чарта   │
       └─────┬─────┘
             │
             ▼
       ┌───────────┐
       │ Разработка│
       │   и тест  │
       └─────┬─────┘
             │
             ▼
       ┌───────────┐
       │ Упаковка  │
       │   чарта   │
       └─────┬─────┘
             │
             ▼
       ┌───────────┐     ┌───────────┐
       │ Публикация│     │ Установка │
       │   чарта   ├────▶│  релиза   │
       └───────────┘     └───────────┘
```

## Релизы (Releases)

Релиз — это установленный экземпляр чарта в кластере Kubernetes. Когда чарт устанавливается, Helm создает релиз и присваивает ему уникальное имя.

### Состояния релизов

- **Deployed**: Релиз успешно установлен в кластере
- **Failed**: Установка или обновление релиза завершилось с ошибкой
- **Uninstalled**: Релиз удален из кластера, но история сохранена
- **Superseded**: Релиз заменен новой версией

### Управление версиями релизов

Helm сохраняет историю всех установок, обновлений и откатов релиза, что позволяет:

1. Просматривать историю изменений
2. Откатывать релиз к предыдущей версии
3. Отслеживать, кто и когда вносил изменения

```
Версия 1 ──▶ Версия 2 ──▶ Версия 3
    ▲            │            │
    │            │            │
    └────────────┴────────────┘
        Возможность отката
```

### Хранение информации о релизах

В Helm v3 информация о релизах хранится в Secret-ресурсах Kubernetes в соответствующем namespace. Эта информация включает:

- Применяемые манифесты
- Values, использованные при установке
- Метаданные релиза

## Репозитории (Repositories)

Репозиторий Helm — это хранилище, содержащее упакованные чарты, которые можно установить. Репозитории могут быть:

1. **Публичными**: Доступные всем (например, Bitnami, Stable)
2. **Частными**: Внутри организации, с ограниченным доступом

### Структура репозитория

Репозиторий Helm состоит из:

- **index.yaml**: Файл индекса, содержащий информацию о доступных чартах
- **Архивы чартов**: Файлы `.tgz`, содержащие упакованные чарты

### Популярные публичные репозитории

- **[Artifact Hub](https://artifacthub.io/)**: Центральный репозиторий Helm чартов
- **[Bitnami](https://charts.bitnami.com/bitnami)**: Коллекция чартов для популярных приложений
- **[VMware Tanzu](https://github.com/vmware-tanzu/helm-charts)**: Чарты от VMware Tanzu
- **[Elastic](https://helm.elastic.co/)**: Чарты для продуктов Elastic stack

### Работа с репозиториями

```bash
# Добавление репозитория
helm repo add bitnami https://charts.bitnami.com/bitnami

# Обновление информации о репозиториях
helm repo update

# Поиск чартов в репозиториях
helm search repo nginx

# Поиск всех версий чарта
helm search repo nginx --versions
```

## Шаблоны (Templates)

Шаблоны в Helm — это файлы, содержащие Kubernetes-манифесты с динамическими значениями, которые заменяются при установке чарта. Helm использует шаблонизатор на основе Go templates.

### Основы шаблонизации

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
    release: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
      release: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
        release: {{ .Release.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
```

### Встроенные объекты в шаблонах

- **`.Release`**: Информация о релизе (имя, namespace и т.д.)
- **`.Chart`**: Содержимое файла `Chart.yaml`
- **`.Values`**: Значения из `values.yaml` и переопределенные при установке
- **`.Template`**: Информация о текущем шаблоне
- **`.Capabilities`**: Информация о возможностях кластера Kubernetes

## Values

Values — это значения, которые используются для настройки чарта при установке. Они могут быть определены:

1. В файле `values.yaml` чарта (значения по умолчанию)
2. В файлах `values.yaml` родительского чарта (для подчартов)
3. Через флаг `-f` или `--values` при установке
4. Через флаг `--set` при установке (для отдельных значений)

### Пример файла values.yaml

```yaml
# Default values for mychart.
# This is a YAML-formatted file.

replicaCount: 1

image:
  repository: nginx
  tag: "1.21.6"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 80
```

### Объединение values из разных источников

При установке или обновлении чарта Helm объединяет values в следующем порядке (значения из последующих источников переопределяют предыдущие):

1. Values из `values.yaml` чарта
2. Values из родительского чарта (для подчартов)
3. Values из файла, указанного через `--values`
4. Values, указанные через `--set`

## Hooks

Хуки (Hooks) — это способ вмешаться в жизненный цикл релиза для выполнения дополнительных действий. Хуки объявляются как обычные шаблоны Kubernetes с аннотациями.

### Типы хуков

- **pre-install**: Выполняется перед установкой
- **post-install**: Выполняется после установки
- **pre-delete**: Выполняется перед удалением
- **post-delete**: Выполняется после удаления
- **pre-upgrade**: Выполняется перед обновлением
- **post-upgrade**: Выполняется после обновления
- **pre-rollback**: Выполняется перед откатом
- **post-rollback**: Выполняется после отката
- **test**: Выполняется при запуске тестов

### Пример хука

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-db-init
  annotations:
    "helm.sh/hook": post-install
    "helm.sh/hook-weight": "10"
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  template:
    spec:
      containers:
      - name: db-init
        image: postgres:latest
        command: ["psql", "-c", "CREATE DATABASE app;"]
      restartPolicy: Never
```

## Зависимости (Dependencies)

Зависимости в Helm позволяют использовать один чарт внутри другого. Это помогает создавать сложные приложения из нескольких компонентов.

### Объявление зависимостей

Зависимости объявляются в файле `Chart.yaml`:

```yaml
# Chart.yaml
apiVersion: v2
name: my-app
version: 1.0.0
dependencies:
  - name: postgresql
    version: 12.1.3
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
  - name: redis
    version: 17.3.14
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

### Управление зависимостями

```bash
# Загрузка зависимостей
helm dependency update ./my-app

# Список зависимостей
helm dependency list ./my-app
```

### Доступ к values зависимостей

Values для зависимых чартов задаются в родительском `values.yaml` в переменной с именем зависимости:

```yaml
# values.yaml родительского чарта
postgresql:
  enabled: true
  auth:
    username: myuser
    password: mypassword
    database: mydb
  
redis:
  enabled: true
  auth:
    password: myredispassword
```

## Версионирование

Helm использует семантическое версионирование (SemVer) для чартов и версий зависимостей:

```
MAJOR.MINOR.PATCH
```

- **MAJOR**: Несовместимые изменения API
- **MINOR**: Новая функциональность с обратной совместимостью
- **PATCH**: Исправления багов с обратной совместимостью

### Контроль версий в Chart.yaml

```yaml
apiVersion: v2
name: myapp
version: 1.2.3  # Версия самого чарта
appVersion: "4.0.0"  # Версия приложения, которое устанавливает чарт
```

### Условия в зависимостях

Helm позволяет устанавливать чарты с определенными версиями зависимостей:

```yaml
dependencies:
  - name: postgresql
    version: ">= 12.0.0, < 13.0.0"
    repository: https://charts.bitnami.com/bitnami
```

## Практический пример

Рассмотрим пример связи между основными концепциями Helm:

1. Разработчик создает чарт для приложения.
2. Чарт загружается в репозиторий.
3. DevOps-инженер устанавливает чарт в кластер, настраивая его через values.
4. Helm создает релиз, применяя шаблоны с заданными значениями values.
5. При необходимости, релиз обновляется с новыми values или новой версией чарта.
6. Если что-то идет не так, релиз можно откатить к предыдущей версии.

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│             │     │             │     │             │     │             │
│    Чарт     │ ──▶ │ Репозиторий │ ──▶ │   Values    │ ──▶ │   Релиз     │
│  (Chart)    │     │(Repository) │     │             │     │ (Release)   │
│             │     │             │     │             │     │             │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
``` 