# Структура Helm чартов

В этом документе подробно рассматривается структура Helm чартов, описание каждого файла и директории, а также лучшие практики по организации чартов.

## Содержание
- [Базовая структура чарта](#базовая-структура-чарта)
- [Файл Chart.yaml](#файл-chartyaml)
- [Файл values.yaml](#файл-valuesyaml)
- [Директория templates](#директория-templates)
- [Директория charts](#директория-charts)
- [Файл .helmignore](#файл-helmignore)
- [Директория crds](#директория-crds)
- [Типы чартов](#типы-чартов)
- [Лучшие практики](#лучшие-практики)

## Базовая структура чарта

Когда вы создаете чарт командой `helm create mychart`, генерируется стандартная структура:

```
mychart/
├── .helmignore         # Файлы для игнорирования при упаковке
├── Chart.yaml          # Информация о чарте
├── values.yaml         # Значения по умолчанию
├── charts/             # Директория для зависимых чартов
├── templates/          # Директория с шаблонами
│   ├── NOTES.txt       # Документация для пользователя
│   ├── _helpers.tpl    # Шаблоны-помощники
│   ├── deployment.yaml # Развертывание
│   ├── service.yaml    # Сервис
│   ├── serviceaccount.yaml # Учетная запись службы
│   ├── hpa.yaml        # Horizontal Pod Autoscaler (опционально)
│   ├── ingress.yaml    # Ingress (опционально)
│   └── tests/          # Тесты
│       └── test-connection.yaml # Тест подключения
└── crds/               # Custom Resource Definitions (если требуются)
```

## Файл Chart.yaml

Файл `Chart.yaml` содержит метаданные о чарте, такие как имя, версия, описание и т.д.

### Пример Chart.yaml

```yaml
apiVersion: v2             # Версия API чарта (для Helm 3 это всегда v2)
name: mychart              # Имя чарта
description: A Helm chart for Kubernetes # Описание чарта
type: application          # Тип чарта (application или library)
version: 0.1.0             # Версия чарта (SemVer 2)
appVersion: "1.16.0"       # Версия приложения, которое упаковано в чарт

# Дополнительные метаданные (опционально)
icon: https://example.com/logo.png # URL иконки чарта
keywords:                          # Ключевые слова для поиска
  - web
  - nginx
  - http
  - proxy
maintainers:                       # Информация о разработчиках
  - name: John Doe
    email: john@example.com
    url: https://example.com/john
home: https://example.com/chart     # Домашняя страница проекта
sources:                           # Исходный код
  - https://github.com/example/mychart
deprecated: false                  # Является ли чарт устаревшим

# Зависимости (опционально)
dependencies:
  - name: postgresql
    version: 12.1.3
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
    tags:
      - database
```

### Обязательные поля

- **apiVersion**: Для Helm 3 всегда `v2`
- **name**: Имя чарта
- **version**: Версия чарта (должна следовать SemVer 2)

### Типы чартов

- **application** (по умолчанию): Чарт для установки приложения
- **library**: Чарт, содержащий шаблоны и функции, которые могут использоваться другими чартами

## Файл values.yaml

Файл `values.yaml` содержит значения по умолчанию для шаблонов чарта. Эти значения могут быть переопределены при установке.

### Пример values.yaml

```yaml
# Количество реплик для deployment
replicaCount: 1

# Настройки образа
image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: ""  # По умолчанию использует appVersion из Chart.yaml

# Настройки imagePullSecrets
imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

# Настройки serviceAccount
serviceAccount:
  # Создавать ли serviceAccount
  create: true
  # Аннотации для serviceAccount
  annotations: {}
  # Имя serviceAccount (если не указано, будет сгенерировано)
  name: ""

# Настройки безопасности Pod
podSecurityContext: {}
  # fsGroup: 2000

# Настройки безопасности контейнера
securityContext: {}
  # capabilities:
  #   drop:
  #   - ALL
  # readOnlyRootFilesystem: true
  # runAsNonRoot: true
  # runAsUser: 1000

# Настройки сервиса
service:
  type: ClusterIP
  port: 80

# Настройки Ingress
ingress:
  enabled: false
  className: ""
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local

# Настройки ресурсов
resources: {}
  # limits:
  #   cpu: 100m
  #   memory: 128Mi
  # requests:
  #   cpu: 100m
  #   memory: 128Mi

# Настройки автоскейлинга
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
  # targetMemoryUtilizationPercentage: 80

# Аннотации для Pod
podAnnotations: {}

# Аффинити правила для Pod
affinity: {}

# Настройки tolerations
tolerations: []

# Настройки nodeSelector
nodeSelector: {}

# Настройки зависимостей (если есть)
postgresql:
  enabled: false  # Включение/отключение зависимого чарта
```

### Лучшие практики для values.yaml

1. **Структурируйте значения логически**: Группируйте связанные значения в структуры.
2. **Добавляйте комментарии**: Каждое значение должно иметь описание.
3. **Используйте осмысленные значения по умолчанию**: Установите безопасные и общепринятые значения по умолчанию.
4. **Включайте примеры**: Закомментированные примеры помогают пользователям понять, как настроить сложные параметры.

## Директория templates

Директория `templates` содержит шаблоны Kubernetes-манифестов, которые будут заполнены значениями из `values.yaml` и другими данными.

### Основные файлы в templates

- **NOTES.txt**: Текст, который отображается пользователю после установки чарта.
- **_helpers.tpl**: Общие шаблоны и функции-помощники.
- **deployment.yaml**: Манифест Deployment для приложения.
- **service.yaml**: Манифест Service для доступа к приложению.
- **ingress.yaml**: Манифест Ingress для маршрутизации HTTP-трафика.
- **serviceaccount.yaml**: Учетная запись службы для приложения.
- **configmap.yaml**: Конфигурационные данные.
- **secret.yaml**: Секретные данные.
- **hpa.yaml**: Горизонтальное автомасштабирование.

### Файл NOTES.txt

Файл `NOTES.txt` содержит инструкции по использованию установленного приложения. Он отображается после успешной установки чарта.

```
Пример NOTES.txt:

1. Получите IP-адрес приложения следующей командой:
{{- if .Values.ingress.enabled }}
{{- range $host := .Values.ingress.hosts }}
  http{{ if $.Values.ingress.tls }}s{{ end }}://{{ $host.host }}{{ .path }}
{{- end }}
{{- else if contains "NodePort" .Values.service.type }}
  export NODE_PORT=$(kubectl get --namespace {{ .Release.Namespace }} -o jsonpath="{.spec.ports[0].nodePort}" services {{ include "mychart.fullname" . }})
  export NODE_IP=$(kubectl get nodes --namespace {{ .Release.Namespace }} -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT
{{- else if contains "LoadBalancer" .Values.service.type }}
  export SERVICE_IP=$(kubectl get svc --namespace {{ .Release.Namespace }} {{ include "mychart.fullname" . }} --template "{{"{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}"}}")
  echo http://$SERVICE_IP:{{ .Values.service.port }}
{{- else if contains "ClusterIP" .Values.service.type }}
  export POD_NAME=$(kubectl get pods --namespace {{ .Release.Namespace }} -l "app.kubernetes.io/name={{ include "mychart.name" . }},app.kubernetes.io/instance={{ .Release.Name }}" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace {{ .Release.Namespace }} $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Посетите http://127.0.0.1:8080 для использования вашего приложения"
  kubectl --namespace {{ .Release.Namespace }} port-forward $POD_NAME 8080:$CONTAINER_PORT
{{- end }}
```

### Файл _helpers.tpl

Файл `_helpers.tpl` содержит общие шаблоны-помощники, которые можно использовать в других шаблонах. Имена функций обычно начинаются с имени чарта.

```
{{/*
Expand the name of the chart.
*/}}
{{- define "mychart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "mychart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "mychart.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "mychart.labels" -}}
helm.sh/chart: {{ include "mychart.chart" . }}
{{ include "mychart.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "mychart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use
*/}}
{{- define "mychart.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "mychart.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

### Директория tests

Директория `templates/tests` содержит тесты для чарта. Тесты выполняются командой `helm test` после установки чарта.

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

## Директория charts

Директория `charts` содержит зависимые чарты (подчарты). Когда вы используете команду `helm dependency update`, Helm загружает зависимые чарты в эту директорию.

```
charts/
├── postgresql-12.1.3.tgz
└── redis-17.3.14.tgz
```

## Файл .helmignore

Файл `.helmignore` указывает, какие файлы и директории должны быть исключены при упаковке чарта. Синтаксис аналогичен `.gitignore`.

```
# Шаблоны в этом файле определяют, какие файлы будут игнорироваться при упаковке
# Синтаксис соответствует .gitignore
# Комментарии начинаются с символа '#'

# Распространенные VCS директории
.git/
.gitignore
.bzr/
.bzrignore
.hg/
.hgignore
.svn/

# Распространенные временные файлы
*.tmp
*.bak
*.swp
*.swo
*~

# Различные IDE и редакторы
.project
.idea/
*.tmproj
.vscode/

# Системные файлы
.DS_Store
```

## Директория crds

Директория `crds` содержит Custom Resource Definitions (CRD), которые должны быть установлены до применения шаблонов. Важно отметить, что CRD устанавливаются без шаблонизации и не обновляются при обновлении чарта.

```yaml
# crds/crontab.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
```

## Типы чартов

### 1. Стандартный чарт (Application Chart)

Стандартный чарт предназначен для установки одного приложения. Это самый распространенный тип чарта.

### 2. Библиотечный чарт (Library Chart)

Библиотечный чарт содержит шаблоны и функции, которые могут использоваться другими чартами. Он не устанавливается в кластер.

В файле `Chart.yaml` нужно указать `type: library`:

```yaml
apiVersion: v2
name: mylib
type: library
version: 0.1.0
```

### 3. Umbrella чарт (Meta Chart)

Umbrella чарт объединяет несколько чартов в один. Он содержит минимальное количество шаблонов и в основном полагается на зависимости.

```yaml
# Chart.yaml
apiVersion: v2
name: myapp
version: 0.1.0
description: My application
dependencies:
  - name: frontend
    version: 0.1.0
    repository: "file://../frontend"
  - name: backend
    version: 0.1.0
    repository: "file://../backend"
  - name: database
    version: 12.1.3
    repository: https://charts.bitnami.com/bitnami
    condition: database.enabled
```

## Лучшие практики

### Организация структуры чарта

1. **Поддерживайте единый стиль именования**: Используйте последовательный стиль для имен файлов и шаблонов.
2. **Создавайте шаблоны-помощники**: Выносите повторяющийся код в `_helpers.tpl`.
3. **Разделяйте большие манифесты**: Разбивайте сложные манифесты на отдельные файлы.
4. **Используйте подчарты для компонентов**: Выделяйте логические компоненты в отдельные чарты.

### Шаблонизация и values

1. **Делайте значения настраиваемыми**: Все, что может отличаться в разных окружениях, должно быть в `values.yaml`.
2. **Используйте значения по умолчанию**: Устанавливайте разумные значения по умолчанию.
3. **Проверяйте значения**: Используйте условия для проверки обязательных значений.
4. **Документируйте**: Добавляйте комментарии к значениям в `values.yaml`.

### Версионирование и зависимости

1. **Следуйте SemVer**: Используйте семантическое версионирование для чартов.
2. **Фиксируйте версии зависимостей**: Указывайте конкретные версии зависимостей для предсказуемости.
3. **Используйте условия**: Используйте условия для включения/отключения компонентов.

### Безопасность

1. **Не хардкодьте секреты**: Используйте Kubernetes Secrets или внешние системы управления секретами.
2. **Настраивайте безопасность по умолчанию**: Включайте безопасные значения по умолчанию.
3. **Проверяйте разрешения**: Убедитесь, что приложение имеет минимально необходимые разрешения.

## Практические примеры

### Пример структуры чарта для веб-приложения с базой данных

```
webapp/
├── Chart.yaml               # Метаданные чарта
├── values.yaml              # Значения по умолчанию
├── values-production.yaml   # Значения для production
├── values-staging.yaml      # Значения для staging
├── charts/                  # Зависимые чарты
│   └── postgresql-12.1.3.tgz # Зависимый чарт PostgreSQL
├── templates/               # Шаблоны
│   ├── NOTES.txt            # Заметки для пользователя
│   ├── _helpers.tpl         # Шаблоны-помощники
│   ├── configmap.yaml       # Конфигурация приложения
│   ├── deployment.yaml      # Развертывание приложения
│   ├── hpa.yaml             # Горизонтальное автомасштабирование
│   ├── ingress.yaml         # Ingress для доступа извне
│   ├── secret.yaml          # Секреты приложения
│   ├── service.yaml         # Сервис для доступа к приложению
│   ├── serviceaccount.yaml  # Учетная запись службы
│   └── tests/               # Тесты
│       └── test-connection.yaml # Тест подключения
└── .helmignore              # Файлы для игнорирования
```

### Пример структуры Umbrella чарта для микросервисного приложения

```
myapp/
├── Chart.yaml               # Метаданные чарта с зависимостями
├── values.yaml              # Значения по умолчанию для всех компонентов
├── values-production.yaml   # Значения для production
├── charts/                  # Зависимые чарты
│   ├── frontend-0.1.0.tgz   # Чарт фронтенда
│   ├── backend-0.1.0.tgz    # Чарт бэкенда
│   ├── auth-0.1.0.tgz       # Чарт сервиса аутентификации
│   ├── postgresql-12.1.3.tgz # Чарт базы данных
│   └── redis-17.3.14.tgz    # Чарт Redis
├── templates/               # Общие шаблоны (минимальное количество)
│   ├── NOTES.txt            # Заметки для пользователя
│   ├── _helpers.tpl         # Шаблоны-помощники
│   └── tests/               # Тесты
│       └── test-connections.yaml # Тест подключения ко всем сервисам
└── .helmignore              # Файлы для игнорирования
``` 