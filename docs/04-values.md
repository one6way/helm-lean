# Работа с Values и переменными в Helm

Values (значения) - ключевой механизм в Helm, который позволяет настраивать чарты для разных окружений и случаев использования. В этом разделе рассмотрим, как работать с values.yaml и эффективно использовать переменные в шаблонах.

## Основные концепции Values

### Что такое Values?

Values - это параметры конфигурации, передаваемые в чарт для настройки генерируемых манифестов. Они определяют, как должно быть сконфигурировано приложение:

- Количество реплик
- Значения ресурсов (CPU, память)
- Настройки хранилища
- Конфигурационные параметры (env переменные, конфиги)
- Версии образов
- И многое другое

### Источники Values

Helm получает values из нескольких источников в следующем порядке приоритета (от низшего к высшему):

1. Значения по умолчанию из файла `values.yaml` в чарте
2. Значения из родительского чарта (если чарт является подчартом)
3. Значения из пользовательского файла values, указанного через опцию `-f` или `--values`
4. Значения, переданные через опцию `--set`

## Структура values.yaml

Типичный файл `values.yaml` структурирован иерархически и содержит связанные группы параметров:

```yaml
# Базовые параметры приложения
replicaCount: 1
image:
  repository: nginx
  tag: "1.21.6"
  pullPolicy: IfNotPresent

# Конфигурация сервиса
service:
  type: ClusterIP
  port: 80

# Конфигурация ingress
ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

# Ресурсы
resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi

# Autoscaling
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
```

### Лучшие практики структурирования values.yaml

1. **Логическая группировка**: Объединяйте связанные параметры в логические группы
2. **Обеспечьте разумные значения по умолчанию**: Все параметры должны иметь рабочие значения по умолчанию
3. **Используйте комментарии**: Поясняйте назначение параметров и блоков
4. **Не храните секреты**: Избегайте хранения паролей и ключей в values.yaml
5. **Поддерживайте иерархию**: Используйте вложенные структуры для организации сложных конфигураций

## Доступ к Values в шаблонах

### Базовый синтаксис

Для доступа к значениям в шаблонах используется объект `.Values`:

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
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: {{ .Values.service.port }}
```

### Доступ к вложенным значениям

Для доступа к вложенным значениям используется точечная нотация:

```yaml
{{ .Values.image.repository }}
```

### Проверка наличия значений

Используйте функции и условные выражения, чтобы проверить наличие значений:

```yaml
{{- if .Values.serviceAccount.create -}}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "mychart.serviceAccountName" . }}
{{- end }}
```

## Переопределение Values при установке

### Использование файла values

Для переопределения значений через файл:

```bash
helm install -f myvalues.yaml myrelease ./mychart
```

```bash
helm upgrade -f myvalues.yaml myrelease ./mychart
```

Файл `myvalues.yaml` может содержать только те параметры, которые нужно переопределить:

```yaml
# myvalues.yaml
replicaCount: 3
resources:
  requests:
    cpu: 200m
```

### Использование --set

Для переопределения отдельных значений используйте флаг `--set`:

```bash
helm install --set replicaCount=3 --set resources.requests.cpu=200m myrelease ./mychart
```

Для сложных строк или значений со специальными символами используйте одинарные кавычки:

```bash
helm install --set nodeSelector."kubernetes\.io/role"=master myrelease ./mychart
```

### Режим --set-string

Использует `--set-string` для явного указания строкового типа (когда нужно избежать интерпретации числа или булева значения):

```bash
helm install --set-string version=1.2.3 myrelease ./mychart
```

### Использование --set-file

Для передачи содержимого файла в значение используйте `--set-file`:

```bash
helm install --set-file config.json=./config.json myrelease ./mychart
```

## Работа с Values для подчартов

### Структура Values для подчартов

```yaml
# Главный values.yaml
mysql:  # Имя подчарта
  auth:
    rootPassword: mypassword
    database: mydatabase

myapp:  # Значения для собственного чарта
  replicaCount: 2
```

### Глобальные Values

Значения в секции `global` доступны во всех подчартах:

```yaml
# Главный values.yaml
global:
  imageRegistry: myregistry.com
  imagePullSecrets: 
    - name: regcred

mysql:
  image:
    repository: mysql  # Будет преобразовано в myregistry.com/mysql
```

В шаблоне подчарта:

```yaml
image: "{{ .Values.global.imageRegistry }}/{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

## Использование схем Values (values.schema.json)

Схемы values позволяют валидировать вводимые значения по JSON Schema:

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "required": [
    "image"
  ],
  "properties": {
    "image": {
      "type": "object",
      "required": [
        "repository",
        "tag"
      ],
      "properties": {
        "repository": {
          "type": "string"
        },
        "tag": {
          "type": "string"
        }
      }
    },
    "replicaCount": {
      "type": "integer",
      "minimum": 1
    }
  }
}
```

## Работа с секретными данными

### Подход 1: Внешние секреты

Рекомендуемый подход - не хранить секреты в values:

```yaml
# values.yaml
existingSecret: my-db-credentials
```

```yaml
# templates/deployment.yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: {{ .Values.existingSecret }}
      key: password
```

### Подход 2: Отдельный защищенный файл

```yaml
# secret-values.yaml (хранится в защищенном месте)
dbPassword: supersecret
apiKey: topsecret
```

```bash
helm install -f values.yaml -f secret-values.yaml myrelease ./mychart
```

### Подход 3: Использование external-secrets или sealed-secrets

```yaml
# templates/externalsecret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: {{ .Release.Name }}-db-credentials
spec:
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: {{ .Release.Name }}-db-credentials
  data:
  - secretKey: password
    remoteRef:
      key: databases/{{ .Values.environment }}/mysql
      property: password
```

## Продвинутые техники работы с Values

### Использование переменных шаблона

```yaml
{{- $releaseName := .Release.Name -}}
{{- $chartName := .Chart.Name -}}

spec:
  template:
    metadata:
      labels:
        app: {{ $chartName }}
        release: {{ $releaseName }}
```

### Динамические ключи в Values

```yaml
# values.yaml
env:
  BACKEND_URL: http://backend-service
  API_KEY: 1234567
  DEBUG: true
```

```yaml
# templates/deployment.yaml
env:
{{- range $key, $value := .Values.env }}
- name: {{ $key }}
  value: {{ $value | quote }}
{{- end }}
```

### Объединение значений

```yaml
{{- $defaultConfig := .Values.defaultConfig -}}
{{- $overrideConfig := .Values.overrideConfig -}}
{{- $mergedConfig := mergeOverwrite $defaultConfig $overrideConfig -}}

config: {{ toYaml $mergedConfig | nindent 2 }}
```

## Отладка Values

### Использование --debug и --dry-run

```bash
helm install --debug --dry-run myrelease ./mychart
```

### Использование helm template

```bash
helm template -f myvalues.yaml ./mychart
```

### Вывод отдельных шаблонов

```bash
helm template -f myvalues.yaml -s templates/deployment.yaml ./mychart
```

### Использование .helm/debug function

В шаблонах:
```yaml
{{ .Values | toYaml | nindent 2 }}
```

## Шпаргалка по работе с Values

| Операция | Команда |
|----------|---------|
| Установка с файлом values | `helm install -f myvalues.yaml myrelease ./mychart` |
| Установка с несколькими файлами values | `helm install -f common.yaml -f env.yaml myrelease ./mychart` |
| Установка с переопределением значений | `helm install --set key1=val1,key2=val2 myrelease ./mychart` |
| Обновление с файлом values | `helm upgrade -f myvalues.yaml myrelease ./mychart` |
| Отладка values | `helm template -f myvalues.yaml ./mychart` |
| Проверка схемы values | `helm lint --strict ./mychart` |
| Просмотр значений установленного релиза | `helm get values myrelease` |

## Заключение

Эффективное управление Values - ключевой навык при работе с Helm. Правильно структурированные и документированные values.yaml делают ваши чарты гибкими, многократно используемыми и понятными для других разработчиков. Следуя лучшим практикам и применяя продвинутые техники, вы сможете создавать чарты, которые легко адаптировать для различных окружений и сценариев использования. 