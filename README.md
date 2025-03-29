Разворачиваем Elastic Cloud on Kubernetes используя FluxCD

Установка elastic operator
```
Название файла: ./namespace.yaml
Содержимое файла:
---
apiVersion: v1
kind: Namespace
metadata:
  name: es-operator
-----------------------------
Название файла: ./eck.yaml
Содержимое файла:
---
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

---
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
-----------------------------
```

Установка elasticsearch 
```
Название файла: ./exporter.yaml
Содержимое файла:
---
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
      sourceRef: # Use exiting prometheus helm repository
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
-----------------------------
Название файла: ./prometheus-rules.yaml
Содержимое файла:
---
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

        - alert: ElasticsearchClusterxDown
          expr: elasticsearch_cluster_health_up != 1
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
            summary: Elasticsearch Cpu Usage High
            description: "The {{ $labels.cluster }} node {{ $labels.name }} cpu usage is over 80% value {{ $value }}"

        - alert: ElasticsearchCpuUsageTooHigh
          expr: "elasticsearch_process_cpu_percent > 90"
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: Elasticsearch Cpu Usage Too High
            description: "The {{ $labels.cluster }} node {{ $labels.name }} cpu usage is over 90% value {{ $value }}"

        - alert: ElasticsearchHeapUsageTooHigh
          expr: '(elasticsearch_jvm_memory_used_bytes{area="heap"} / elasticsearch_jvm_memory_max_bytes{area="heap"}) * 100 > 90'
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: Elasticsearch Heap Usage Too High
            description: "The {{ $labels.cluster }} node {{ $labels.name }} heap usage is over 90% value {{ $value }}"

        - alert: ElasticsearchHeapUsageWarning
          expr: '(elasticsearch_jvm_memory_used_bytes{area="heap"} / elasticsearch_jvm_memory_max_bytes{area="heap"}) * 100 > 80'
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: Elasticsearch Heap Usage warning
            description: "The {{ $labels.cluster }} node {{ $labels.name }} heap usage is over 80% value {{ $value }}"

        - alert: ElasticsearchDiskOutOfSpace
          expr: "elasticsearch_filesystem_data_available_bytes / elasticsearch_filesystem_data_size_bytes * 100 < 10"
          for: 0m
          labels:
            severity: warning
          annotations:
            summary: Elasticsearch disk out of space
            description: "The {{ $labels.cluster }} node {{ $labels.name }} disk usage is over 90% value {{ $value }}"

        - alert: ElasticsearchDiskSpaceLow
          expr: "elasticsearch_filesystem_data_available_bytes / elasticsearch_filesystem_data_size_bytes * 100 < 20"
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: Elasticsearch disk space low
            description: "The {{ $labels.cluster }} node {{ $labels.name }} disk usage is over 80% value {{ $value }}"

        - alert: ElasticsearchClusterRed
          expr: 'elasticsearch_cluster_health_status{color="red"} == 1'
          for: 0m
          labels:
            severity: warning
          annotations:
            summary: Elasticsearch Cluster Red!
            description: "Elastic Cluster {{ $labels.cluster }} is in Red status!"

        - alert: ElasticsearchClusterYellow
          expr: 'elasticsearch_cluster_health_status{color="yellow"} == 1'
          for: 0m
          labels:
            severity: warning
          annotations:
            summary: Elasticsearch Cluster Yellow
            description: "Elastic Cluster {{ $labels.cluster }} is Yellow status"

        - alert: ElasticsearchHealthyNodes
          expr: 'sum by(cluster) (elasticsearch_cluster_health_number_of_nodes) < sum by(cluster)(elasticsearch_nodes_roles{role =~"master|data"})'
          for: 0m
          labels:
            severity: warning
          annotations:
            summary: Elasticsearch Healthy Nodes
            description: "Missing {{ $value }} node in Elasticsearch {{ $labels.cluster }} cluster"

        - alert: ElasticsearchHealthyDataNodes
          expr: 'sum by(cluster) (elasticsearch_cluster_health_number_of_nodes) < sum by(cluster)(elasticsearch_nodes_roles{role =~"data"})'
          for: 0m
          labels:
            severity: warning
          annotations:
            summary: Elasticsearch Healthy Data Nodes
            description: "Elasticsearch {{ $labels.cluster }} is missing {{ $value }} data node in Elasticsearch cluster"

        - alert: ElasticsearchRelocatingShards
          expr: "elasticsearch_cluster_health_relocating_shards > 0"
          for: 0m
          labels:
            severity: info
          annotations:
            summary: Elasticsearch relocating shards
            description: "Elasticsearch {{ $labels.cluster }} is relocating {{ $value }} shards"

        - alert: ElasticsearchRelocatingShardsTooLong
          expr: "elasticsearch_cluster_health_relocating_shards > 0"
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: Elasticsearch relocating shards too long
            description: "Elasticsearch {{ $labels.cluster }} has been relocating {{ $value }} shards for 15min"

        - alert: ElasticsearchInitializingShards
          expr: "elasticsearch_cluster_health_initializing_shards > 0"
          for: 0m
          labels:
            severity: info
          annotations:
            summary: Elasticsearch initializing shards
            description: "Elasticsearch {{ $labels.cluster }} is initializing {{ $value }} shards"

        - alert: ElasticsearchInitializingShardsTooLong
          expr: "elasticsearch_cluster_health_initializing_shards > 0"
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: Elasticsearch initializing shards too long
            description: "Elasticsearch {{ $labels.cluster }} has been initializing {{ $value}} shards for 15 min"

        - alert: ElasticsearchUnassignedShards
          expr: "elasticsearch_cluster_health_unassigned_shards > 0"
          for: 0m
          labels:
            severity: warning
          annotations:
            summary: Elasticsearch unassigned shards
            description: "Elasticsearch {{ $labels.cluster }} has {{ $value }} unassigned shards"

        - alert: ElasticsearchPendingTasks
          expr: "elasticsearch_cluster_health_number_of_pending_tasks > 0"
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: Elasticsearch pending tasks
            description: "Elasticsearch {{ $labels.cluster }} has {{ $value }} pending tasks. Cluster works slowly"
-----------------------------
Название файла: ./eso-auth.yaml
Содержимое файла:
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

---
# ES myelasticsearch user
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
        roles: viewer,myelasticsearch-role # Роли, примененные к учетке
  data:
    - secretKey: username
      remoteRef:
        key: ycloud/elasticsearch/myelasticsearch
        property: search_suggestions_user
    - secretKey: password
      remoteRef:
        key: ycloud/elasticsearch/myelasticsearch
        property: search_suggestions_password


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
-----------------------------
Название файла: ./es-roles.yaml
Содержимое файла:
# Docs: https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-users-and-roles.html
---
kind: Secret
apiVersion: v1
metadata:
  name: myelasticsearch-role
  namespace: myelasticsearch
stringData:
  roles.yml: |-
    myelasticsearch-role: # Название роли
      cluster: ['monitor'] # Права на мониторинг кластера
      indices:
        - names: ['myelasticsearch-*'] # Применяется к индексам myelasticsearch-*
          privileges: [ 'all' ] # Полные права
        - names: ['myelasticsearch'] # Применяется к индексу myelasticsearch
          privileges: [ 'all' ] # Полные права

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
-----------------------------
Название файла: ./namespace.yaml
Содержимое файла:
---
apiVersion: v1
kind: Namespace
metadata:
  name: myelasticsearch
-----------------------------
Название файла: ./es-cr-operator.yaml
Содержимое файла:
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
      # url: "https://myelasticsearch.es.k8s.corp"
      url: "http://myelasticsearch-es-http:9200"
      # Даже если elastic слушает только HTTP, все равно нужно указать иначе es-cr-operator ругается на отсутствие секрета
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
-----------------------------
Название файла: ./kibana.yaml
Содержимое файла:
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
    name: myelasticsearch # name of elastic-cluster
  # Если настройка HTTPS для kibana закомментирована, то используется самоподписанный сертификат
  # Можно указать сертификат от cert-manager, но разницы нет,
  #  так как ходит ingress не проверяет сертификат
  #  http:
  #    tls:
  #      certificate:
  #        secretName: kibana-myelasticsearch-tls
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

---
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
-----------------------------
Название файла: ./elasticsearch.yaml
Содержимое файла:
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
      - secretName: es-admin # Добавляем пользователя es-admin
      - secretName: viewer-user # Добавляем пользователя viewer
    roles:
      - secretName: myelasticsearch-role
      - secretName: viewer-role
  # HTTPS отключен для снижения latency. Latency 10-20 ms критично для поиска.
  # Если HTTPS включить и будет ошибка Failed to create/update es.eck.github.com/v1alpha1/SnapshotRepository myelasticsearch-backup-repository:
  # x509: certificate is valid for myelasticsearch.es.k8s.corp, not myelasticsearch-es-http, то нужно использовать либо серт от cert-manager
  # либо поменять url в es-cr-operator
  http:
    tls:
      selfSignedCertificate:
        disabled: true
  podDisruptionBudget:
    spec:
      minAvailable: 2
      selector:
        matchLabels:
          elasticsearch.k8s.elastic.co/cluster-name: myelasticsearch
  nodeSets:
    - name: master
      count: 3
      config:
        node.roles: ["master", "remote_cluster_client"]
        xpack.monitoring.collection.enabled: true
        s3.client.backups.endpoint: storage.yandexcloud.net
        s3.client.backups.region: ru-central1
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
              resources:
                requests:
                  cpu: 4
                  memory: 8Gi
                limits:
                  cpu: 4
                  memory: 8Gi
          affinity:
            podAntiAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                - labelSelector:
                    matchLabels:
                      elasticsearch.k8s.elastic.co/cluster-name: myelasticsearch
                      elasticsearch.k8s.elastic.co/node-master: "true"
                  topologyKey: kubernetes.io/hostname
          nodeSelector:
            role: elasticsearch-master
          tolerations:
            - key: role
              operator: Equal
              value: elasticsearch-master
              effect: NoSchedule
      volumeClaimTemplates:
        - metadata:
            name: elasticsearch-data
            namespace: elasticsearch
          spec:
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 20Gi
            storageClassName: yc-network-ssd
    - name: data-a
      count: 3
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
        node.attr.zone: ${ZONE}
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
                  memory: 30Gi
                limits:
                  cpu: 10
                  memory: 30Gi
          affinity:
            podAntiAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                - labelSelector:
                    matchLabels:
                      elasticsearch.k8s.elastic.co/cluster-name: myelasticsearch
                      elasticsearch.k8s.elastic.co/node-data: "true"
                  topologyKey: kubernetes.io/hostname
          nodeSelector:
            role: elasticsearch-data-a
          tolerations:
            - key: role
              operator: Equal
              value: elasticsearch-data-a
              effect: NoSchedule
      volumeClaimTemplates:
        - metadata:
            name: elasticsearch-data
          spec:
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 279Gi
            storageClassName: yc-network-ssd-nonreplicated
    - name: data-b
      count: 3
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
        node.attr.zone: ${ZONE}
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
                  cpu: 2
                  memory: 6Gi
                limits:
                  cpu: 2
                  memory: 6Gi
          affinity:
            podAntiAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                - labelSelector:
                    matchLabels:
                      elasticsearch.k8s.elastic.co/cluster-name: myelasticsearch
                      elasticsearch.k8s.elastic.co/node-data: "true"
                  topologyKey: kubernetes.io/hostname
          nodeSelector:
            role: elasticsearch-data-b
          tolerations:
            - key: role
              operator: Equal
              value: elasticsearch-data-b
              effect: NoSchedule
      volumeClaimTemplates:
        - metadata:
            name: elasticsearch-data
          spec:
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 279Gi
            storageClassName: yc-network-ssd-nonreplicated
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: cluster-issuer
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "false"
    # nginx.ingress.kubernetes.io/backend-protocol: HTTPS
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
  name: myelasticsearch-elastic
  namespace: myelasticsearch
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myelasticsearch.es.k8s.corp
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
                  number: 9200
-----------------------------
Название файла: ./README.md
Содержимое файла:
## Проверка прав индекса suggestions:

```shell
curl -k -X PUT -u myelasticsearch-user:пароль \
"https://myelasticsearch.es.k8s.corp/myelasticsearch-index" \
-H "Content-Type: application/json" -d '{}'
```

-----------------------------
Название файла: ./kustomization.yaml
Содержимое файла:
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - elasticsearch.yaml
  - es-cr-operator.yaml
  - eso-auth.yaml
  - es-roles.yaml
  - exporter.yaml
  - kibana.yaml
  - namespace.yaml
  - prometheus-rules.yaml
-----------------------------

```


## Бекап elasticsearch

```
Название файла: ./snapshot-policy.yaml
Содержимое файла:
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
-----------------------------
Название файла: ./snapshot-repository.yaml
Содержимое файла:
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
-----------------------------
Название файла: ./kustomization.yaml
Содержимое файла:
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - snapshot-policy.yaml
  - snapshot-repository.yaml
-----------------------------

```
