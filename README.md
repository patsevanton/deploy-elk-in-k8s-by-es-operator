# Разворачиваем Elasticsearch в Kubernetes используя operator и FluxCD

## Введение

Elasticsearch — это распределённая поисковая и аналитическая система для обработки больших объёмов данных в реальном времени.
В Kubernetes его развёртывание обычно осуществляется через оператор ECK (Elastic Cloud on Kubernetes), который упрощает управление жизненным циклом кластера.
Для эффективного мониторинга работы Elasticsearch используется связка Prometheus и специального экспортёра метрик.
Экспортёр собирает ключевые показатели (индексация, поисковые запросы, использование ресурсов) и делает их доступными для Prometheus через ServiceMonitor.
Конфигурация включает настройку аутентификации, SSL-сертификатов для безопасного соединения и томов для хранения данных.
Оператор ECK автоматизирует эти процессы, обеспечивая отказоустойчивый кластер с возможностью горизонтального масштабирования.


## Подготовка окружения

Для развертывания Elasticsearch через ECK с помощью FluxCD требуется предварительно подготовленный Kubernetes-кластер с установленным и настроенным FluxCD, 
подключенным к Git-репозиторию, содержащему манифесты развертывания. 
После добавления соответствующих конфигурационных файлов в отслеживаемый репозиторий FluxCD автоматически выполнит установку оператора ECK 
и создание Elasticsearch-кластера согласно заданным параметрам, обеспечивая последующую синхронизацию при любых изменениях конфигурации.

## Создание Namespace

Создадим `Namespace` для размещения оператора ECK:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: es-operator
```

Этот `Namespace` будет использоваться для развертывания всех связанных с ECK ресурсов.

## Настройка GitRepository

Добавим в FluxCD репозиторий Git, содержащий манифесты для развертывания ECK.
Этот манифест указывает FluxCD клонировать репозиторий ECK с тегом `v2.10.0`, но игнорировать все файлы, кроме директории `deploy/eck-operator`.

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: eck
  namespace: es-operator
spec:
  interval: 24h
  ref:
    tag: v2.10.0
  url: https://github.com/elastic/cloud-on-k8s
  ignore: |
    # exclude all
    /*
    # include this path
    !/deploy/eck-operator
```

## Установка es-operator
Этот код развертывает оператор ECK (Elastic Cloud on Kubernetes) через Helm-релиз, используя чарт из Git-репозитория. 
Он настраивает оператор для управления Elasticsearch-кластерами в указанном namespace (myelasticsearch), 
использует образ версии 2.10.0 и проверяет обновления каждые 5 минут.

```yaml
# Определение Helm-релиза для развертывания оператора ECK (Elastic Cloud on Kubernetes)
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: eck  # Название ресурса HelmRelease в Kubernetes
  namespace: es-operator  # Namespace, где будет развернут оператор ECK
spec:
  releaseName: es-operator  # Имя самого Helm-релиза
  chart:
    spec:
      chart: ./deploy/eck-operator  # Путь к чарту внутри Git-репозитория
      sourceRef:
        kind: GitRepository  # Источник чарта - Git-репозиторий
        name: eck  # Имя ресурса GitRepository
        namespace: es-operator  # Namespace GitRepository
  interval: 5m0s  # Интервал проверки обновлений (каждые 5 минут)
  # Настройки установки CRD (Custom Resource Definitions)
  install:
    crds: Create  # Стратегия создания CRD при установке
  upgrade:
    crds: CreateReplace  # Стратегия обновления CRD (создать или заменить)
  # Кастомные значения для чарта
  values:
    managedNamespaces:
      - myelasticsearch  # Namespace, где оператор будет управлять ресурсами Elasticsearch
    # Настройки образа оператора
    image:
      repository: elastic/eck-operator  # Репозиторий с образом
      pullPolicy: IfNotPresent  # Политика загрузки образа
      tag: 2.10.0  # Версия оператора
    replicaCount: 1  # Количество реплик оператора
```


# Управление секретами Elasticsearch с помощью External Secrets Operator

## Введение
В современных DevOps-практиках управление секретами является критически важной задачей.
External Secrets Operator (ESO) упрощает этот процесс, интегрируясь с внешними хранилищами секретов, такими как HashiCorp Vault, AWS Secrets Manager и другие. 
В этой статье мы рассмотрим, как настроить ESO для управления учетными данными Elasticsearch.

## Установка External Secrets Operator
Прежде чем приступить к настройке секретов, необходимо установить External Secrets Operator в Kubernetes-кластере с использованием FluxCD.

```sh
---
# Определение Helm-репозитория для External Secrets Operator
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: external-secrets  # Имя репозитория, которое будет использоваться для ссылок
spec:
  interval: 24h  # Интервал проверки обновлений в репозитории (1 раз в сутки)
  url: https://charts.external-secrets.io  # URL официального репозитория External Secrets

---
# Определение Helm-релиза для установки External Secrets Operator
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: external-secrets  # Имя ресурса HelmRelease
spec:
  releaseName: external-secrets  # Имя самого релиза в Helm
  interval: 5m  # Интервал проверки состояния релиза (каждые 5 минут)
  chart:
    spec:
      chart: external-secrets  # Название чарта из репозитория
      version: 0.9.20  # Конкретная версия чарта для установки
      sourceRef:
        kind: HelmRepository  # Тип источника - ранее определенный Helm-репозиторий
        name: external-secrets  # Имя репозитория, откуда брать чарт
  install:
    crds: CreateReplace  # Стратегия установки CRD: создание или замена при установке
    createNamespace: true  # Автоматическое создание namespace при установке
  upgrade:
    crds: CreateReplace  # Стратегия обновления CRD: создание или замена при обновлении
```

После установки необходимо убедиться, что оператор успешно запущен:

```sh
kubectl get pods -n external-secrets
```

## Настройка хранения секретов
В данном примере используется HashiCorp Vault в качестве хранилища секретов. Предполагается, что Vault уже настроен и доступен.

### Создание `ClusterSecretStore`
Этот код настраивает интеграцию Kubernetes с HashiCorp Vault через External Secrets Operator.
Секрет (vault-secret) – хранит конфиденциальный secret-id, необходимый для аутентификации в Vault через механизм AppRole.
ClusterSecretStore (vault-backend) – определяет подключение к Vault (адрес, путь, версия KV-движка), аутентификацию через AppRole (с использованием roleId и secret-id из секрета) и TLS-сертификат для безопасного соединения.
Таким образом, External Secrets Operator сможет безопасно получать секреты из Vault и передавать их в Kubernetes.

```yaml
---
# Секрет Kubernetes, содержащий sensitive данные для аутентификации в Vault
apiVersion: v1
kind: Secret
metadata:
  name: vault-secret  # Имя секрета, который будет использоваться для аутентификации
type: Opaque  # Тип секрета - непрозрачные данные
stringData:
  secret-id: "secret-id-of-roleId"  # Секретный ID для AppRole аутентификации в Vault
  # Это чувствительные данные, которые хранятся в секрете

---
# Конфигурация ClusterSecretStore для External Secrets Operator
# Определяет как подключиться к Vault и получать секреты
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend  # Имена хранилища секретов, доступного на уровне кластера
spec:
  provider:
    vault:  # Провайдер - HashiCorp Vault
      server: "https://vault.example.com"  # URL сервера Vault
      path: "secret"  # Путь к бэкенду секретов в Vault
      version: v2  # Версия KV (Key-Value) бэкенда (v1 или v2)
      caBundle: "base64 encoded string of certificate"  # CA сертификат для проверки TLS соединения
      auth:
        appRole:  # Метод аутентификации - AppRole
          path: path-secret  # Путь, по которому находится AppRole в Vault
          roleId: "roleId"  # Role ID для аутентификации
          secretRef:  # Ссылка на Kubernetes Secret, содержащий Secret ID
            key: secret-id  # Ключ в секрете, содержащий Secret ID
            name: vault-secret  # Имя секрета, определенного выше
```

## Определение секретов для Elasticsearch
Ниже приведен YAML-файл `eso-auth.yaml`, который создаёт и обновляет Kubernetes-секрет для доступа к Elasticsearch, подтягивая логин и пароль из Vault каждую минуту. 
В итоге получается секрет с типом basic-auth, куда дополнительно прописываются дополнительные роли, например kibana_admin и superuser. 
Всё работает через External Secrets Operator, который берёт данные из указанного пути в Vault и упаковывает их в секрет.

```yaml
---
# ExternalSecret - ресурс для синхронизации секретов из внешнего хранилища в Kubernetes
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: es-admin  # Название ExternalSecret ресурса
  namespace: myelasticsearch  # Namespace, куда будет создан секрет

spec:
  # Ссылка на SecretStore, где определено подключение к Vault
  secretStoreRef:
    name: vault-backend  # Имя ClusterSecretStore, определенного ранее
    kind: ClusterSecretStore  # Тип хранилища (кластерный)

  refreshInterval: "1m"  # Интервал обновления секрета (каждую минуту)

  # Настройки конечного Kubernetes Secret
  target:
    name: es-admin  # Имя создаваемого секрета в Kubernetes

    # Шаблон для генерации секрета
    template:
      engineVersion: v2  # Версия шаблонизатора
      type: kubernetes.io/basic-auth  # Тип секрета - для базовой аутентификации

      # Данные, которые будут в секрете
      data:
        username: "{{ .username }}"  # Шаблон для имени пользователя
        password: "{{ .password }}"  # Шаблон для пароля
        roles: kibana_admin,superuser  # Статические роли (не из Vault)

  # Определение данных, которые нужно получить из Vault
  data:
    - secretKey: username  # Ключ в целевом секрете
      remoteRef:
        key: ycloud/elasticsearch/myelasticsearch  # Путь к секрету в Vault
        property: admin_user  # Конкретное свойство в секрете Vault

    - secretKey: password  # Ключ в целевом секрете
      remoteRef:
        key: ycloud/elasticsearch/myelasticsearch  # Путь к секрету в Vault
        property: admin_password  # Конкретное свойство в секрете Vault
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myelasticsearch-user
  namespace: myelasticsearch
spec:
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  refreshInterval: "1m"
  target:
    name: myelasticsearch-user
    template:
      engineVersion: v2
      type: kubernetes.io/basic-auth
      data:
        username: "{{ .username }}"
        password: "{{ .password }}"
        roles: viewer,myelasticsearch-role
  data:
    - secretKey: username
      remoteRef:
        key: ycloud/elasticsearch/myelasticsearch
        property: myelasticsearch_user
    - secretKey: password
      remoteRef:
        key: ycloud/elasticsearch/myelasticsearch
        property: myelasticsearch_password
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: viewer-user
  namespace: myelasticsearch
spec:
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  refreshInterval: "1m"
  target:
    name: viewer-user
    template:
      engineVersion: v2
      type: kubernetes.io/basic-auth
      data:
        username: "{{ .username }}"
        password: "{{ .password }}"
        roles: viewer-role,kibana_user
  data:
    - secretKey: username
      remoteRef:
        key: ycloud/elasticsearch/myelasticsearch
        property: viewer_user
    - secretKey: password
      remoteRef:
        key: ycloud/elasticsearch/myelasticsearch
        property: viewer_password
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: elastic-backup-myelasticsearch-credentials
  namespace: myelasticsearch
spec:
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  refreshInterval: "1m"
  target:
    name: elastic-backup-myelasticsearch-credentials
  data:
    - secretKey: s3.client.backups.access_key
      remoteRef:
        key: ycloud/elasticsearch/myelasticsearch
        property: backup_s3_accesskey
    - secretKey: s3.client.backups.secret_key
      remoteRef:
        key: ycloud/elasticsearch/myelasticsearch
        property: backup_s3_secretkey
```

## Развертывание и проверка
Проверьте, что секреты успешно созданы:

```sh
kubectl get externalsecrets.external-secrets.io -n myelasticsearch
kubectl get secrets -n myelasticsearch
```

Вы также можете просмотреть конкретный секрет:

```sh
kubectl get secret es-admin -n myelasticsearch -o yaml
```

## Развертывание Elasticsearch и Kibana в Kubernetes с ролевой моделью доступа

### Создание пространства имен
Прежде чем начать, создадим пространство имен `myelasticsearch`:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: myelasticsearch
```

## Настройка ролей для Elasticsearch

Для ограничения прав доступа создадим роли, используя Kubernetes Secrets.

### Роль с полными правами

```yaml
---
kind: Secret
apiVersion: v1
metadata:
  name: myelasticsearch-role
  namespace: myelasticsearch
stringData:
  roles.yml: |-
    myelasticsearch-role:
      cluster: ['monitor']
      indices:
        - names: ['myelasticsearch-*']
          privileges: [ 'all' ]
        - names: ['myelasticsearch']
          privileges: [ 'all' ]
```

### Роль для просмотра данных

```yaml
---
kind: Secret
apiVersion: v1
metadata:
  name: viewer-role
  namespace: myelasticsearch
stringData:
  roles.yml: |-
    viewer-role:
      cluster: ['monitor']
      indices:
        - names: ['*']
          privileges: ['read', 'view_index_metadata', 'monitor']
```

## Установка оператора ECK Custom Resources

ECK Custom Resources упрощает управление Elasticsearch и Kibana в Kubernetes.
ECK Custom Resources позволяет создавать и настраивать индексы, шаблоны, пользователей, роли, пространства в Kibana и управлять бэкапами через snapshot-репозитории. 
Также автоматизирует применение изменений и интегрируется с Kubernetes, что делает администрирование Elastic-ресурсов удобнее.

Для управления Elasticsearch используем HelmRelease:

```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: eck-custom-resources
  namespace: myelasticsearch
spec:
  interval: 24h
  url: https://xco-sk.github.io/eck-custom-resources/

---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: es-cr-operator
  namespace: myelasticsearch
spec:
  chart:
    spec:
      chart: eck-custom-resources-operator
      version: 0.4.4
      sourceRef:
        kind: HelmRepository
        name: eck-custom-resources
  interval: 5m
  releaseName: es-cr-operator
  values:
    image:
      repository: harbor.corp/dockerhub/xcosk/eck-custom-resources
    elasticsearch:
      enabled: true
      url: "http://myelasticsearch-es-http:9200"
      certificate:
        secretName: myelasticsearch-es-http-certs-public
        certificateKey: ca.crt
      authentication:
        usernamePasswordSecret:
          secretName: myelasticsearch-es-elastic-user
          username: elastic
```


## Развертывание Kibana

```yaml
---
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana
  namespace: myelasticsearch
spec:
  version: 8.14.1
  count: 1
  image: harbor.corp/dockerhub/elastic/kibana:8.14.1
  elasticsearchRef:
    name: myelasticsearch
  config:
    server.publicBaseUrl: https://kibana-kb-http:5601
  podTemplate:
    spec:
      nodeSelector:
        role: elasticsearch-master
      tolerations:
        - key: role
          operator: Equal
          value: elasticsearch-master
          effect: NoSchedule
      containers:
        - name: kibana
          resources:
            requests:
              memory: 500Mi
            limits:
              memory: 1Gi
```


# Настройка Ingress для Kibana в Kubernetes

В этой статье рассмотрим, как настроить Ingress для Kibana в кластере Kubernetes с использованием Nginx и TLS.

## Введение

Ingress-контроллер позволяет управлять входящими HTTP/HTTPS-запросами в кластер Kubernetes. 
В данном примере мы используем **nginx-ingress** и **cert-manager** для автоматического управления сертификатами.

## Конфигурация Ingress

Приведённый ниже манифест определяет Ingress-ресурс для Kibana.

### YAML-файл:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: cluster-issuer
    # Сервис kibana-kb-http всегда слушает HTTPS, на HTTP стоит 302 редирект на HTTPS
    nginx.ingress.kubernetes.io/backend-protocol: HTTPS
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "false"
    nginx.ingress.kubernetes.io/proxy-body-size: 100m
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "100"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "100"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "100"
    nginx.ingress.kubernetes.io/proxy-buffering: "on"
  name: myelasticsearch-kibana
  namespace: myelasticsearch
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - kibana.myelasticsearch.es.k8s.corp
      secretName: kibana-myelasticsearch-tls
  rules:
    - host: kibana.myelasticsearch.es.k8s.corp
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kibana-kb-http
                port:
                  number: 5601
```

# Развертывание Elasticsearch в Kubernetes


## Конфигурация Elasticsearch

Файл `elasticsearch.yaml` содержит описание ресурсов Kubernetes для развертывания Elasticsearch.

### Основные параметры:
- Версия Elasticsearch: `8.14.1`
- Образ Docker: `harbor.corp/dockerhub/elastic/elasticsearch:8.14.1`
- Аутентификация: `fileRealm` с пользователями `es-admin` и `viewer`
- HTTPS отключен для снижения задержек
- Использование `podDisruptionBudget` для обеспечения отказоустойчивости
- Разделение узлов на:
  - **Master-узлы** (3 шт.)
  - **Data-узлы** (3 шт.)

### Полный YAML-код Elasticsearch

```yaml
---
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: myelasticsearch
  namespace: myelasticsearch
  annotations:
    eck.k8s.elastic.co/downward-node-labels: "topology.kubernetes.io/zone"
spec:
  version: 8.14.1
  image: harbor.corp/dockerhub/elastic/elasticsearch:8.14.1
  secureSettings:
    - secretName: elastic-backup-myelasticsearch-credentials
  auth:
    fileRealm:
      - secretName: es-admin  # Файл с учетными данными администратора
      - secretName: viewer-user  # Файл с учетными данными пользователя с правами просмотра
    roles:
      - secretName: myelasticsearch-role  # Роли для кластера
      - secretName: viewer-role  # Роли для просмотра
  http:
    tls:
      selfSignedCertificate:
        disabled: true  # Отключение самоподписанных сертификатов для упрощенного доступа
  podDisruptionBudget:
    spec:
      minAvailable: 2  # Минимальное количество доступных подов для обеспечения отказоустойчивости
      selector:
        matchLabels:
          elasticsearch.k8s.elastic.co/cluster-name: myelasticsearch
  nodeSets:
    - name: master
      count: 3  # Количество мастер-узлов
      config:
        node.roles: ["master", "remote_cluster_client"]  # Роли мастер-узлов
        xpack.monitoring.collection.enabled: true  # Включение мониторинга
        s3.client.backups.endpoint: storage.yandexcloud.net
        s3.client.backups.region: ru-central1
      podTemplate:
        spec:
          initContainers:
            - name: sysctl
              securityContext:
                privileged: true
                runAsUser: 0
              command: ["sh", "-c", "sysctl -w vm.max_map_count=262144"]  # Настройка параметра памяти
          containers:
            - name: elasticsearch
              resources:
                requests:
                  cpu: 4
                  memory: 8Gi
                limits:
                  cpu: 4
                  memory: 8Gi
    - name: data
      count: 3  # Количество data-узлов
      config:
        node.roles:
          - "data"
          - "ingest"
          - "ml"
          - "transform"
          - "remote_cluster_client"
        s3.client.backups.endpoint: storage.yandexcloud.net
        s3.client.backups.region: ru-central1
        xpack.monitoring.collection.enabled: true
        node.attr.zone: ${ZONE}  # Зона размещения узла
        cluster.routing.allocation.awareness.attributes: k8s_node_name,zone
      podTemplate:
        spec:
          initContainers:
            - name: sysctl
              securityContext:
                privileged: true
                runAsUser: 0
              command: ["sh", "-c", "sysctl -w vm.max_map_count=262144"]
          containers:
            - name: elasticsearch
              env:
                - name: ZONE
                  valueFrom:
                    fieldRef:
                      fieldPath: metadata.annotations['topology.kubernetes.io/zone']
              resources:
                requests:
                  cpu: 4
                  memory: 10Gi  # Запрос ресурсов для узлов типа A
                limits:
                  cpu: 4
                  memory: 10Gi
```

## Настройка Ingress для доступа к Elasticsearch

Для доступа к кластеру Elasticsearch извне используем ресурс `Ingress`:

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: cluster-issuer
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "false"
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
  name: myelasticsearch-elastic
  namespace: myelasticsearch
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myelasticsearch.es.k8s.corp  # Доменное имя для доступа к Elasticsearch
      secretName: myelasticsearch-tls
  rules:
    - host: myelasticsearch.es.k8s.corp
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myelasticsearch-es-http
                port:
                  number: 9200  # Порт для HTTP-доступа к Elasticsearch
```

## Проверка прав индекса myelasticsearch-index:

```shell
curl -k -X PUT -u myelasticsearch-user:пароль \
"https://myelasticsearch.es.k8s.corp/myelasticsearch-index" \
-H "Content-Type: application/json" -d '{}'
```

# Установка и мониторинг Elasticsearch используя Prometheus

## Установка Elasticsearch Exporter
Prometheus Elasticsearch Exporter используется для сбора метрик из Elasticsearch и их передачи в Prometheus.
Чтобы развернуть его в Kubernetes, создадим `HelmRelease` с нужной конфигурацией.

### Файл `exporter.yaml`
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: myelasticsearch-exporter
  namespace: myelasticsearch
spec:
  chart:
    spec:
      chart: prometheus-elasticsearch-exporter
      version: 5.9.0
      sourceRef:
        kind: HelmRepository
        name: prometheus-community
        namespace: monitoring
  interval: 5m
  releaseName: myelasticsearch-exporter
  values:
    env:
      ES_USERNAME: elastic
    extraEnvSecrets:
      ES_PASSWORD:
        secret: myelasticsearch-es-elastic-user
        key: elastic
    es:
      uri: http://myelasticsearch-es-http:9200
      useExistingSecrets: true
      sslSkipVerify: true
    secretMounts:
      - name: elastic-certs # если вы будете подключатся к elasticsearch по HTTPS
        secretName: myelasticsearch-es-http-certs-internal
        path: /ssl
    log:
      format: json
    serviceMonitor:
      enabled: true
```

## Настройка правил мониторинга
Для настройки алертов в Prometheus необходимо создать `PrometheusRule`, который будет отслеживать критические показатели и генерировать предупреждения.

### Файл `prometheus-rules.yaml`
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myelasticsearch
  namespace: myelasticsearch
  labels:
    release: prometheus-operator
spec:
  groups:
    - name: ElasticsearchExporter
      rules:
        - alert: ElasticsearchExporterDown
          expr: up{service=~ ".*elasticsearch.*"} != 1
          for: 1m
          labels:
            severity: warning
          annotations:
            summary: Elasticsearch exporter down!
            description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute"

        - alert: ElasticsearchCpuUsageHigh
          expr: "elasticsearch_process_cpu_percent > 80"
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: Elasticsearch CPU Usage High
            description: "The {{ $labels.cluster }} node {{ $labels.name }} CPU usage is over 80% (value {{ $value }})"

        - alert: ElasticsearchHeapUsageTooHigh
          expr: '(elasticsearch_jvm_memory_used_bytes{area="heap"} / elasticsearch_jvm_memory_max_bytes{area="heap"}) * 100 > 90'
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: Elasticsearch Heap Usage Too High
            description: "The {{ $labels.cluster }} node {{ $labels.name }} heap usage is over 90% (value {{ $value }})"

        - alert: ElasticsearchDiskSpaceLow
          expr: "elasticsearch_filesystem_data_available_bytes / elasticsearch_filesystem_data_size_bytes * 100 < 20"
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: Elasticsearch disk space low
            description: "The {{ $labels.cluster }} node {{ $labels.name }} disk usage is over 80% (value {{ $value }})"

        - alert: ElasticsearchClusterRed
          expr: 'elasticsearch_cluster_health_status{color="red"} == 1'
          for: 0m
          labels:
            severity: warning
          annotations:
            summary: Elasticsearch Cluster Red!
            description: "Elastic Cluster {{ $labels.cluster }} is in Red status!"
```



Название файла: kustomization.yaml
```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - elasticsearch.yaml  # Основной конфигурационный файл Elasticsearch
  - es-cr-operator.yaml  # Файл оператора Elasticsearch
  - eso-auth.yaml  # Настройки аутентификации
  - es-roles.yaml  # Определение ролей
  - exporter.yaml  # Конфигурация экспортера метрик
  - kibana.yaml  # Конфигурация Kibana
  - namespace.yaml  # Определение пространства имен
  - prometheus-rules.yaml  # Настройки правил для Prometheus
```


## Бекап elasticsearch
Название файла: snapshot-policy.yaml
Содержимое файла:
```yaml
apiVersion: es.eck.github.com/v1alpha1
kind: SnapshotLifecyclePolicy
metadata:
  name: myelasticsearch-backup-policy
  namespace: myelasticsearch
spec:
  body: |
    {
      "schedule": "0 */15 * * * ?", 
      "name": "myelasticsearch-snapshot-policy", 
      "repository": "myelasticsearch-backup-repository", 
      "config": { 
        "indices": ["*"], 
        "ignore_unavailable": false,
        "include_global_state": true
      },
      "retention": { 
        "expire_after": "14d", 
        "min_count": 72, 
        "max_count": 336 
      }
    }
```

Название файла: snapshot-repository.yaml
Содержимое файла:
```yaml
apiVersion: es.eck.github.com/v1alpha1
kind: SnapshotRepository
metadata:
  name: backup-repository
  namespace: myelasticsearch
spec:
  body: |
    {
      "type": "s3",
      "settings": {
        "bucket": "elastic-backup",
        "client": "backups"
      }
    }
```

Название файла: kustomization.yaml
Содержимое файла:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - snapshot-policy.yaml
  - snapshot-repository.yaml
```

Теперь ваш кластер готов к использованию!
## Итог
Мы рассмотрели установку и мониторинг Elasticsearch с использованием Prometheus.
Настроенные алерты помогут оперативно реагировать на проблемы с производительностью и доступностью кластера.
Эти файлы можно использовать в Kubernetes-кластере для автоматического развертывания и мониторинга.
## Заключение

Развертывание Elasticsearch в Kubernetes требует детальной настройки параметров хранения, ресурсов и сетевого взаимодействия.
В данной статье был рассмотрен готовый манифест для настройки высокодоступного кластера Elasticsearch с master и data-узлами, а также Ingress для внешнего доступа.
