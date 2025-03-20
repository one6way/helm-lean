# Практические примеры использования Helm

В этом разделе собраны конкретные примеры использования Helm для решения практических задач. Здесь вы найдете типовые сценарии, с которыми часто сталкиваются DevOps-инженеры и разработчики при работе с Kubernetes.

## Пример 1: Развертывание веб-приложения с базой данных

Этот пример демонстрирует развертывание полного стека приложения, состоящего из фронтенда, бэкенда и базы данных PostgreSQL.

### Структура Umbrella-чарта

```
myapp/
├── Chart.yaml
├── values.yaml
├── charts/
│   ├── frontend/
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       └── ingress.yaml
│   └── backend/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           └── configmap.yaml
└── templates/
    └── secrets.yaml
```

### Chart.yaml основного чарта

```yaml
apiVersion: v2
name: myapp
description: A complete application stack
version: 0.1.0
appVersion: "1.0.0"
dependencies:
  - name: postgresql
    version: 11.9.13
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

### values.yaml основного чарта

```yaml
# Глобальные настройки
global:
  environment: production
  domain: myapp.example.com

# Настройки PostgreSQL
postgresql:
  enabled: true
  auth:
    username: myapp
    password: "your-password"
    database: myapp
  primary:
    persistence:
      size: 10Gi

# Настройки фронтенда
frontend:
  replicaCount: 2
  image:
    repository: myregistry/frontend
    tag: 1.0.0
  ingress:
    enabled: true
    className: nginx
    hosts:
      - host: "{{ .Values.global.domain }}"
        paths:
          - path: /
            pathType: Prefix

# Настройки бэкенда
backend:
  replicaCount: 3
  image:
    repository: myregistry/backend
    tag: 1.0.0
  config:
    logLevel: info
    dbHost: "{{ .Release.Name }}-postgresql"
    dbName: "{{ .Values.postgresql.auth.database }}"
    dbUser: "{{ .Values.postgresql.auth.username }}"
    apiKey: "your-api-key"
```

### Команда установки

```bash
helm install myapp ./myapp \
  --set global.domain=prod.example.com \
  --set postgresql.auth.password=$(openssl rand -base64 32) \
  --set backend.config.apiKey=$(openssl rand -base64 32) \
  --namespace myapp \
  --create-namespace
```

## Пример 2: Настройка мониторинга с Prometheus и Grafana

Этот пример показывает, как развернуть стек мониторинга Prometheus и Grafana и настроить мониторинг вашего приложения.

### Установка мониторинг-стека

```bash
# Добавление репозитория Prometheus
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Установка Prometheus стека (включает Prometheus, Alertmanager, Grafana)
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f prometheus-values.yaml
```

### prometheus-values.yaml

```yaml
prometheus:
  prometheusSpec:
    serviceMonitorSelectorNilUsesHelmValues: false
    serviceMonitorSelector: {}
    serviceMonitorNamespaceSelector: {}
    podMonitorSelectorNilUsesHelmValues: false
    podMonitorSelector: {}
    podMonitorNamespaceSelector: {}
    retention: 30d
    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: 1000m
        memory: 4Gi

grafana:
  adminPassword: "admin-password"
  persistence:
    enabled: true
    size: 10Gi
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
      - name: 'default'
        orgId: 1
        folder: ''
        type: file
        disableDeletion: false
        editable: true
        options:
          path: /var/lib/grafana/dashboards/default
```

### ServiceMonitor для вашего приложения

```yaml
# myapp-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
  namespace: myapp
  labels:
    app: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    interval: 15s
    path: /metrics
  namespaceSelector:
    matchNames:
    - myapp
```

### Применение ServiceMonitor

```bash
kubectl apply -f myapp-servicemonitor.yaml
```

## Пример 3: Развертывание GitLab с помощью Helm

Пример установки полного стека GitLab в Kubernetes.

### Добавление репозитория GitLab

```bash
helm repo add gitlab https://charts.gitlab.io/
helm repo update
```

### Создание файла values для GitLab

```yaml
# gitlab-values.yaml
global:
  edition: ce
  hosts:
    domain: gitlab.example.com
    https: true
    externalIP: 203.0.113.1
  
  ## Настройка TLS с Let's Encrypt
  certmanager-issuer:
    email: admin@example.com

## Настройка хранилища
postgresql:
  persistence:
    size: 20Gi

redis:
  persistence:
    size: 10Gi

gitaly:
  persistence:
    size: 50Gi

## Настройка количества реплик
gitlab-runner:
  replicas: 2

## Настройка Ingress
nginx-ingress:
  enabled: true
  controller:
    service:
      loadBalancerIP: 203.0.113.1

certmanager:
  install: true
  
prometheus:
  install: true
```

### Установка GitLab

```bash
helm install gitlab gitlab/gitlab \
  --namespace gitlab \
  --create-namespace \
  -f gitlab-values.yaml \
  --timeout 600s
```

### Получение пароля root пользователя

```bash
kubectl get secret gitlab-gitlab-initial-root-password -n gitlab -o jsonpath="{.data.password}" | base64 --decode
```

## Пример 4: Автомасштабирование приложения

Пример настройки горизонтального автомасштабирования для приложения.

### values.yaml с настройками HPA

```yaml
# values.yaml
replicaCount: 2

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

### Шаблон для HPA

```yaml
# templates/hpa.yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "myapp.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
  {{- if .Values.autoscaling.targetCPUUtilizationPercentage }}
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
  {{- end }}
  {{- if .Values.autoscaling.targetMemoryUtilizationPercentage }}
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
  {{- end }}
{{- end }}
```

### Проверка работы HPA

```bash
kubectl get hpa -n myapp
```

## Пример 5: Развертывание микросервисной архитектуры

Пример развертывания сложной микросервисной архитектуры с помощью Helm.

### Структура проекта

```
microservices/
├── Chart.yaml
├── values.yaml
└── charts/
    ├── auth-service/
    ├── api-gateway/
    ├── user-service/
    ├── order-service/
    ├── payment-service/
    ├── notification-service/
    ├── frontend/
    └── mongodb/
```

### Chart.yaml для микросервисов

```yaml
apiVersion: v2
name: microservices
description: Микросервисная архитектура
version: 0.1.0
dependencies:
  - name: mongodb
    version: 13.6.2
    repository: https://charts.bitnami.com/bitnami
    condition: mongodb.enabled
  - name: redis
    version: 17.3.14
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
  - name: rabbitmq
    version: 10.3.9
    repository: https://charts.bitnami.com/bitnami
    condition: rabbitmq.enabled
```

### values.yaml для микросервисов

```yaml
global:
  environment: production
  imagePullSecrets:
    - name: regcred
  domain: app.example.com
  
mongodb:
  enabled: true
  auth:
    rootPassword: "mongodb-root-password"
    username: appuser
    password: "mongodb-password"
    database: microservices
  
redis:
  enabled: true
  auth:
    password: "redis-password"
  
rabbitmq:
  enabled: true
  auth:
    username: user
    password: "rabbitmq-password"
  
api-gateway:
  replicaCount: 2
  image:
    repository: myregistry/api-gateway
    tag: 1.0.0
  ingress:
    enabled: true
    hosts:
      - "api.{{ .Values.global.domain }}"
  
auth-service:
  replicaCount: 2
  image:
    repository: myregistry/auth-service
    tag: 1.0.0
  
user-service:
  replicaCount: 2
  image:
    repository: myregistry/user-service
    tag: 1.0.0
  
order-service:
  replicaCount: 2
  image:
    repository: myregistry/order-service
    tag: 1.0.0
  
payment-service:
  replicaCount: 2
  image:
    repository: myregistry/payment-service
    tag: 1.0.0
  
notification-service:
  replicaCount: 2
  image:
    repository: myregistry/notification-service
    tag: 1.0.0
  
frontend:
  replicaCount: 3
  image:
    repository: myregistry/frontend
    tag: 1.0.0
  ingress:
    enabled: true
    hosts:
      - "{{ .Values.global.domain }}"
```

### Команда установки

```bash
helm install microservices ./microservices \
  --namespace microservices \
  --create-namespace \
  -f production-values.yaml
```

## Пример 6: Управление секретами в Helm с помощью SealedSecrets

Пример использования Bitnami SealedSecrets для безопасного хранения секретов в Git.

### Установка SealedSecrets контроллера

```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system
```

### Создание SealedSecret

```bash
# Создаем обычный Secret
kubectl create secret generic mysecret \
  --namespace myapp \
  --dry-run=client \
  --from-literal=db-password=your-password \
  --from-literal=api-key=your-api-key \
  -o yaml > secret.yaml

# Запечатываем Secret
kubeseal --controller-name=sealed-secrets \
  --namespace kube-system \
  --scope namespace-wide \
  < secret.yaml > sealed-secret.yaml
```

### Использование SealedSecret в Helm чарте

```yaml
# templates/sealedsecret.yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: {{ include "myapp.fullname" . }}-secrets
  namespace: {{ .Release.Namespace }}
spec:
  encryptedData:
    db-password: AgBy8hG...
    api-key: AgB5nke...
  template:
    metadata:
      name: {{ include "myapp.fullname" . }}-secrets
      namespace: {{ .Release.Namespace }}
```

### Использование секрета в Deployment

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ include "myapp.fullname" . }}-secrets
                  key: db-password
            - name: API_KEY
              valueFrom:
                secretKeyRef:
                  name: {{ include "myapp.fullname" . }}-secrets
                  key: api-key
```

## Пример 7: Развертывание Kafka кластера

Пример установки и настройки кластера Apache Kafka.

### Установка Kafka

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm install kafka bitnami/kafka \
  --namespace kafka \
  --create-namespace \
  -f kafka-values.yaml
```

### kafka-values.yaml

```yaml
# kafka-values.yaml
global:
  storageClass: "standard"

replicaCount: 3

persistence:
  enabled: true
  size: 50Gi

zookeeper:
  enabled: true
  replicaCount: 3
  persistence:
    enabled: true
    size: 10Gi

externalAccess:
  enabled: true
  service:
    type: LoadBalancer
    ports:
      external: 9094
  autoDiscovery:
    enabled: true

metrics:
  kafka:
    enabled: true
  jmx:
    enabled: true
  serviceMonitor:
    enabled: true
    namespace: monitoring

rbac:
  create: true
```

### Создание топика Kafka

```bash
kubectl run kafka-client --restart='Never' --image docker.io/bitnami/kafka:3.4.0 --namespace kafka --command -- sleep infinity

kubectl exec -it kafka-client -n kafka -- kafka-topics.sh \
  --create --topic my-topic \
  --bootstrap-server kafka.kafka.svc.cluster.local:9092 \
  --replication-factor 3 \
  --partitions 10
```

## Пример 8: Настройка Ingress с TLS

Пример настройки Ingress с TLS и автоматическим управлением сертификатами через cert-manager.

### Установка cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

### Создание Issuer для Let's Encrypt

```yaml
# letsencrypt-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

### Применение Issuer

```bash
kubectl apply -f letsencrypt-issuer.yaml
```

### values.yaml с настройками Ingress и TLS

```yaml
# values.yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/tls-acme: "true"
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: myapp-tls
      hosts:
        - myapp.example.com
```

### Шаблон Ingress с TLS

```yaml
# templates/ingress.yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
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
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "myapp.fullname" $ }}
                port:
                  number: {{ $.Values.service.port }}
          {{- end }}
    {{- end }}
{{- end }}
```

## Пример 9: Настройка бэкапов с Velero

Этот пример показывает, как настроить Velero для резервного копирования ресурсов Kubernetes.

### Установка Velero с MinIO в качестве хранилища

```bash
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm repo update

helm install velero vmware-tanzu/velero \
  --namespace velero \
  --create-namespace \
  -f velero-values.yaml
```

### velero-values.yaml

```yaml
# velero-values.yaml
configuration:
  provider: aws
  backupStorageLocation:
    name: default
    provider: aws
    bucket: velero
    config:
      region: minio
      s3ForcePathStyle: true
      s3Url: http://minio.minio.svc.cluster.local:9000
  volumeSnapshotLocation:
    name: default
    provider: aws
    config:
      region: minio

credentials:
  useSecret: true
  secretContents:
    cloud: |
      [default]
      aws_access_key_id = minio
      aws_secret_access_key = minio123

initContainers:
- name: velero-plugin-for-aws
  image: velero/velero-plugin-for-aws:v1.5.0
  volumeMounts:
  - mountPath: /target
    name: plugins

metrics:
  enabled: true
  serviceMonitor:
    enabled: true

snapshotsEnabled: true
deployRestic: true
restic:
  privileged: true
```

### Создание бэкапа

```bash
# Создание резервной копии всего namespace
kubectl -n velero create -f - <<EOF
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: myapp-backup
  namespace: velero
spec:
  includedNamespaces:
  - myapp
  storageLocation: default
  volumeSnapshotLocations:
  - default
  includedResources:
  - '*'
  ttl: 720h
EOF
```

### Восстановление из бэкапа

```bash
kubectl -n velero create -f - <<EOF
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: myapp-restore
  namespace: velero
spec:
  backupName: myapp-backup
  includedNamespaces:
  - myapp
  restorePVs: true
EOF
```

## Пример 10: Управление сетевыми политиками

Пример настройки NetworkPolicy для обеспечения безопасной сетевой коммуникации между микросервисами.

### values.yaml с настройками NetworkPolicy

```yaml
# values.yaml
networkPolicies:
  enabled: true
  ingressRules:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: api-gateway
      ports:
        - protocol: TCP
          port: 8080
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: prometheus
      ports:
        - protocol: TCP
          port: 9090
```

### Шаблон для NetworkPolicy

```yaml
# templates/networkpolicy.yaml
{{- if .Values.networkPolicies.enabled }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  podSelector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    {{- toYaml .Values.networkPolicies.ingressRules | nindent 4 }}
  egress:
    - {}  # Разрешить весь исходящий трафик
{{- end }}
```

## Пример 11: Настройка StatefulSet

Пример создания StatefulSet с постоянным хранилищем и Headless Service для стабильных сетевых идентификаторов.

### values.yaml для StatefulSet

```yaml
# values.yaml
replicaCount: 3

statefulSet:
  serviceName: myapp
  podManagementPolicy: Parallel
  updateStrategy:
    type: RollingUpdate

persistence:
  enabled: true
  storageClass: "standard"
  size: 10Gi
  accessModes:
    - ReadWriteOnce

service:
  type: ClusterIP
  port: 80
  headless: true
```

### Шаблон StatefulSet

```yaml
# templates/statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  serviceName: {{ .Values.statefulSet.serviceName }}
  replicas: {{ .Values.replicaCount }}
  podManagementPolicy: {{ .Values.statefulSet.podManagementPolicy }}
  updateStrategy:
    type: {{ .Values.statefulSet.updateStrategy.type }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          ports:
            - containerPort: 80
              name: http
          volumeMounts:
            - name: data
              mountPath: /data
  {{- if .Values.persistence.enabled }}
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: {{ .Values.persistence.accessModes }}
        storageClassName: {{ .Values.persistence.storageClass }}
        resources:
          requests:
            storage: {{ .Values.persistence.size }}
  {{- end }}
```

### Шаблон Headless Service

```yaml
# templates/service-headless.yaml
{{- if .Values.service.headless }}
apiVersion: v1
kind: Service
metadata:
  name: {{ include "myapp.fullname" . }}-headless
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  clusterIP: None
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "myapp.selectorLabels" . | nindent 4 }}
{{- end }}
```

## Заключение

В этом разделе мы рассмотрели практические примеры использования Helm для решения различных задач: от простого развертывания приложения до настройки сложных микросервисных архитектур, интеграции с системами мониторинга, управления секретами и обеспечения безопасности.

Эти примеры демонстрируют гибкость и мощь Helm как инструмента для управления приложениями в Kubernetes и могут служить отправной точкой для ваших собственных проектов. Используйте их как шаблоны и адаптируйте под свои специфические потребности.

Помните, что для каждого примера важно понимать специфику именно вашего проекта и инфраструктуры. Решения, представленные здесь, могут требовать адаптации для конкретных условий эксплуатации, масштаба и требований производительности. 