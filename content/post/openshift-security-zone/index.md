---
title: "Deep dive - OpenShift 叢集節點跨網段架構規劃"
slug:  "openshift"
url: "openshift/security-zones"
date: 2022-09-27T:23:50+08:00
draft: false
description: "本篇文章主要分享，提供在 OpenShift Container Platform 中透過 Route Sharding，以叢集角度為特定目的創建多個 Route，像是在不同的網段上公開不同的 Router，讓內部和外部署用者訪問不同的 Router，將限制使用者於DMZ區域，無法訪問或直接使用其他叢集區域中的資源，直接保護這些節點及組件來提高安全性。"
categories:
  - OpenShift
tags:
  - OpenShift
---

## /概述/ 為什麼有這樣需求？

近期剛好接觸客戶需求，希望能透過「單一叢集OpenSihft 運作於跨網段區域」上架構需求，希望有基於能限制不同角色，訪問流量進入 OpenShift 叢集只允許使用特定功能，包含以下需求：

1. 能限制只有叢集管理人員/開發團隊，能直接存取操作 OpenShift 叢集於內部網路中，因此叢集對應的內部服務、API、儲存等，能獲得更多更安全性。
2. 將提供外部應用程式/服務，隔離於對外 Security Zones 區域，並限制只允許特定的 Router 使用某些特定功能(例如：單純網站服務運行, APP 後端服務)等。

> 主要目的：希望透過 Shard Route 負責，在不同 IP 地址，讓內部和外部使用者訪問不同的 Router 以達到 DMZ 與 OCP 訪問限制，來減少成本下能更加提升叢集運作安全性。

補充 - 更多場景情境，歡迎大家可以參考 Luis Javier Arizmendi Alonso 分享針對 OpenShift 於 Security Zones 系列文章 - [Security Zones in OpenShift worker nodes](https://itnext.io/security-zones-in-openshift-worker-nodes-part-i-introduction-4f85762962d7)

## /探討/  相關架構考量

先探討結論，為什麼不採用「針對 Security Zones 進行實體隔離」來達到企業內部制定相對安全標準和法規，例如直接在 DMZ 區域多部署一座 OpenShift 叢集提供對外服務，更直接保護叢集運行產品的應用及服務更能確保安全性，其實重點包含`成本考量`還有 `OpenShift維護成本`高，可能企業內部需要更多維運人員來管理不同 OpenShift 叢集上版流程、應用程式、API及相關服務配置管理等，都是總總問題。

所以或許另一種選擇，透過有一個大型的 OpenShift「`單一叢集跨網段架構`」能使更容易維運管理及同時達到提高安全性架構，就是此次架構分享重點。

> 這邊還是一個重點，取決於你對於當前環境規劃與架構設計需求而做選擇

以下針對架構圖做細部「單一叢集跨網段架構」探討：

![](https://i.imgur.com/PHuCv7u.png)

- 每座 OpenShift 叢集都應被視為僅在內部管理，而不是暴露於外部 Internet
- 若劃分 Infra Nodes 創建在不同的 Zone 區域中，每個區域中提供使用者運行 Ingress 等應用及 Services 服務的 Pod 
- 將限制使用者無法訪問或直接使用其他叢集區域中的資源，直接保護這些節點及組件。
    - 例如：Master API, Logging, Metrics, Registry 等應該運行於內部 Internet 
- 針對特定 Zone 區域運行應用，如果有特定服務需求，可以使用 node-selectors 將 Pod 實體隔離到特定 Worker 節點
    - 例如：上圖 Pod 跑在特定「藍色節點」上，只能單獨在藍色區域節點運行 Pod


## 單一叢集 OpenSihft 運作於跨網段區域

下圖 - 為-驗證規劃 OpenShift 示意圖：

![](https://i.imgur.com/6GgowkZ.png)


## 操作步驟

1. 需創建對外服務的 Ingress Controller 要如何配置規範
- Infra 節點依據網段區域定義標籤
- DNS 配置對外 External Service 使用的 WildCard
- 創建 Ingress Controller 指派為外部應用使用
- 設置  route 透過標籤指派運行於對外區域的 infra 節點


2. 服務與應用程式部署完成後，對外的 Router 要定義經過哪一個 Ingress 節點去提供服務
- 修改預設 Ingress Controller - default 配置不讓其他 Service 服務進入
- 新增對外服務的 Ingress Controller - service 如何配置


### 1. DNS 配置對外 External Service 使用的 WildCard


| Usage | FQDN | 
| - | - | 
| Kubernetes API | api.<cluster_name>.<base_domain> | 
| Internal API | api-int.<cluster_name>.<base_domain> | 
| Ingress route | \*.apps.<cluster_name>.<base_domain> |
| Application, External Service | \*.service.<cluster_name>.<base_domain> |


### 2. 將 Router 固定到 Infra Node 

- 將 Infra Node 依據服務提供區 Zone 1, Zone 2，指定標籤 (範例配置,  region=8, region=10)
- 在 IngressController 對象的 matchLabels 中指定  Label 設置為對應 Infra Node

```
$ oc label nodes <INFRA_NODE> region=8
$ oc patch ingresscontroller/default -n  openshift-ingress-operator  --type=merge -p '{"spec":{"nodePlacement": {"nodeSelector": {"matchLabels": {"region": "8"}}}}}'


apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: default
  namespace: openshift-ingress-operator
spec:
  replicas: 3
  nodePlacement:
    nodeSelector:
      matchLabels:
        region: "8"


$ oc label nodes <INFRA_NODE> region=10
```

- 檢查節點 label 狀態

```bash
$ oc get node -A --show-labels
NAME                            STATUS   ROLES    AGE   VERSION           LABELS
infra-1.ocp4.lab.example.com    Ready    worker   60d   v1.23.5+3afdacb   region=8, beta.kubernetes.io/arch=amd64,...
infra-2.ocp4.lab.example.com    Ready    worker   60d   v1.23.5+3afdacb   region=8, beta.kubernetes.io/arch=amd64,...
infra-3.ocp4.lab.example.com    Ready    worker   60d   v1.23.5+3afdacb   region=8, beta.kubernetes.io/arch=amd64,...
infra-4.ocp4.lab.example.com    Ready    worker   60d   v1.23.5+3afdacb   region=10, beta.kubernetes.io/arch=amd64,...
infra-5.ocp4.lab.example.com    Ready    worker   60d   v1.23.5+3afdacb   region=10, beta.kubernetes.io/arch=amd64,...
```


### 3. 創建 Ingress Controller 指派為外部應用使用

- 創建命名為 “Service” 的 Ingress Controller(Router) 將允許從外部訪問的應用程式或服務，引流到對應的 Ingress 
- 如下圖 (指定的 Label  region: ‘10’ )為對外使用的網域


```
$ cat ingress-service.yaml 
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: service
  namespace: openshift-ingress-operator
spec:
  replicas: 2  
  domain: service.ocp4.lab.example.com   
  nodePlacement:
    nodeSelector:
      matchLabels:
        region: '10'
  routeSelector:
    matchExpressions:
      - key: type
        operator: In
        values:
          - service
```

> 這邊指定對外 External Service 使用的網域

- 設置 ingress
```
$ oc apply -f ingress-service.yaml
```

### 4. 限制對內對外服務 Ingress Controller 配置

![](https://i.imgur.com/rAARkR1.png)



- 修改預設 Ingress Controller - default 配置不讓其他 Service 服務進入
- 新增對外服務的 Ingress Controller - service 如何配置

> 設定 Ingress Controller - service 指定 service 服務可以進入

```
$ oc edit ingresscontroller service -n openshift-ingress-operator
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: service
  namespace: openshift-ingress-operator
spec:
  replicas: 2
  domain: service.ocp4.lab.example.com 
  nodePlacement:
    nodeSelector:
      matchLabels:
        region: '10'
  routeSelector:
    matchExpressions:
      - key: type
        operator: In
        values:
        - service
```

> 檢查 Ingress Controller - default 限制不讓 service 服務進入

```
$ oc edit ingresscontroller default -n openshift-ingress-operator 
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: service
  namespace: openshift-ingress-operator
spec:
  replicas: 3 
  domain: apps.ocp4.lab.example.com 
  nodePlacement:
    nodeSelector:
      matchLabels:
        region: "8"
  routeSelector:
    matchExpressions:
    - key: type
      operator: NotIn
      values:
      - service
```

### 5. 部署應用，生成 Route 並指定其 Hostname和 Label (type=service)

```
$ oc new-project demo
$ oc new-app rails-postgresql-example
```

```
$ oc get pod -o wide
NAME                                  READY   STATUS      RESTARTS   AGE     IP            NODE                           
postgresql-1-deploy                   0/1     Completed   0          2d20h   10.129.6.16   worker-3.ocp4.lab.example.com 
postgresql-1-hjztg                    1/1     Running     0          2d20h   10.129.4.98   worker-2.ocp4.lab.example.com
rails-postgresql-example-1-build      0/1     Completed   0          2d20h   10.129.4.97   worker-2.ocp4.lab.example.com
rails-postgresql-example-1-deploy     0/1     Completed   0          2d19h   10.129.8.9    infra-1.ocp4.lab.example.com 
rails-postgresql-example-1-f8w4l      1/1     Running     0          2d19h   10.129.3.18   worker-1.ocp4.lab.example.com 
rails-postgresql-example-1-hook-pre   0/1     Completed   0          2d19h   10.129.3.17   worker-1.ocp4.lab.example.com
```
> 在創建 route 的時候需指定 hostname，不可使用預設名稱，此部分跳過說明，可以看到如以下 route domain 為 `.service.ocp4.lab.example.com` 結尾。

```
$ oc get route
NAME                       HOST/PORT                                               PATH   SERVICES            PORT    TERMINATION   WILDCARD
rails-postgresql-example   rails-postgresql-example-demo.service.ocp4.lab.example.com     rails-postgresql-example          <all>      None
```
> #指定 label 讓 rails-postgresql-example 的 route 地址與設置解析
```
$ oc label route rails-postgresql-example type=service
```

### 6. 訪問 AP 的 Route 地址，確認狀態可以訪問

- 預設應用及服務走 default 網頁會讀取不到
```
$ oc label route rails-postgresql-example type- 
```
![](https://i.imgur.com/2sld746.png)

- 增加外部服務對應 label (如範例 type=service)
```
$ oc label route rails-postgresql-example type=service
```


![](https://i.imgur.com/gDe472U.jpg)


## Reference
- [Deep dive of Route Sharding in OpenShift 4](https://rcarrata.com/openshift/ocp4_route_sharding/)
- [Security Zones in OpenShift worker nodes — Part I — Introduction](https://itnext.io/security-zones-in-openshift-worker-nodes-part-i-introduction-4f85762962d7)
- [OpenShift 4 - Ingress、Route與Shard](https://blog.csdn.net/weixin_43902588/article/details/103543191?spm=1001.2014.3001.5502)
