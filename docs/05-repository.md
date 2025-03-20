# Управление репозиториями Helm

Репозитории Helm - это коллекции упакованных чартов, которые можно скачивать и использовать в своих проектах. Использование репозиториев упрощает распространение, обновление и повторное использование чартов.

## Концепция репозиториев Helm

### Что такое репозиторий Helm?

Репозиторий Helm - это HTTP-сервер, который содержит:
- Индексный файл (index.yaml) со списком доступных чартов
- Упакованные чарты (файлы .tgz)
- Опционально - провенансные файлы (подписи для проверки подлинности чартов)

Репозитории помогают:
- Организовать и хранить чарты
- Делиться чартами между командами и сообществом
- Обеспечивать версионирование и обновление чартов
- Управлять зависимостями между чартами

## Работа с публичными репозиториями

### Популярные публичные репозитории

| Название | URL | Описание |
|----------|-----|----------|
| Artifact Hub | https://artifacthub.io/ | Официальный каталог чартов (заменил Helm Hub) |
| Bitnami | https://charts.bitnami.com/bitnami | Обширная коллекция чартов для популярных приложений |
| Kubernetes | https://kubernetes-charts.storage.googleapis.com/ | Официальные стабильные чарты (устаревший) |
| Elastic | https://helm.elastic.co | Чарты для Elasticsearch, Kibana и других продуктов Elastic |
| Prometheus | https://prometheus-community.github.io/helm-charts | Чарты для Prometheus и связанных инструментов |
| Grafana | https://grafana.github.io/helm-charts | Чарты для Grafana и связанных продуктов |

### Добавление репозитория

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

### Обновление репозиториев

Обновление загружает последнюю версию индекса репозитория:

```bash
helm repo update
```

Обновление конкретного репозитория:

```bash
helm repo update bitnami
```

### Поиск чартов в репозиториях

```bash
# Поиск по всем репозиториям
helm search repo mysql

# Поиск в конкретном репозитории
helm search repo bitnami/mysql

# Поиск с включением всех версий
helm search repo mysql --versions
```

### Просмотр информации о чарте

```bash
helm show chart bitnami/mysql
helm show values bitnami/mysql
helm show readme bitnami/mysql
helm show all bitnami/mysql
```

### Удаление репозитория

```bash
helm repo remove bitnami
```

### Список добавленных репозиториев

```bash
helm repo list
```

## Создание собственного репозитория

### Простой статический репозиторий

#### 1. Упаковка чартов

```bash
helm package ./mychart
```

#### 2. Создание индекса

```bash
helm repo index .
```

#### 3. Размещение на HTTP-сервере

Можно использовать:
- GitHub/GitLab Pages
- AWS S3 + CloudFront
- Google Cloud Storage
- Обычный веб-сервер (NGINX, Apache)

#### Пример с GitHub Pages

```bash
# Структура репозитория
helm-repo/
  ├── index.yaml
  ├── mychart-0.1.0.tgz
  └── README.md

# Создание индекса
cd helm-repo
helm repo index .

# Публикация
git add .
git commit -m "Update helm repository"
git push origin gh-pages
```

### Использование ChartMuseum

[ChartMuseum](https://chartmuseum.com/) - серверное приложение для управления репозиториями Helm.

#### Установка ChartMuseum

```bash
# Установка с помощью Helm
helm repo add chartmuseum https://chartmuseum.github.io/charts
helm install my-chartmuseum chartmuseum/chartmuseum \
  --set env.open.STORAGE=local \
  --set persistence.enabled=true \
  --set persistence.size=10Gi
```

#### Настройка ChartMuseum с разными бэкендами

```yaml
# Amazon S3
env:
  open:
    STORAGE: amazon
    STORAGE_AMAZON_BUCKET: my-helm-charts
    STORAGE_AMAZON_REGION: us-east-1

# Google Cloud Storage
env:
  open:
    STORAGE: google
    STORAGE_GOOGLE_BUCKET: my-helm-charts

# Azure Blob Storage
env:
  open:
    STORAGE: microsoft
    STORAGE_MICROSOFT_CONTAINER: my-helm-charts
```

#### Загрузка чартов в ChartMuseum

```bash
# С использованием helm-push плагина
helm plugin install https://github.com/chartmuseum/helm-push.git
helm push mychart-0.1.0.tgz chartmuseum
```

### Использование Harbor для Helm чартов

[Harbor](https://goharbor.io/) - это контейнерный реестр, который поддерживает Helm чарты.

```bash
# Добавление Harbor как репозитория Helm
helm repo add myrepo https://harbor.example.com/chartrepo/myproject

# Загрузка чарта через UI или API
curl -u "admin:password" -X POST \
  -H "Content-Type: multipart/form-data" \
  -F "chart=@mychart-0.1.0.tgz" \
  https://harbor.example.com/api/chartrepo/myproject/charts
```

### Использование Nexus Repository

Nexus OSS/Pro поддерживает хостинг Helm репозиториев.

```bash
# Добавление Nexus как репозитория Helm
helm repo add nexus https://nexus.example.com/repository/helm-hosted/
```

## Подписывание и проверка чартов

### Генерация ключей GPG

```bash
gpg --full-generate-key
```

### Подписание чарта

```bash
helm package --sign --key 'name@example.com' --keyring ~/.gnupg/secring.gpg ./mychart
```

### Проверка подписи чарта

```bash
helm verify mychart-0.1.0.tgz
```

### Установка с проверкой подписи

```bash
helm install --verify myrelease ./mychart-0.1.0.tgz
```

## Управление зависимостями через репозитории

### Определение зависимостей в Chart.yaml

```yaml
# Chart.yaml
dependencies:
  - name: mysql
    version: 8.8.6
    repository: https://charts.bitnami.com/bitnami
  - name: redis
    version: ~12.0.0
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

### Обновление зависимостей

```bash
helm dependency update ./mychart
```

### Сборка зависимостей локально

```bash
helm dependency build ./mychart
```

## Лучшие практики работы с репозиториями

### Организационные практики

1. **Версионирование**: Всегда правильно версионируйте чарты по SemVer.
2. **Документация**: Поддерживайте актуальные README и NOTES.txt.
3. **Структура**: Организуйте репозитории по типам приложений или командам.
4. **CI/CD**: Автоматизируйте обновление репозитория через CI/CD.

### Безопасность

1. **Подписывайте чарты**: Особенно для production-окружений.
2. **Ограничивайте доступ**: Используйте аутентификацию для приватных репозиториев.
3. **Сканируйте уязвимости**: Используйте инструменты типа Trivy для сканирования чартов.
4. **Управление секретами**: Не храните чувствительные данные в чартах.

### Производительность

1. **Кэширование**: Настройте кэширование для часто используемых чартов.
2. **CDN**: Используйте CDN для географически распределенных команд.
3. **Локальные зеркала**: Создавайте локальные зеркала для изолированных сред.

## Инструменты для работы с репозиториями

### Плагины Helm

- **helm-push**: Позволяет загружать чарты в репозиторий.
- **helm-s3**: Добавляет поддержку S3 как бэкенда для Helm.
- **helm-diff**: Показывает diff при обновлении релизов.

```bash
# Установка плагина helm-push
helm plugin install https://github.com/chartmuseum/helm-push.git

# Установка плагина helm-s3
helm plugin install https://github.com/hypnoglow/helm-s3.git

# Установка плагина helm-diff
helm plugin install https://github.com/databus23/helm-diff
```

### Службы для хостинга репозиториев

1. **GitHub/GitLab Pages**: Бесплатный хостинг статических репозиториев.
2. **ChartMuseum**: Серверное приложение для Helm репозиториев.
3. **JFrog Artifactory**: Универсальный менеджер артефактов с поддержкой Helm.
4. **Sonatype Nexus**: Управление репозиториями с поддержкой Helm.
5. **Harbor**: Реестр контейнеров с поддержкой Helm чартов.
6. **OCI Registry**: Хранение чартов в OCI-совместимых реестрах (с Helm 3).

## OCI Registry для Helm чартов

С Helm 3 появилась поддержка OCI (Open Container Initiative) реестров для хранения чартов.

### Использование OCI Registry

```bash
# Вход в реестр
helm registry login registry.example.com

# Сохранение чарта в OCI реестр
helm chart save mychart/ registry.example.com/charts/mychart:0.1.0

# Загрузка чарта в реестр
helm chart push registry.example.com/charts/mychart:0.1.0

# Установка из OCI реестра
helm install myrelease oci://registry.example.com/charts/mychart --version 0.1.0
```

### Преимущества OCI реестров

1. **Единая инфраструктура**: Хранение образов контейнеров и Helm чартов в одном месте.
2. **Стандартизация**: Использование открытого стандарта OCI.
3. **Безопасность**: Интегрированное управление доступом и сканирование уязвимостей.
4. **Версионирование**: Естественная поддержка тегов и дайджестов.

## Шпаргалка по командам

| Операция | Команда |
|----------|---------|
| Добавление репозитория | `helm repo add [name] [url]` |
| Обновление репозиториев | `helm repo update` |
| Поиск чартов | `helm search repo [keyword]` |
| Информация о чарте | `helm show chart [chart]` |
| Значения чарта | `helm show values [chart]` |
| Список репозиториев | `helm repo list` |
| Удаление репозитория | `helm repo remove [name]` |
| Упаковка чарта | `helm package [chart-path]` |
| Создание индекса | `helm repo index [dir]` |
| Управление зависимостями | `helm dependency update [chart]` |

## Заключение

Эффективное управление репозиториями Helm - важный аспект инфраструктуры Kubernetes. Хорошо организованные репозитории повышают продуктивность, обеспечивают стандартизацию и упрощают совместную работу в команде. При выборе подхода к управлению репозиториями учитывайте потребности вашей организации, требования безопасности и масштаб развертывания. 