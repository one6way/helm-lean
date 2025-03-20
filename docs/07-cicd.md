# Интеграция Helm с CI/CD

Автоматизация процессов доставки приложений — ключевой аспект современных DevOps-практик. Helm отлично интегрируется с различными системами непрерывной интеграции и доставки (CI/CD), что позволяет автоматизировать развертывание приложений в Kubernetes.

## Основные концепции интеграции Helm с CI/CD

### Преимущества автоматизации Helm-релизов

1. **Повторяемость**: Устранение человеческих ошибок и обеспечение идентичности процесса развертывания
2. **Скорость**: Ускорение доставки новых версий приложений
3. **Масштабируемость**: Возможность управлять множеством релизов в разных окружениях
4. **Аудит**: Отслеживание истории изменений и автоматическое логирование деплоев
5. **Стандартизация**: Единый процесс развертывания для всех приложений

### Типичный CI/CD-конвейер с Helm

1. **Сборка**: Компиляция кода и создание Docker-образа
2. **Тестирование**: Тестирование образа и кода
3. **Упаковка**: Упаковка Helm-чарта и обновление версии
4. **Публикация**: Публикация образа и чарта в репозитории
5. **Развертывание**: Установка или обновление Helm-релиза в целевом окружении
6. **Проверка**: Тестирование развернутого приложения
7. **Откат**: При необходимости — автоматический или ручной откат к предыдущей версии

## Интеграция с GitLab CI/CD

### Типичный .gitlab-ci.yml

```yaml
stages:
  - build
  - test
  - package
  - deploy-staging
  - deploy-production

variables:
  DOCKER_REGISTRY: registry.example.com
  CHART_REGISTRY: registry.example.com/charts
  APP_VERSION: ${CI_COMMIT_SHA:0:8}
  CHART_VERSION: 0.1.${CI_PIPELINE_IID}

build:
  stage: build
  script:
    - docker build -t ${DOCKER_REGISTRY}/${CI_PROJECT_NAME}:${APP_VERSION} .
    - docker push ${DOCKER_REGISTRY}/${CI_PROJECT_NAME}:${APP_VERSION}

test:
  stage: test
  script:
    - cd helm-chart && helm lint .

package:
  stage: package
  script:
    # Обновление версии образа и чарта
    - sed -i "s/appVersion:.*/appVersion: ${APP_VERSION}/" helm-chart/Chart.yaml
    - sed -i "s/version:.*/version: ${CHART_VERSION}/" helm-chart/Chart.yaml
    - sed -i "s|repository:.*|repository: ${DOCKER_REGISTRY}/${CI_PROJECT_NAME}|" helm-chart/values.yaml
    - sed -i "s|tag:.*|tag: ${APP_VERSION}|" helm-chart/values.yaml
    
    # Упаковка и публикация чарта
    - helm package helm-chart/
    - helm push ${CI_PROJECT_NAME}-${CHART_VERSION}.tgz oci://${CHART_REGISTRY}

deploy-staging:
  stage: deploy-staging
  script:
    - helm upgrade --install ${CI_PROJECT_NAME} oci://${CHART_REGISTRY}/${CI_PROJECT_NAME} --version ${CHART_VERSION} --namespace staging --create-namespace --atomic --timeout 5m
  environment:
    name: staging

deploy-production:
  stage: deploy-production
  script:
    - helm upgrade --install ${CI_PROJECT_NAME} oci://${CHART_REGISTRY}/${CI_PROJECT_NAME} --version ${CHART_VERSION} --namespace production --create-namespace --atomic --timeout 5m
  environment:
    name: production
  when: manual
  only:
    - master
```

### Использование Helm в GitLab CI с Kubernetes integration

```yaml
deploy:
  stage: deploy
  image: 
    name: alpine/helm:3.11.1
    entrypoint: [""]
  script:
    - helm upgrade --install myapp ./helm-chart 
      --namespace $KUBE_NAMESPACE 
      --set image.tag=$CI_COMMIT_SHORT_SHA 
      --atomic 
      --wait
  environment:
    name: development
    kubernetes:
      namespace: myapp-dev
```

### Использование GitLab для хранения Helm чартов

GitLab Package Registry поддерживает хранение Helm чартов:

```yaml
package:
  stage: package
  image: alpine/helm:3.11.1
  script:
    # Аутентификация для доступа к Container Registry
    - echo "${CI_REGISTRY_PASSWORD}" | helm registry login -u "${CI_REGISTRY_USER}" --password-stdin "${CI_REGISTRY}"
    # Упаковка и публикация
    - helm package ./helm-chart --app-version ${CI_COMMIT_SHORT_SHA} --version ${CI_PIPELINE_IID}
    - helm push ./myapp-${CI_PIPELINE_IID}.tgz oci://${CI_REGISTRY}/helm-charts
```

## Интеграция с GitHub Actions

### Пример workflow для GitHub Actions

```yaml
name: Deploy with Helm

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v2
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.sha }}
            ghcr.io/${{ github.repository }}:latest
            
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Set up Kubernetes config
        uses: azure/k8s-set-context@v1
        with:
          kubeconfig: ${{ secrets.KUBECONFIG }}
      
      - name: Install Helm
        uses: azure/setup-helm@v1
        with:
          version: '3.8.0'
          
      - name: Update Helm chart version
        run: |
          sed -i "s/appVersion:.*/appVersion: ${GITHUB_SHA::8}/" ./chart/Chart.yaml
          
      - name: Deploy with Helm
        run: |
          helm upgrade --install myapp ./chart \
            --namespace myapp \
            --create-namespace \
            --set image.repository=ghcr.io/${{ github.repository }} \
            --set image.tag=${{ github.sha }} \
            --atomic \
            --wait
```

### Использование GitHub Container Registry для Helm чартов

```yaml
- name: Log in to GitHub Container Registry
  run: echo "${{ secrets.GITHUB_TOKEN }}" | helm registry login ghcr.io -u ${{ github.actor }} --password-stdin

- name: Package Helm chart
  run: helm package ./chart --destination ./dist

- name: Push Helm chart to GitHub Container Registry
  run: |
    helm push ./dist/myapp-*.tgz oci://ghcr.io/${{ github.repository }}/charts
```

## Интеграция с Jenkins

### Пример Jenkinsfile

```groovy
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: helm
    image: alpine/helm:3.11.1
    command:
    - cat
    tty: true
  - name: docker
    image: docker:20.10.12-dind
    command:
    - cat
    tty: true
    volumeMounts:
    - name: docker-socket
      mountPath: /var/run/docker.sock
  volumes:
  - name: docker-socket
    hostPath:
      path: /var/run/docker.sock
      type: Socket
"""
        }
    }
    
    environment {
        DOCKER_REGISTRY = 'registry.example.com'
        CHART_VERSION = "1.0.${BUILD_NUMBER}"
        DOCKER_TAG = "${env.GIT_COMMIT.substring(0, 8)}"
    }
    
    stages {
        stage('Build') {
            steps {
                container('docker') {
                    sh """
                    docker build -t ${DOCKER_REGISTRY}/myapp:${DOCKER_TAG} .
                    docker push ${DOCKER_REGISTRY}/myapp:${DOCKER_TAG}
                    """
                }
            }
        }
        
        stage('Helm Lint') {
            steps {
                container('helm') {
                    sh 'helm lint ./chart'
                }
            }
        }
        
        stage('Package Chart') {
            steps {
                container('helm') {
                    sh """
                    sed -i "s/appVersion:.*/appVersion: ${DOCKER_TAG}/" ./chart/Chart.yaml
                    sed -i "s/version:.*/version: ${CHART_VERSION}/" ./chart/Chart.yaml
                    helm package ./chart
                    """
                }
            }
        }
        
        stage('Deploy to Staging') {
            steps {
                container('helm') {
                    sh """
                    helm upgrade --install myapp ./chart \
                      --namespace staging \
                      --set image.repository=${DOCKER_REGISTRY}/myapp \
                      --set image.tag=${DOCKER_TAG} \
                      --atomic \
                      --timeout 5m
                    """
                }
            }
        }
        
        stage('Deploy to Production') {
            when { branch 'main' }
            input {
                message "Deploy to production?"
                ok "Yes, deploy it!"
            }
            steps {
                container('helm') {
                    sh """
                    helm upgrade --install myapp ./chart \
                      --namespace production \
                      --set image.repository=${DOCKER_REGISTRY}/myapp \
                      --set image.tag=${DOCKER_TAG} \
                      --atomic \
                      --timeout 5m
                    """
                }
            }
        }
    }
}
```

### Использование Jenkins X

Jenkins X — это облачная CD-платформа с тесной интеграцией Kubernetes и Helm:

```yaml
# jenkins-x.yml
buildPack: charts
pipelineConfig:
  pipelines:
    release:
      pipeline:
        stages:
        - name: build-helm-chart
          steps:
          - sh: jx step helm build
        - name: deploy-to-staging
          steps:
          - sh: jx step helm release
            environment:
              - name: DEPLOY_NAMESPACE
                value: jx-staging
        - name: promote-to-production
          steps:
          - sh: jx promote -b --all-auto --timeout 1h --env production
```

## Интеграция с ArgoCD

### Пример использования Helm в ArgoCD

ArgoCD — инструмент GitOps для декларативного управления Kubernetes-ресурсами.

```yaml
# Application CRD для ArgoCD
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myrepo.git
    targetRevision: HEAD
    path: helm-chart
    helm:
      valueFiles:
        - values.yaml
        - values-prod.yaml
      parameters:
        - name: image.tag
          value: v1.2.3
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Интеграция ArgoCD с другими CI/CD системами

```yaml
# GitLab CI для обновления ArgoCD приложения
deploy:
  stage: deploy
  script:
    # Обновление значений в helm-chart/values.yaml
    - sed -i "s/tag:.*/tag: ${CI_COMMIT_SHORT_SHA}/" helm-chart/values.yaml
    - git config --global user.email "ci@example.com"
    - git config --global user.name "GitLab CI"
    - git add helm-chart/values.yaml
    - git commit -m "Update image tag to ${CI_COMMIT_SHORT_SHA}" || true
    - git push origin HEAD:main
    # ArgoCD автоматически синхронизирует изменения
```

## Интеграция с Flux CD

### Helm Controller в Flux CD

Flux CD — еще один инструмент GitOps с поддержкой Helm.

```yaml
# Пример HelmRelease CRD для Flux
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: myapp
  namespace: myapp
spec:
  interval: 5m
  chart:
    spec:
      chart: ./chart
      sourceRef:
        kind: GitRepository
        name: myapp-repo
        namespace: flux-system
  values:
    replicaCount: 2
    image:
      repository: ghcr.io/myorg/myapp
      tag: latest
  install:
    remediation:
      retries: 3
  upgrade:
    remediation:
      remediateLastFailure: true
    cleanupOnFail: true
  test:
    enable: true
```

### Использование Flux в CI/CD-конвейере

```yaml
# GitHub Actions для обновления Flux HelmRelease
name: Update Helm Release

on:
  push:
    branches: [main]
    paths:
      - 'src/**'
      - 'Dockerfile'

jobs:
  build-and-update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
        with:
          fetch-depth: 0
      
      - name: Build and push image
        uses: docker/build-push-action@v2
        with:
          context: .
          push: true
          tags: ghcr.io/myorg/myapp:${{ github.sha }}
      
      - name: Update HelmRelease
        run: |
          git config --global user.email "actions@github.com"
          git config --global user.name "GitHub Actions"
          
          # Обновление тега образа в HelmRelease
          sed -i "s|tag:.*|tag: ${{ github.sha }}|" k8s/helmrelease.yaml
          
          git add k8s/helmrelease.yaml
          git commit -m "Update image tag to ${{ github.sha }}"
          git push
```

## Продвинутые сценарии CI/CD с Helm

### Развертывание между несколькими кластерами

```yaml
# GitLab CI для развертывания в несколько кластеров
deploy:
  stage: deploy
  script:
    - |
      for cluster in us-east eu-west asia-east; do
        echo "Deploying to $cluster cluster"
        export KUBECONFIG=./kubeconfig-$cluster.yaml
        
        helm upgrade --install myapp ./chart \
          --namespace myapp \
          --set image.tag=${CI_COMMIT_SHORT_SHA} \
          --set cluster.region=$cluster \
          -f values-$cluster.yaml \
          --atomic
      done
```

### Автоматическое тестирование после развертывания

```yaml
# GitHub Actions с тестированием после развертывания
deploy-and-test:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v2
    
    - name: Deploy with Helm
      run: |
        helm upgrade --install myapp ./chart \
          --namespace myapp \
          --set image.tag=${{ github.sha }} \
          --wait
          
    - name: Run integration tests
      run: |
        # Ожидание доступности сервиса
        kubectl -n myapp wait --for=condition=available deployment/myapp --timeout=5m
        
        # Запуск тестов
        ./integration-tests.sh https://myapp.example.com
```

### Автоматический откат при неудаче

```yaml
# Jenkins pipeline с автоматическим откатом
pipeline {
    agent any
    
    stages {
        stage('Deploy') {
            steps {
                script {
                    try {
                        sh """
                        helm upgrade --install myapp ./chart \
                          --namespace myapp \
                          --set image.tag=${GIT_COMMIT} \
                          --atomic \
                          --timeout 5m
                        """
                    } catch (Exception e) {
                        echo "Deployment failed, rollback automatically triggered by --atomic flag"
                        currentBuild.result = 'FAILURE'
                        error("Deployment failed: ${e.message}")
                    }
                }
            }
        }
        
        stage('Verify') {
            steps {
                script {
                    try {
                        sh './verify-deployment.sh'
                    } catch (Exception e) {
                        echo "Verification failed, performing manual rollback"
                        sh "helm rollback myapp 0 --namespace myapp"
                        currentBuild.result = 'FAILURE'
                        error("Verification failed: ${e.message}")
                    }
                }
            }
        }
    }
}
```

## Безопасность в CI/CD с Helm

### Хранение секретных данных

#### Использование секретов CI/CD системы

```yaml
# GitLab CI с безопасным хранением секретов
deploy:
  stage: deploy
  script:
    - helm upgrade --install myapp ./chart \
        --namespace myapp \
        --set database.password=${DB_PASSWORD} \
        --set apiKey=${API_KEY}
  variables:
    DB_PASSWORD: ${DB_PASSWORD}  # Переменная, определенная в GitLab CI/CD Variables
    API_KEY: ${API_KEY}          # Переменная, определенная в GitLab CI/CD Variables
```

#### Интеграция с системами управления секретами

```yaml
# Интеграция с HashiCorp Vault в Jenkins
pipeline {
    agent any
    
    stages {
        stage('Deploy') {
            steps {
                withVault(
                    configuration: [
                        vaultUrl: 'https://vault.example.com',
                        vaultCredentialId: 'vault-approle'
                    ],
                    vaultSecrets: [
                        [path: 'secret/myapp/database', secretValues: [
                            [envVar: 'DB_PASSWORD', vaultKey: 'password']
                        ]]
                    ]
                ) {
                    sh """
                    helm upgrade --install myapp ./chart \
                      --namespace myapp \
                      --set database.password=${DB_PASSWORD}
                    """
                }
            }
        }
    }
}
```

#### Использование external-secrets и sealed-secrets

```yaml
# Использование SealedSecrets в Helm в CI/CD
deploy:
  stage: deploy
  script:
    # Генерация SealedSecret
    - kubectl create secret generic myapp-secret --dry-run=client -o yaml \
        --from-literal=password=${DB_PASSWORD} \
        --namespace myapp > mysecret.yaml
    - kubeseal --controller-name=sealed-secrets -o yaml < mysecret.yaml > sealed-secret.yaml
    
    # Развертывание с SealedSecret
    - kubectl apply -f sealed-secret.yaml
    - helm upgrade --install myapp ./chart \
        --namespace myapp \
        --set useExistingSecret=true \
        --set existingSecretName=myapp-secret
```

### Сканирование безопасности Helm чартов

#### Интеграция с Trivy

```yaml
# GitHub Actions с проверкой чарта на уязвимости
name: Helm Security Scan

on:
  push:
    paths:
      - 'chart/**'

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Install Helm
        uses: azure/setup-helm@v1
        
      - name: Install Trivy
        run: |
          wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
          echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
          sudo apt-get update
          sudo apt-get install -y trivy
          
      - name: Lint Helm Chart
        run: helm lint ./chart
        
      - name: Template Helm Chart
        run: |
          helm template ./chart > templated-manifests.yaml
          
      - name: Scan Kubernetes manifests with Trivy
        run: |
          trivy config templated-manifests.yaml
```

#### Проверка политик с помощью OPA/Conftest

```yaml
# GitLab CI с проверкой politices
stages:
  - test
  - deploy

test:
  stage: test
  image: instrumenta/conftest:latest
  script:
    - helm template ./chart > templated.yaml
    - conftest test templated.yaml -p ./policies

deploy:
  stage: deploy
  only:
    - master
  script:
    - helm upgrade --install myapp ./chart --namespace myapp
```

## Мониторинг и наблюдаемость в CI/CD с Helm

### Интеграция с Prometheus и Grafana

```yaml
# Развертывание с включенным мониторингом
deploy:
  stage: deploy
  script:
    - |
      helm upgrade --install myapp ./chart \
        --namespace myapp \
        --set monitoring.enabled=true \
        --set monitoring.serviceMonitor.enabled=true \
        --set monitoring.prometheusRule.enabled=true
        
      # Проверка создания ServiceMonitor
      kubectl get servicemonitor -n myapp myapp -o jsonpath='{.spec.endpoints[0].path}'
```

### Интеграция с Slack/Teams/Discord

```yaml
# GitHub Actions с оповещениями в Slack
deploy:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v2
    
    - name: Deploy with Helm
      id: deploy
      run: |
        helm upgrade --install myapp ./chart --namespace myapp
      continue-on-error: true
      
    - name: Notify Slack - Success
      if: steps.deploy.outcome == 'success'
      uses: rtCamp/action-slack-notify@v2
      env:
        SLACK_CHANNEL: deployments
        SLACK_COLOR: good
        SLACK_MESSAGE: 'Successfully deployed myapp :rocket:'
        SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
        
    - name: Notify Slack - Failure
      if: steps.deploy.outcome == 'failure'
      uses: rtCamp/action-slack-notify@v2
      env:
        SLACK_CHANNEL: deployments
        SLACK_COLOR: danger
        SLACK_MESSAGE: 'Failed to deploy myapp :x:'
        SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
```

## Шпаргалка по командам Helm в CI/CD

| Операция | Команда |
|----------|---------|
| Установка и обновление | `helm upgrade --install <release> <chart> -f values.yaml` |
| Проверка без установки | `helm template <chart> -f values.yaml` |
| Сборка зависимостей | `helm dependency update <chart>` |
| Упаковка чарта | `helm package <chart>` |
| Публикация чарта | `helm push <chart> <repo>` |
| Проверка синтаксиса | `helm lint <chart>` |
| Тесты после установки | `helm test <release>` |
| Симуляция обновления | `helm diff upgrade <release> <chart>` |
| Получение версии релиза | `helm ls -q <release>` |
| Отладка в CI/CD | `helm --debug --dry-run upgrade <release> <chart>` |

## Заключение

Интеграция Helm с CI/CD системами значительно упрощает автоматизацию развертывания и управления приложениями в Kubernetes. С помощью модульности Helm и гибкости современных CI/CD-инструментов можно создавать эффективные и безопасные конвейеры доставки для любых типов приложений. Используйте GitOps-подход, инструменты сканирования безопасности и автоматическое тестирование, чтобы повысить качество и надежность процесса развертывания. 