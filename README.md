Разворачиваем Elastic Cloud в Kubernetes используя FluxCD

# Развертывание Elastic Cloud в Kubernetes (ECK) с использованием FluxCD

## Введение

В этой статье мы рассмотрим, как развернуть Elastic Cloud on Kubernetes (ECK) с использованием FluxCD. 
Мы создадим необходимые ресурсы, включая `Namespace`, `GitRepository` и `HelmRelease`.

## Подготовка окружения

Перед началом убедитесь, что у вас установлен и настроен FluxCD. 
Вам также потребуется кластер Kubernetes и доступ к репозиторию с манифестами.

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

Добавим в FluxCD репозиторий Git, содержащий манифесты для развертывания ECK:

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

Этот манифест указывает FluxCD клонировать репозиторий ECK с тегом `v2.10.0`, но игнорировать все файлы, кроме директории `deploy/eck-operator`.

## Установка HelmRelease

Теперь создадим ресурс `HelmRelease`, который позволит развернуть ECK с помощью Helm:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: eck
  namespace: es-operator
spec:
  releaseName: es-operator
  chart:
    spec:
      chart: ./deploy/eck-operator
      sourceRef:
        kind: GitRepository
        name: eck
        namespace: es-operator
  interval: 5m0s
  install:
    crds: Create
  upgrade:
    crds: CreateReplace
  values:
    managedNamespaces:
      - myelasticsearch # namespace где находится kind: Elasticsearch
    image:
      repository: harbor.corp/dockerhub/elastic/eck-operator
      pullPolicy: IfNotPresent
      tag: 2.10.0
    replicaCount: 1
```

Этот манифест указывает FluxCD установить ECK-оператор из Git-репозитория, а также задаёт параметры образа контейнера, политики обновления CRD и количество реплик.

## Развертывание

Чтобы применить эти манифесты, добавьте их в ваш Git-репозиторий, который отслеживает FluxCD.
После этого FluxCD автоматически синхронизирует и развернет ECK в вашем кластере.

## Заключение

Использование FluxCD для развертывания ECK позволяет автоматизировать управление Elasticsearch в Kubernetes, обеспечивая декларативный подход и контроль версий. 
Следуя этой инструкции, вы сможете развернуть и настроить ECK-оператор в вашем кластере Kubernetes.



# Установка и мониторинг Elasticsearch с Prometheus

## Введение
Elasticsearch — мощная поисковая и аналитическая система, широко используемая для работы с большими объемами данных. 
В данной статье мы рассмотрим процесс установки Elasticsearch и настройки его мониторинга с помощью Prometheus.

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
      uri: https://myelasticsearch-es-http:9200
      useExistingSecrets: true
      sslSkipVerify: true
    secretMounts:
      - name: elastic-certs
        secretName: myelasticsearch-es-http-certs-internal
        path: /ssl
    log:
      format: json
    serviceMonitor:
      enabled: true
    nodeSelector:
      role: elasticsearch-master
    tolerations:
      - key: role
        operator: Equal
        value: elasticsearch-master
        effect: NoSchedule
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

## Итог
Мы рассмотрели установку и мониторинг Elasticsearch с использованием Prometheus.
Настроенные алерты помогут оперативно реагировать на проблемы с производительностью и доступностью кластера.
Эти файлы можно использовать в Kubernetes-кластере для автоматического развертывания и мониторинга.


# Управление секретами Elasticsearch с помощью External Secrets Operator

## Введение
В современных DevOps-практиках управление секретами является критически важной задачей.
External Secrets Operator (ESO) упрощает этот процесс, интегрируясь с внешними хранилищами секретов, такими как HashiCorp Vault, AWS Secrets Manager и другие. 
В этой статье мы рассмотрим, как настроить ESO для управления учетными данными Elasticsearch.

## Установка External Secrets Operator
Прежде чем приступить к настройке секретов, необходимо установить External Secrets Operator в Kubernetes-кластере. Установить ESO можно с помощью Helm:

```sh
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm install external-secrets external-secrets/external-secrets --namespace external-secrets --create-namespace
```

После установки необходимо убедиться, что оператор успешно запущен:

```sh
kubectl get pods -n external-secrets
```

## Настройка хранения секретов
В данном примере используется HashiCorp Vault в качестве хранилища секретов. Предполагается, что Vault уже настроен и доступен.

### Создание `ClusterSecretStore`

Для начала создадим `ClusterSecretStore`, который определяет, откуда ESO будет получать секреты:

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
Ниже приведен YAML-файл `eso-auth.yaml`, который определяет несколько `ExternalSecret` ресурсов для управления учетными данными Elasticsearch.

```yaml
# ES admin
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: es-admin
  namespace: myelasticsearch
spec:
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  refreshInterval: "1m"
  target:
    name: es-admin
    template:
      engineVersion: v2
      type: kubernetes.io/basic-auth
      data:
        username: "{{ .username }}"
        password: "{{ .password }}"
        roles: kibana_admin,superuser
  data:
    - secretKey: username
      remoteRef:
        key: ycloud/elasticsearch/myelasticsearch
        property: admin_user
    - secretKey: password
      remoteRef:
        key: ycloud/elasticsearch/myelasticsearch
        property: admin_password
```

Этот ресурс создаст Kubernetes Secret с учетными данными администратора Elasticsearch. Аналогично можно определить секреты для других пользователей.

### Пользователь `myelasticsearch-user`

```yaml
# ES myelasticsearch user
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
```

### Пользователь `viewer-user`

```yaml
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
```

### Доступ к учетным данным для резервного копирования

```yaml
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
kubectl get secrets -n myelasticsearch
```

Вы также можете просмотреть конкретный секрет:

```sh
kubectl get secret es-admin -n myelasticsearch -o yaml
```

## Заключение
External Secrets Operator значительно упрощает управление секретами в Kubernetes, позволяя безопасно интегрировать внешние хранилища секретов. 
В данной статье мы рассмотрели, как настроить ESO для работы с Elasticsearch, создав учетные данные пользователей и доступы к S3 для резервного копирования. 
Этот подход помогает повысить безопасность и удобство работы с конфиденциальными данными в Kubernetes-кластере.

# Развертывание Elasticsearch и Kibana в Kubernetes с ролевой моделью доступа

## Введение

В этой статье мы рассмотрим, как развернуть Elasticsearch и Kibana в Kubernetes с помощью операторов и настроить роли для управления доступом.

## Подготовка к развертыванию

### Создание пространства имен
Прежде чем начать, создадим пространство имен `myelasticsearch`:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: myelasticsearch
```

Сохраните этот файл как `namespace.yaml` и примените командой:

```sh
kubectl apply -f namespace.yaml
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

Примените файлы:

```sh
kubectl apply -f es-roles.yaml
```

## Установка оператора ECK Custom Resources

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
    nodeSelector:
      role: elasticsearch-master
    tolerations:
      - key: role
        operator: Equal
        value: elasticsearch-master
        effect: NoSchedule
```

Примените файл:

```sh
kubectl apply -f es-cr-operator.yaml
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

Примените файл:

```sh
kubectl apply -f kibana.yaml
```

## Заключение

В этой статье мы рассмотрели процесс развертывания Elasticsearch и Kibana в Kubernetes, настройку операторов и управление ролями доступа. 
Теперь ваш кластер настроен и готов к использованию.


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

## Введение

Elasticsearch — это мощная распределенная поисковая и аналитическая система, которая широко используется для обработки больших объемов данных. 
В данной статье рассмотрим развертывание Elasticsearch в Kubernetes с использованием оператора Elastic Cloud on Kubernetes (ECK).

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
  - **Data-узлы типа A** (3 шт.) с увеличенными ресурсами
  - **Data-узлы типа B** (3 шт.) с меньшими ресурсами

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
    - name: data-a
      count: 3  # Количество data-узлов типа A
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
                  cpu: 10
                  memory: 30Gi  # Запрос ресурсов для узлов типа A
                limits:
                  cpu: 10
                  memory: 30Gi
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

## Заключение

Развертывание Elasticsearch в Kubernetes требует детальной настройки параметров хранения, ресурсов и сетевого взаимодействия.
В данной статье был рассмотрен готовый манифест для настройки высокодоступного кластера Elasticsearch с master и data-узлами, а также Ingress для внешнего доступа.


## Проверка прав индекса myelasticsearch-index:

```shell
curl -k -X PUT -u myelasticsearch-user:пароль \
"https://myelasticsearch.es.k8s.corp/myelasticsearch-index" \
-H "Content-Type: application/json" -d '{}'
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

Название файла: ./kustomization.yaml
Содержимое файла:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - snapshot-policy.yaml
  - snapshot-repository.yaml
```

Теперь ваш кластер готов к использованию!