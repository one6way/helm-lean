# Продвинутые техники и практики Helm

В этом разделе рассмотрим продвинутые подходы к работе с Helm, которые позволяют решать сложные задачи разработки и управления чартами.

## Кастомизация чартов без изменения исходного кода

### Использование overlay чартов

Overlay-чарты позволяют настраивать базовые чарты, не изменяя их напрямую:

```
my-overlay/
├── Chart.yaml           # Зависимость от базового чарта
├── values.yaml          # Переопределение значений базового чарта
└── templates/           # Дополнительные или переопределяющие шаблоны
    ├── configmap.yaml   # Добавление нового ресурса
    └── NOTES.txt        # Переопределение заметок
```

Пример `Chart.yaml`:

```yaml
apiVersion: v2
name: my-nginx-overlay
version: 0.1.0
description: An overlay chart for customizing nginx
dependencies:
  - name: nginx
    version: 13.1.5
    repository: https://charts.bitnami.com/bitnami
```

### Использование post-renderer

Post-renderer позволяет модифицировать манифесты после их создания Helm:

```bash
# Пример использования Kustomize как post-renderer
helm install myapp ./mychart --post-renderer ./kustomize-post-renderer.sh
```

Пример скрипта `kustomize-post-renderer.sh`:

```bash
#!/bin/bash
cat > kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
patchesJson6902:
- target:
    group: apps
    version: v1
    kind: Deployment
    name: myapp
  patch: |-
    - op: add
      path: /spec/template/metadata/annotations
      value: {"sidecar.istio.io/inject": "true"}
EOF

kubectl kustomize .
```

## Создание многосредовых конфигураций

### Использование нескольких файлов values для разных окружений

```
mychart/
├── values.yaml          # Значения по умолчанию
├── values-dev.yaml      # Значения для окружения разработки
├── values-staging.yaml  # Значения для тестового окружения
└── values-prod.yaml     # Значения для производственного окружения
```

Установка с множественными файлами values:

```bash
helm install myapp ./mychart -f values-prod.yaml -f values-prod-eu.yaml
```

### Использование Helmfile

[Helmfile](https://github.com/helmfile/helmfile) позволяет декларативно описывать развертывания Helm в различных окружениях:

```yaml
# helmfile.yaml
repositories:
  - name: bitnami
    url: https://charts.bitnami.com/bitnami

environments:
  default:
    values:
      - default-values.yaml
  production:
    values:
      - production-values.yaml
  staging:
    values:
      - staging-values.yaml

releases:
  - name: nginx
    namespace: frontend
    chart: bitnami/nginx
    version: 13.1.5
    values:
      - values/nginx.yaml
    set:
      - name: replicaCount
        value: 2
  
  - name: postgresql
    namespace: database
    chart: bitnami/postgresql
    version: 11.9.13
    values:
      - values/postgresql.yaml
    secrets:
      - values/secrets.yaml
```

Использование:

```bash
# Установка для окружения разработки
helmfile -e staging apply

# Синхронизация только одного релиза
helmfile -e production -l name=nginx apply
```

## Создание сложных library charts

Library charts не создают ресурсы Kubernetes напрямую, а предоставляют шаблоны и функции для других чартов.

### Создание library chart

```
common/
├── Chart.yaml
├── templates/
│   └── _helpers.tpl
└── values.yaml
```

Пример `Chart.yaml`:

```yaml
apiVersion: v2
name: common
version: 0.1.0
type: library
```

Пример `_helpers.tpl`:

```go
{{/*
Создает стандартные метки для ресурсов Kubernetes
*/}}
{{- define "common.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Создает сборщик ресурсов
*/}}
{{- define "common.resources" -}}
resources:
  {{- toYaml .Values.resources | nindent 2 }}
{{- end }}
```

### Использование library chart

В зависимом чарте:

```yaml
# Chart.yaml
dependencies:
  - name: common
    version: 0.1.0
    repository: file://../common
```

В шаблонах:

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  labels:
    {{- include "common.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          {{- include "common.resources" . | nindent 10 }}
```

## Создание своих плагинов Helm

Плагины позволяют расширять функциональность Helm.

### Структура плагина

```
$HELM_PLUGINS/
└── myplugin/
    ├── plugin.yaml      # Метаданные плагина
    └── myplugin.sh      # Исполняемый скрипт
```

Пример `plugin.yaml`:

```yaml
name: "myplugin"
version: "0.1.0"
usage: "Мой кастомный плагин для Helm"
description: "Демонстрация создания плагина для Helm"
command: "$HELM_PLUGIN_DIR/myplugin.sh"
hooks:
  install: "chmod +x $HELM_PLUGIN_DIR/myplugin.sh"
```

Пример `myplugin.sh`:

```bash
#!/bin/bash

# Получение информации о всех развернутых релизах
function list_all_releases() {
  for namespace in $(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'); do
    echo "Namespace: $namespace"
    helm list -n $namespace
    echo
  done
}

# Основная логика
case "${1:-}" in
  list-all)
    list_all_releases
    ;;
  *)
    echo "Usage: helm myplugin list-all"
    ;;
esac
```

### Установка и использование плагина

```bash
# Установка
helm plugin install /path/to/myplugin

# Использование
helm myplugin list-all
```

## Создание операторов на основе Helm

Операторы Kubernetes можно создавать с использованием Helm для управления установкой и обновлением ресурсов.

### Использование Operator SDK с Helm

Установка Operator SDK:

```bash
brew install operator-sdk
```

Создание оператора на основе Helm:

```bash
# Создание проекта
operator-sdk init --plugins=helm --domain example.com --helm-chart bitnami/nginx

# Сборка оператора
make docker-build docker-push IMG=example.com/nginx-operator:v0.1.0

# Установка оператора
make deploy IMG=example.com/nginx-operator:v0.1.0
```

Создание CR (Custom Resource):

```yaml
# nginx-sample.yaml
apiVersion: charts.example.com/v1alpha1
kind: Nginx
metadata:
  name: nginx-sample
spec:
  # Значения для чарта Nginx
  replicaCount: 2
  service:
    type: LoadBalancer
```

## Работа с OCI (Open Container Initiative)

Начиная с Helm 3.8.0, OCI (Open Container Initiative) стал официально поддерживаемым форматом для распространения Helm-чартов.

### Загрузка чартов в OCI-репозиторий

```bash
# Вход в реестр OCI
helm registry login registry.example.com

# Сохранение чарта как OCI-артефакт
helm chart save ./mychart registry.example.com/charts/mychart:1.0.0

# Загрузка чарта в реестр
helm chart push registry.example.com/charts/mychart:1.0.0
```

### Установка из OCI-репозитория

```bash
# Явная загрузка чарта
helm chart pull registry.example.com/charts/mychart:1.0.0

# Прямая установка из OCI-репозитория
helm install myapp oci://registry.example.com/charts/mychart --version 1.0.0
```

## Продвинутые приемы шаблонизации

### Использование кастомных функций Go templates

```go
{{/* Разделение строки на части и взятие определенного элемента */}}
{{ $parts := splitList "/" .Values.path }}
{{ $filename := last $parts }}

{{/* Форматирование строки */}}
{{ printf "myapp-%s" .Values.environment }}

{{/* Условные выражения */}}
{{ if and .Values.persistence.enabled (eq .Values.persistence.type "pvc") }}
  # Логика для PVC
{{ else if .Values.persistence.enabled }}
  # Логика для других типов хранилища
{{ else }}
  # Логика без создания хранилища
{{ end }}
```

### Использование трансформеров

```go
{{/* YAML в JSON и обратно */}}
{{ $yamlData := .Values.config | toYaml }}
{{ $jsonData := .Values.config | toJson }}

{{/* Преобразование из строки в map */}}
{{ $configMap := .Values.configString | fromYaml }}

{{/* Глубокое объединение структур */}}
{{ $merged := mergeOverwrite .Values.defaultConfig .Values.userConfig }}
```

### Создание и работа со словарями (maps)

```go
{{/* Создание нового словаря */}}
{{ $data := dict "key1" "value1" "key2" "value2" }}

{{/* Обход словаря */}}
{{- range $key, $value := $data }}
  {{ $key }}: {{ $value }}
{{- end }}

{{/* Добавление в словарь */}}
{{ $_ := set $data "key3" "value3" }}

{{/* Удаление из словаря */}}
{{ $_ := unset $data "key1" }}
```

### Использование переменных и подшаблонов со scope

```go
{{/* Определение переменной */}}
{{- $releaseName := .Release.Name -}}

{{/* Передача переменных в подшаблон */}}
{{- include "mychart.pod" (dict "Release" .Release "Values" .Values "extraLabels" (dict "app.kubernetes.io/component" "api")) -}}

{{/* В подшаблоне */}}
{{- define "mychart.pod" -}}
apiVersion: v1
kind: Pod
metadata:
  name: {{ .Release.Name }}
  labels:
    {{- range $key, $value := .extraLabels }}
    {{ $key }}: {{ $value }}
    {{- end }}
{{- end -}}
```

## Расширенная конфигурация безопасности

### Использование securityContext и podSecurityContext

```yaml
# values.yaml
securityContext:
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
  capabilities:
    drop:
      - ALL
    add:
      - NET_BIND_SERVICE
  readOnlyRootFilesystem: true
  runAsNonRoot: true
```

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
```

### Настройка NetworkPolicy

```yaml
# values.yaml
networkPolicy:
  enabled: true
  ingressRules:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/component: frontend
      ports:
        - protocol: TCP
          port: 80
```

```yaml
# templates/networkpolicy.yaml
{{- if .Values.networkPolicy.enabled }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  podSelector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  policyTypes:
    - Ingress
  ingress:
    {{- toYaml .Values.networkPolicy.ingressRules | nindent 4 }}
{{- end }}
```

## Интеграция с Service Mesh

### Настройка Istio

```yaml
# values.yaml
istio:
  enabled: true
  gateway:
    enabled: true
    hosts:
      - "myapp.example.com"
  virtualService:
    enabled: true
    hosts:
      - "myapp.example.com"
    gateways:
      - "istio-system/ingressgateway"
    http:
      - route:
        - destination:
            host: "{{ .Release.Name }}-service"
            port:
              number: 80
```

```yaml
# templates/istio-gateway.yaml
{{- if and .Values.istio.enabled .Values.istio.gateway.enabled }}
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: {{ include "mychart.fullname" . }}-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        {{- toYaml .Values.istio.gateway.hosts | nindent 8 }}
{{- end }}
```

## Оптимизация чартов и их производительности

### Организация зависимостей

Для оптимизации зависимостей используйте опцию `condition` и `tags`:

```yaml
# Chart.yaml
dependencies:
  - name: redis
    version: 17.3.14
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
    tags:
      - cache
  - name: postgresql
    version: 12.1.6
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
    tags:
      - database
```

```yaml
# values.yaml
# Глобальное отключение компонентов с определенным тегом
tags:
  cache: true
  database: false

# Состояние отдельных компонентов
redis:
  enabled: true
postgresql:
  enabled: false
```

### Условная генерация ресурсов

```yaml
# values.yaml
components:
  api:
    enabled: true
  worker:
    enabled: true
  scheduler:
    enabled: false
```

```yaml
# templates/api-deployment.yaml
{{- if .Values.components.api.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}-api
spec:
  # ...
{{- end }}
```

## Отладка и тестирование чартов

### Отладка шаблонов

```bash
# Получение отладочной информации
helm install --debug --dry-run myapp ./mychart

# Просмотр всех сгенерированных манифестов
helm template ./mychart > all-manifests.yaml

# Просмотр конкретного шаблона
helm template ./mychart -s templates/deployment.yaml
```

Вывод отладочной информации в шаблонах:

```go
{{/* Отладочный вывод */}}
{{ .Values | toYaml | nindent 2 }}
```

### Автоматическое тестирование чартов

Создание тестов для Helm чартов с использованием [chart-testing](https://github.com/helm/chart-testing):

```yaml
# ct.yaml
chart-dirs:
  - charts
excluded-charts:
  - common
chart-repos:
  - bitnami=https://charts.bitnami.com/bitnami
helm-extra-args: "--timeout 600s"
```

Запуск тестов:

```bash
ct lint --config ct.yaml
ct install --config ct.yaml
```

### Написание юнит-тестов для Helm чартов

Использование [helm-unittest](https://github.com/quintush/helm-unittest):

```yaml
# charts/mychart/tests/deployment_test.yaml
suite: Deployment test
templates:
  - deployment.yaml
tests:
  - it: should set the correct image
    set:
      image.repository: nginx
      image.tag: 1.19.0
    asserts:
      - equal:
          path: spec.template.spec.containers[0].image
          value: nginx:1.19.0
  - it: should set the correct replicas
    set:
      replicaCount: 3
    asserts:
      - equal:
          path: spec.replicas
          value: 3
```

Запуск тестов:

```bash
helm unittest ./charts/mychart
```

## Создание Umbrella-чартов

Umbrella-чарты объединяют несколько связанных приложений в одном чарте.

### Структура Umbrella-чарта

```
mystack/
├── Chart.yaml           # Основной чарт с зависимостями
├── values.yaml          # Переопределение значений для дочерних чартов
└── charts/              # Локальные чарты (если нужны)
    └── custom-app/      # Кастомное приложение
```

Пример `Chart.yaml`:

```yaml
apiVersion: v2
name: mystack
version: 0.1.0
description: A Helm chart for deploying the complete application stack
dependencies:
  - name: postgresql
    version: ~12.1.6
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
  - name: redis
    version: ~17.3.14
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
  - name: rabbitmq
    version: ~10.3.9
    repository: https://charts.bitnami.com/bitnami
    condition: rabbitmq.enabled
```

### Настройка коммуникации между компонентами

```yaml
# values.yaml
postgresql:
  enabled: true
  auth:
    username: myapp
    password: mypassword
    database: myapp

redis:
  enabled: true
  auth:
    password: myredispassword
    
api:
  config:
    databaseUrl: "postgresql://myapp:mypassword@{{ .Release.Name }}-postgresql:5432/myapp"
    redisUrl: "redis://:myredispassword@{{ .Release.Name }}-redis-master:6379/0"
```

## Шпаргалка по продвинутым командам Helm

| Категория | Команда | Описание |
|-----------|---------|----------|
| Отладка | `helm template --debug` | Вывод шаблонов с отладочной информацией |
| Отладка | `helm template -s FILE` | Вывод конкретного шаблона |
| Отладка | `helm install --dry-run --debug` | Симуляция установки с отладкой |
| Зависимости | `helm dependency update` | Обновление зависимостей |
| Зависимости | `helm dependency build` | Сборка зависимостей локально |
| Зависимости | `helm dependency list` | Список зависимостей |
| OCI | `helm chart save` | Сохранение чарта как OCI-артефакта |
| OCI | `helm chart push` | Загрузка чарта в OCI-реестр |
| OCI | `helm chart pull` | Загрузка чарта из OCI-реестра |
| OCI | `helm registry login` | Аутентификация в OCI-реестре |
| Плагины | `helm plugin install` | Установка плагина |
| Плагины | `helm plugin list` | Список установленных плагинов |
| Плагины | `helm plugin update` | Обновление плагина |
| Плагины | `helm plugin uninstall` | Удаление плагина |
| Тесты | `helm test` | Запуск тестов для релиза |
| Преобразование | `helm convert` | Преобразование старых чартов в новый формат |
| Подчарты | `helm pull --untar` | Загрузка и распаковка чарта |

## Заключение

Изучив продвинутые практики работы с Helm, вы сможете создавать более эффективные, безопасные и масштабируемые чарты, которые легко настраиваются и поддерживаются. Эти техники позволяют решать сложные задачи управления приложениями в Kubernetes, от многокомпонентных систем до интеграции со сложными инфраструктурными компонентами. 