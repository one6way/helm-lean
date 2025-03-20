# Система шаблонизации Helm

В этом документе рассматривается система шаблонизации Helm, основанная на Go Templates, которая позволяет создавать динамические Kubernetes-манифесты.

## Содержание
- [Основы Go Templates](#основы-go-templates)
- [Встроенные объекты в Helm](#встроенные-объекты-в-helm)
- [Функции и конвейеры](#функции-и-конвейеры)
- [Управляющие структуры](#управляющие-структуры)
- [Определение именованных шаблонов](#определение-именованных-шаблонов)
- [Включение файлов](#включение-файлов)
- [Работа с YAML](#работа-с-yaml)
- [Пространства имен и отступы](#пространства-имен-и-отступы)
- [Примеры шаблонов](#примеры-шаблонов)
- [Лучшие практики](#лучшие-практики)

## Основы Go Templates

Helm использует систему шаблонизации Go Templates, которая была расширена через библиотеку Sprig и дополнительные функции, специфичные для Helm.

### Синтаксис шаблонов

- `{{ }}` - Базовый синтаксис для выражения шаблона
- `{{- }}` - Trim левый пробел
- `{{- }}` - Trim правый пробел
- `{{ /* комментарий */ }}` - Комментарий

### Базовые выражения

```yaml
# Простая подстановка значения
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
```

### Типы данных

- **Строки**: `"abc"`, `'abc'`
- **Числа**: `42`, `3.14`
- **Булевы значения**: `true`, `false`
- **Списки**: `list 1 2 3`
- **Словари**: `.Values.image`
- **Nil**: `nil` (нулевое значение)

## Встроенные объекты в Helm

Helm предоставляет несколько встроенных объектов, которые доступны во всех шаблонах:

### .Release

Объект с информацией о релизе:

- `.Release.Name`: Имя релиза
- `.Release.Namespace`: Namespace, в котором установлен релиз
- `.Release.IsInstall`: `true`, если это установка
- `.Release.IsUpgrade`: `true`, если это обновление
- `.Release.Revision`: Номер ревизии
- `.Release.Service`: Сервис, который производит установку (обычно "Helm")

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  namespace: {{ .Release.Namespace }}
  labels:
    app: {{ .Release.Name }}
    release: {{ .Release.Name }}
    revision: "{{ .Release.Revision }}"
```

### .Chart

Содержит информацию из файла `Chart.yaml`:

- `.Chart.Name`: Имя чарта
- `.Chart.Version`: Версия чарта
- `.Chart.AppVersion`: Версия приложения
- `.Chart.Description`: Описание чарта
- `.Chart.Keywords`: Ключевые слова
- `.Chart.Home`: URL домашней страницы
- `.Chart.Sources`: Список источников
- `.Chart.Dependencies`: Список зависимостей
- `.Chart.Maintainers`: Список сопровождающих
- `.Chart.Type`: Тип чарта (application или library)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  labels:
    chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
    app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
```

### .Values

Содержит значения из файла `values.yaml` и значения, переданные через `--set` или `-f`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
data:
  app.conf: |-
    # Конфигурация приложения
    log_level: {{ .Values.logLevel | default "info" }}
    database_url: {{ .Values.database.url }}
    {{- if .Values.cache.enabled }}
    cache_host: {{ .Values.cache.host }}
    cache_port: {{ .Values.cache.port | default "6379" }}
    {{- end }}
```

### .Template

Информация о текущем шаблоне:

- `.Template.Name`: Имя текущего шаблона
- `.Template.BasePath`: Базовый путь к шаблонам

```yaml
# Этот шаблон: {{ .Template.Name }}
```

### .Capabilities

Информация о возможностях Kubernetes-кластера:

- `.Capabilities.APIVersions`: Список доступных API-версий
- `.Capabilities.APIVersions.Has "apps/v1"`: Проверяет, доступна ли API-версия
- `.Capabilities.KubeVersion`: Информация о версии Kubernetes
- `.Capabilities.KubeVersion.Major`: Мажорная версия Kubernetes
- `.Capabilities.KubeVersion.Minor`: Минорная версия Kubernetes
- `.Capabilities.KubeVersion.Version`: Полная строка версии Kubernetes

```yaml
{{- if .Capabilities.APIVersions.Has "networking.k8s.io/v1" }}
apiVersion: networking.k8s.io/v1
{{- else if .Capabilities.APIVersions.Has "networking.k8s.io/v1beta1" }}
apiVersion: networking.k8s.io/v1beta1
{{- else }}
apiVersion: extensions/v1beta1
{{- end }}
kind: Ingress
metadata:
  name: {{ .Release.Name }}-ingress
```

### .Files

Позволяет получить доступ к файлам в чарте:

- `.Files.Get "path/to/file.txt"`: Получить содержимое файла
- `.Files.GetBytes "path/to/file.txt"`: Получить содержимое файла как байты
- `.Files.Glob "path/to/*.yaml"`: Получить список файлов по шаблону
- `.Files.Lines "path/to/file.txt"`: Получить файл как список строк
- `.Files.AsSecrets`: Кодирует файлы в base64 для использования в Secret
- `.Files.AsConfig`: Получает файлы для использования в ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
data:
  {{- (.Files.Glob "config/*.conf").AsConfig | nindent 2 }}
  app.ini: {{ .Files.Get "config/app.ini" | quote }}
```

## Функции и конвейеры

В Go Templates функции имеют синтаксис `function arg1 arg2...`. Конвейеры (pipelines) позволяют передавать результат одной функции в качестве аргумента другой с помощью символа `|`.

### Основные функции

- **quote**: Заключает значение в кавычки
- **default**: Устанавливает значение по умолчанию
- **upper/lower**: Преобразует строку в верхний/нижний регистр
- **trim**: Удаляет пробельные символы
- **nindent**: Добавляет новую строку с отступом
- **toYaml**: Преобразует значение в YAML

```yaml
# Пример использования функций
ports:
  - name: http
    port: {{ .Values.service.port | default 80 }}
    protocol: TCP
    targetPort: {{ .Values.service.targetPort | default .Values.service.port }}

# Пример использования конвейеров
data:
  config.yaml: |-
    {{- .Values.config | toYaml | nindent 4 }}
```

### Основные функции Sprig

Helm включает библиотеку Sprig, которая добавляет множество полезных функций:

#### Строковые функции
- **trim**: `trim "  hello  "` → `"hello"`
- **trimAll**: `trimAll "$" "$5.00"` → `"5.00"`
- **trimPrefix**: `trimPrefix "-" "-hello"` → `"hello"`
- **trimSuffix**: `trimSuffix "-" "hello-"` → `"hello"`
- **upper**: `upper "hello"` → `"HELLO"`
- **lower**: `lower "HELLO"` → `"hello"`
- **title**: `title "hello world"` → `"Hello World"`
- **substr**: `substr 0 5 "hello world"` → `"hello"`
- **repeat**: `repeat 3 "hello"` → `"hellohellohello"`
- **nospace**: `nospace "hello world"` → `"helloworld"`
- **trunc**: `trunc 5 "hello world"` → `"hello"`
- **contains**: `contains "hello" "hello world"` → `true`
- **hasPrefix**: `hasPrefix "hello" "hello world"` → `true`
- **hasSuffix**: `hasSuffix "world" "hello world"` → `true`

#### Числовые функции
- **add**: `add 1 2` → `3`
- **sub**: `sub 3 2` → `1`
- **mul**: `mul 2 3` → `6`
- **div**: `div 6 3` → `2`
- **mod**: `mod 5 3` → `2`
- **max**: `max 1 2` → `2`
- **min**: `min 1 2` → `1`

#### Функции для коллекций
- **list**: `list 1 2 3` → `[1, 2, 3]`
- **first**: `first (list 1 2 3)` → `1`
- **rest**: `rest (list 1 2 3)` → `[2, 3]`
- **last**: `last (list 1 2 3)` → `3`
- **initial**: `initial (list 1 2 3)` → `[1, 2]`
- **append**: `append (list 1 2) 3` → `[1, 2, 3]`
- **prepend**: `prepend (list 2 3) 1` → `[1, 2, 3]`
- **compact**: `compact (list 1 "" 2)` → `[1, 2]`
- **sortAlpha**: `sortAlpha (list "c" "a" "b")` → `["a", "b", "c"]`

#### Словарные функции
- **dict**: `dict "key1" "value1" "key2" "value2"` → `{"key1": "value1", "key2": "value2"}`
- **get**: `get (dict "key1" "value1") "key1"` → `"value1"`
- **set**: `set (dict "key1" "value1") "key2" "value2"` → `{"key1": "value1", "key2": "value2"}`
- **unset**: `unset (dict "key1" "value1" "key2" "value2") "key2"` → `{"key1": "value1"}`
- **hasKey**: `hasKey (dict "key1" "value1") "key1"` → `true`
- **pluck**: `pluck "name" (list (dict "name" "fred") (dict "name" "barney"))` → `["fred", "barney"]`
- **merge**: `merge (dict "key1" "value1") (dict "key2" "value2")` → `{"key1": "value1", "key2": "value2"}`

#### Функции для кодирования
- **b64enc**: `b64enc "hello"` → `"aGVsbG8="`
- **b64dec**: `b64dec "aGVsbG8="` → `"hello"`
- **toJson**: `toJson (dict "key1" "value1")` → `{"key1":"value1"}`
- **fromJson**: `fromJson "{\"key1\":\"value1\"}"` → `{"key1": "value1"}`
- **toYaml**: `toYaml (dict "key1" "value1")` → `"key1: value1\n"`
- **fromYaml**: `fromYaml "key1: value1"` → `{"key1": "value1"}`

```yaml
# Примеры использования функций Sprig
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
  labels:
    {{- range $key, $value := .Values.labels }}
    {{ $key }}: {{ $value | quote }}
    {{- end }}
data:
  app.conf: |
    # Генерируем случайный пароль, если не указан
    password: {{ .Values.password | default (randAlphaNum 16) | quote }}
    
    # Преобразуем в верхний регистр и добавляем префикс
    environment: {{ .Values.env | upper | cat "ENV_" | quote }}
    
    # Проверяем наличие порта и устанавливаем значение по умолчанию
    port: {{ .Values.port | default 8080 }}
    
    # Собираем строку подключения к базе данных
    {{- $dbuser := .Values.db.username }}
    {{- $dbpass := .Values.db.password }}
    {{- $dbhost := .Values.db.host }}
    {{- $dbport := .Values.db.port | default 5432 }}
    {{- $dbname := .Values.db.name }}
    db_url: {{ printf "postgresql://%s:%s@%s:%d/%s" $dbuser $dbpass $dbhost $dbport $dbname | quote }}
```

## Управляющие структуры

Go Templates предоставляет несколько управляющих структур для условий, циклов и других операций.

### Условия (if/else)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
data:
  features.conf: |
    {{- if eq .Values.environment "production" }}
    log_level: error
    debug: false
    {{- else if eq .Values.environment "staging" }}
    log_level: warning
    debug: true
    {{- else }}
    log_level: debug
    debug: true
    {{- end }}
    
    {{- if .Values.cache.enabled }}
    cache: enabled
    cache_ttl: {{ .Values.cache.ttl | default 300 }}
    {{- else }}
    cache: disabled
    {{- end }}
```

Операторы сравнения:
- `eq`: Равно
- `ne`: Не равно
- `lt`: Меньше
- `le`: Меньше или равно
- `gt`: Больше
- `ge`: Больше или равно
- `and`: Логическое И
- `or`: Логическое ИЛИ
- `not`: Отрицание

### Циклы (range)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
data:
  allowed-ips.conf: |
    {{- range .Values.allowedIPs }}
    allow {{ . }};
    {{- end }}
  
  hosts.conf: |
    {{- range $host, $ip := .Values.hosts }}
    {{ $host }} {{ $ip }}
    {{- end }}
  
  servers.conf: |
    {{- range $i, $server := .Values.servers }}
    [server{{ $i }}]
    host = {{ $server.host }}
    port = {{ $server.port }}
    {{- end }}
```

### with

Директива `with` устанавливает область действия для переменных, что упрощает доступ к вложенным структурам:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          {{- with .Values.image }}
          image: "{{ .repository }}:{{ .tag | default $.Chart.AppVersion }}"
          imagePullPolicy: {{ .pullPolicy }}
          {{- end }}
          {{- with .Values.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
```

## Определение именованных шаблонов

Именованные шаблоны (named templates) позволяют определить блоки кода, которые можно многократно использовать в разных местах.

### Определение шаблона

Шаблоны обычно определяются в файле `_helpers.tpl`:

```
{{/* Определяем шаблон для меток */}}
{{- define "mychart.labels" -}}
app: {{ .Release.Name }}
chart: {{ .Chart.Name }}-{{ .Chart.Version }}
release: {{ .Release.Name }}
heritage: {{ .Release.Service }}
{{- end -}}

{{/* Определяем шаблон для selectors */}}
{{- define "mychart.selectorLabels" -}}
app: {{ .Release.Name }}
{{- end -}}

{{/* Определяем шаблон для конфигурации БД */}}
{{- define "mychart.dbConfig" -}}
database:
  host: {{ .Values.database.host }}
  port: {{ .Values.database.port | default 5432 }}
  name: {{ .Values.database.name }}
  user: {{ .Values.database.user }}
{{- end -}}
```

### Использование шаблона

Определенные шаблоны можно использовать с помощью функций `include` и `template`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
```

> **Важно**: Рекомендуется использовать `include` вместо `template`, так как `include` позволяет передавать результат в другие функции через конвейер, например, `nindent`.

## Включение файлов

Шаблоны могут включать другие файлы с помощью функций из объекта `.Files`.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
data:
  # Включение отдельного файла
  app.conf: |-
{{ .Files.Get "config/app.conf" | indent 4 }}

  # Включение всех файлов из директории
  {{- range $path, $content := .Files.Glob "config/**.yaml" }}
  {{ base $path }}: |-
{{ $content | indent 4 }}
  {{- end }}
```

## Работа с YAML

Helm предоставляет функции для работы с YAML, которые упрощают генерацию сложных структур.

### toYaml

Функция `toYaml` преобразует объект в YAML:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
spec:
  {{- with .Values.deployment }}
  replicas: {{ .replicas }}
  {{- if .strategy }}
  strategy:
    {{- toYaml .strategy | nindent 4 }}
  {{- end }}
  {{- end }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          {{- with .Values.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
```

### fromYaml

Функция `fromYaml` преобразует строку YAML в объект:

```yaml
{{- $config := .Files.Get "config/app.yaml" | fromYaml }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
data:
  database_host: {{ $config.database.host }}
  database_port: {{ $config.database.port | quote }}
```

## Пространства имен и отступы

Правильное управление отступами в YAML является важной частью шаблонизации Helm.

### Управление пробелами

Используйте `{{-` и `-}}` для удаления лишних пробелов:

```yaml
# Без управления пробелами
{{ if .Values.enabled }}
enabled: true
{{ end }}

# С управлением пробелами
{{- if .Values.enabled }}
enabled: true
{{- end }}
```

### Функция indent и nindent

Функции `indent` и `nindent` позволяют добавлять отступы к многострочному тексту:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
spec:
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

## Примеры шаблонов

### Пример deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "mychart.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          {{- if .Values.command }}
          command:
            {{- toYaml .Values.command | nindent 12 }}
          {{- end }}
          {{- if .Values.args }}
          args:
            {{- toYaml .Values.args | nindent 12 }}
          {{- end }}
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP
          {{- if .Values.probes.liveness.enabled }}
          livenessProbe:
            {{- toYaml .Values.probes.liveness.probe | nindent 12 }}
          {{- end }}
          {{- if .Values.probes.readiness.enabled }}
          readinessProbe:
            {{- toYaml .Values.probes.readiness.probe | nindent 12 }}
          {{- end }}
          {{- if .Values.probes.startup.enabled }}
          startupProbe:
            {{- toYaml .Values.probes.startup.probe | nindent 12 }}
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          {{- with .Values.env }}
          env:
            {{- range $key, $value := . }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
          {{- end }}
          {{- with .Values.envFrom }}
          envFrom:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          {{- with .Values.volumeMounts }}
          volumeMounts:
            {{- toYaml . | nindent 12 }}
          {{- end }}
      {{- with .Values.volumes }}
      volumes:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### Пример ingress.yaml с условной логикой

```yaml
{{- if .Values.ingress.enabled -}}
{{- $fullName := include "mychart.fullname" . -}}
{{- $svcPort := .Values.service.port -}}
{{- if and .Values.ingress.className (not (semverCompare ">=1.18-0" .Capabilities.KubeVersion.Version)) }}
  {{- if not (hasKey .Values.ingress.annotations "kubernetes.io/ingress.class") }}
  {{- $_ := set .Values.ingress.annotations "kubernetes.io/ingress.class" .Values.ingress.className}}
  {{- end }}
{{- end }}
{{- if semverCompare ">=1.19-0" .Capabilities.KubeVersion.Version -}}
apiVersion: networking.k8s.io/v1
{{- else if semverCompare ">=1.14-0" .Capabilities.KubeVersion.Version -}}
apiVersion: networking.k8s.io/v1beta1
{{- else -}}
apiVersion: extensions/v1beta1
{{- end }}
kind: Ingress
metadata:
  name: {{ $fullName }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if and .Values.ingress.className (semverCompare ">=1.18-0" .Capabilities.KubeVersion.Version) }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            {{- if and .pathType (semverCompare ">=1.18-0" $.Capabilities.KubeVersion.Version) }}
            pathType: {{ .pathType }}
            {{- end }}
            backend:
              {{- if semverCompare ">=1.19-0" $.Capabilities.KubeVersion.Version }}
              service:
                name: {{ $fullName }}
                port:
                  number: {{ $svcPort }}
              {{- else }}
              serviceName: {{ $fullName }}
              servicePort: {{ $svcPort }}
              {{- end }}
          {{- end }}
    {{- end }}
{{- end }}
```

### Пример configmap с данными из файла

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "mychart.fullname" . }}-config
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
data:
  {{- if .Files.Glob "config/*.conf" }}
  {{- range $path, $content := .Files.Glob "config/*.conf" }}
  {{ base $path }}: |-
{{ $content | indent 4 }}
  {{- end }}
  {{- end }}
  
  app.yaml: |-
    # Генерируемая конфигурация
    app_name: {{ .Values.appName | default .Chart.Name | quote }}
    port: {{ .Values.service.port }}
    
    # Настройки базы данных
    {{- with .Values.database }}
    database:
      host: {{ .host | quote }}
      port: {{ .port }}
      name: {{ .name | quote }}
      {{- if .username }}
      username: {{ .username | quote }}
      {{- end }}
    {{- end }}
    
    # Другие настройки
    {{- if .Values.features }}
    features:
      {{- range $feature, $enabled := .Values.features }}
      {{ $feature }}: {{ $enabled }}
      {{- end }}
    {{- end }}
```

## Лучшие практики

### Именование шаблонов

- Используйте префикс с именем чарта для именованных шаблонов: `mychart.fullname`
- Группируйте связанные шаблоны вместе
- Добавляйте комментарии, описывающие назначение шаблона

### Структурирование шаблонов

- Разделяйте большие шаблоны на логические блоки
- Используйте именованные шаблоны для повторяющегося кода
- Храните общие функции в файле `_helpers.tpl`

### Обработка ошибок

- Проверяйте обязательные значения:

```yaml
{{- if not .Values.requiredValue }}
{{- fail "requiredValue must be set" }}
{{- end }}
```

- Устанавливайте значения по умолчанию для необязательных параметров:

```yaml
port: {{ .Values.port | default 8080 }}
```

- Используйте условия для проверки структуры данных:

```yaml
{{- if and .Values.database (hasKey .Values.database "host") }}
host: {{ .Values.database.host }}
{{- else }}
host: localhost
{{- end }}
```

### Отладка шаблонов

Используйте функцию `helm template` для проверки сгенерированных манифестов:

```bash
helm template . --debug
```

Используйте функцию `helm install --dry-run --debug` для тестирования установки:

```bash
helm install my-release . --dry-run --debug
```

### Пользовательские комментарии

Добавляйте полезные комментарии в шаблоны:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
  {{- with .Values.annotations }}
  annotations:
    {{/* Loop through annotations */}}
    {{- toYaml . | nindent 4 }}
  {{- end }}
```

### Универсальность шаблонов

Делайте шаблоны достаточно гибкими для разных версий Kubernetes:

```yaml
{{- if semverCompare ">=1.19-0" .Capabilities.KubeVersion.Version }}
apiVersion: networking.k8s.io/v1
{{- else }}
apiVersion: networking.k8s.io/v1beta1
{{- end }}
``` 