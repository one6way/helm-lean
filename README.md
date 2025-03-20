# Helm: Полное руководство по управлению приложениями в Kubernetes

![Helm Logo](https://helm.sh/img/helm.svg)

## Содержание документации

1. **[Основной файл README.md]** - Введение, обзор Helm и его архитектуры
2. **[01-concepts.md](./docs/01-concepts.md)** - Основные концепции Helm (чарты, релизы, репозитории)
3. **[02-chart-structure.md](./docs/02-chart-structure.md)** - Структура Helm чартов
4. **[03-templates.md](./docs/03-templates.md)** - Система шаблонизации и Go Templates
5. **[04-values.md](./docs/04-values.md)** - Работа с values и переменными
6. **[05-repository.md](./docs/05-repository.md)** - Управление репозиториями Helm
7. **[06-deployment.md](./docs/06-deployment.md)** - Развертывание приложений с Helm
8. **[07-cicd.md](./docs/07-cicd.md)** - Интеграция Helm с CI/CD
9. **[08-advanced.md](./docs/08-advanced.md)** - Продвинутые техники и практики
10. **[09-examples.md](./docs/09-examples.md)** - Практические примеры

## Что такое Helm?

Helm — это пакетный менеджер для Kubernetes, который упрощает установку и управление приложениями в Kubernetes-кластерах. Helm часто называют "apt/yum для Kubernetes", поскольку он выполняет аналогичную роль — управляет жизненным циклом приложений.

### Ключевые особенности Helm:

- **Шаблонизация манифестов**: Использует Go-шаблоны для генерации Kubernetes-манифестов.
- **Версионирование**: Позволяет управлять версиями приложений и откатывать изменения.
- **Чарты**: Helm оперирует чартами — наборами манифестов, которые можно кастомизировать через `values.yaml`.
- **Релизы**: Установка чарта создает релиз, который можно обновлять, откатывать и удалять.
- **Репозитории**: Helm чарты можно хранить в репозиториях и делиться ими с сообществом.

## Архитектура Helm

Helm v3 имеет клиентскую архитектуру, которая напрямую взаимодействует с API Kubernetes.

```
┌───────────────┐           ┌───────────────┐           ┌───────────────┐
│               │           │               │           │               │
│  Helm Client  │ ───────▶  │  Kubernetes   │ ◀───────  │  Chart        │
│  (helm CLI)   │           │  API Server   │           │  Repository   │
│               │           │               │           │               │
└───────────────┘           └───────────────┘           └───────────────┘
                                    ▲
                                    │
                                    ▼
                            ┌───────────────┐
                            │               │
                            │  Kubernetes   │
                            │  Cluster      │
                            │               │
                            └───────────────┘
```

### Компоненты Helm v3:

- **Helm Client**: CLI инструмент, который взаимодействует с API Kubernetes для установки, обновления, удаления и управления чартами.
- **Чарты (Charts)**: Пакеты Helm, содержащие шаблоны манифестов и метаданные.
- **Репозитории (Repositories)**: Хранилища для чартов, откуда их можно устанавливать.
- **Релизы (Releases)**: Экземпляры чартов, запущенных в кластере Kubernetes.

В отличие от Helm v2, версия v3 не использует серверный компонент Tiller, что делает архитектуру более безопасной и соответствующей модели безопасности Kubernetes.

## Области применения Helm

Helm используется для:

- **Автоматизации развёртывания**: Упрощает установку сложных приложений в Kubernetes.
- **Управления версиями**: Позволяет обновлять и откатывать приложения.
- **Интеграции в CI/CD**: Интегрируется с GitLab CI/CD, Jenkins, ArgoCD и другими инструментами.
- **Создания и распространения чартов**: Позволяет создавать и публиковать собственные чарты.
- **Управления зависимостями**: Helm позволяет определять зависимости между чартами.
- **Унификации деплоя**: Стандартизирует процесс развертывания приложений в разных окружениях.

### Жизненный цикл приложения с Helm

```
┌───────────┐     ┌───────────┐     ┌───────────┐     ┌───────────┐
│           │     │           │     │           │     │           │
│  Создание │ ──▶ │ Установка │ ──▶ │Обновление │ ──▶ │  Удаление │
│   чарта   │     │  релиза   │     │  релиза   │     │  релиза   │
│           │     │           │     │           │     │           │
└───────────┘     └───────────┘     └───────────┘     └───────────┘
                        ▲                 │
                        │                 │
                        └─────────────────┘
                             Rollback
```

## Установка Helm

### Установка из репозитория

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

### Установка бинарника с GitHub

```bash
wget https://get.helm.sh/helm-v3.17.1-linux-amd64.tar.gz
tar -xf helm-v3.17.1-linux-amd64.tar.gz
mv linux-amd64/helm /bin
chmod +x /bin/helm
```

### Установка через менеджеры пакетов

#### macOS (Homebrew)
```bash
brew install helm
```

#### Windows (Chocolatey)
```bash
choco install kubernetes-helm
```

#### Windows (Scoop)
```bash
scoop install helm
```

### Настройка автодополнения

```bash
# Для Bash
mkdir -p /etc/bash_completion.d
helm completion bash > /etc/bash_completion.d/helm
source <(helm completion bash)

# Для ZSH
mkdir -p $HOME/.oh-my-zsh/completions
helm completion zsh > $HOME/.oh-my-zsh/completions/_helm

# Для Fish
helm completion fish > ~/.config/fish/completions/helm.fish
```

## Основные команды Helm

### Управление чартами
- **Создание чарта**: `helm create chart_name`
- **Проверка чарта**: `helm lint ./chart_name`
- **Упаковка чарта**: `helm package ./chart_name`

### Установка и управление релизами
- **Установка чарта**: `helm install -f values.yaml $name $folder`
- **Обновление чарта**: `helm upgrade --install --create-namespace -n $namespace -f values.yaml $name $folder`
- **Удаление чарта**: `helm uninstall $name`
- **Просмотр релизов**: `helm list -n $namespace`
- **Просмотр истории релизов**: `helm history $name`
- **Откат релиза**: `helm rollback $name $revision`

### Работа с репозиториями
- **Добавление репозитория**: `helm repo add $reponame $url`
- **Обновление индекса репозиториев**: `helm repo update`
- **Поиск чартов**: `helm search repo $keyword`
- **Список репозиториев**: `helm repo list`

### Шаблонизация и тестирование
- **Рендер манифестов**: `helm template -f values.yaml $name .`
- **Проверка установки**: `helm install --dry-run --debug -f values.yaml $name .`
- **Запуск тестов**: `helm test $name`

## Лучшие практики

1. **Структурируйте values.yaml**: Организуйте значения логическими группами для лучшей читаемости и удобства управления.
2. **Используйте `values.yaml` для кастомизации**: Это позволяет легко адаптировать чарты для разных окружений.
3. **Версионируйте чарты**: Убедитесь, что каждая версия чарта соответствует определённой версии приложения.
4. **Используйте хелперы**: Применяйте встроенные функции Helm для уменьшения дублирования в шаблонах.
5. **Добавляйте комментарии**: Помогайте пользователям понять, как использовать ваш чарт.
6. **Тестируйте чарты**: Используйте helm lint, helm template и helm test для проверки чартов перед публикацией.
7. **Интегрируйте Helm в CI/CD**: Это упрощает автоматическое развёртывание и управление приложениями.
8. **Используйте подчарты (subchart) для компонентов**: Это помогает организовать сложные приложения.

## Ограничения Helm

- **GitOps**: В GitOps-подходе Helm может быть избыточным, так как деплой управляется через Git-репозиторий.
- **Простые приложения**: Для простых приложений использование Helm может быть излишним.
- **Сложность шаблонов**: Go-шаблоны могут быть сложными для новичков.
- **Хранение состояния**: Helm хранит состояние релизов в Secrets кластера, что может быть проблемой при некоторых сценариях.

## Быстрый старт

Чтобы быстро начать работу с Helm:

1. **Установите Helm**: Следуйте инструкциям в разделе "Установка Helm".
2. **Добавьте репозиторий**: `helm repo add bitnami https://charts.bitnami.com/bitnami`
3. **Обновите репозиторий**: `helm repo update`
4. **Установите приложение**: `helm install my-release bitnami/nginx`
5. **Проверьте установку**: `helm list`

## Дополнительные ресурсы

- [Официальная документация Helm](https://helm.sh/docs/)
- [Helm Hub](https://artifacthub.io/) - каталог доступных чартов
- [Сообщество Helm](https://helm.sh/community/)
- [GitHub репозиторий Helm](https://github.com/helm/helm)

Подробные инструкции и примеры можно найти в соответствующих разделах документации в папке `docs/`.
