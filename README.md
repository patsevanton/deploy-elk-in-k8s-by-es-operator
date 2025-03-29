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
