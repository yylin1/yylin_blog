---
title: "在 ACM 透過 OpenShift-GitOps/ArgoCD 管理應用程式"
slug:  "argocd"
url: "openshift/argocd-and-acm"
date: 2022-06-25T09:40:40+08:00
draft: false
description: "本篇介紹 如何在每個託管叢集中連接 ArgoCD/OpenShift GitOps 部署，並在 ACM 單一管理平台中直接瀏覽、控制和管理所部署的 Argo Applications以及 ApplicationSets 配置實戰"
categories:
  - cloud-service-provider
tags:
  - OpenShift
  - CICD
  - ArgoCD
  - ACM
---

## 概述
從 ACM 2.5 版開始正式 GA 功能，您可以透過 ApplicationSets 來配置 ArgoCD / OpenShift-GitOps ，通過單一管理平台以可擴展的方式管理您的所有 GitOps Applications。

ApplicationSet Controller 是一個 Kubernetes Controller，它增加對 ApplicationSet CustomResourceDefinition (CRD) 的支持。

ApplicationSet Controller 在與 Argo CD 一起安裝時，通過添加額外的功能來支持以叢集管理員為中心的場景來補充它。

ApplicationSet Ccontroller 提供：

- 使用單個 Kubernetes 清單通過 Argo CD 從一個或多個 Git 存儲庫部署多個 Application 的能力

- 改進了對 [monorepos](https://blog.maxkit.com.tw/2017/09/monorepos.html) 的支持：在 Argo CD 的上下文中，monorepo 是在單個 Git 存儲庫中定義的多個 Argo CD Application 資源

我們將介紹如何將 ACM 與 OpenShift GitOps 的 ApplicationSets 連接起來，以便在託管 Cluster 中配置和部署 OpenShift GitOps Application 和 ApplicationSet。


## 環境配置-先決條件

- 我們需要使用 Operator Hub 在 ACM Hub 那座 OpenShift Cluster 中安裝 OpenShift GitOps：

```
$ until oc apply -k https://github.com/RedHat-EMEA-SSA-Team/ns-gitops/tree/bootstrap/bootstrap ; do sleep 2; done
```

- 參考 [OpenShift GitOps 的官方文件說明](https://docs.openshift.com/container-platform/4.9/cicd/gitops/installing-openshift-gitops.html)

- 另一方面，我們需要配置管理不同的 Cluster (e.g. Public Cloud)。在我的例子中，我使用我環境中部署 2 座 OCP Cluster 叢集，並將在這篇文章中用於部署我的 Application。

## 於 OpenShift GitOps / ArgoCD 配置託管叢集

要在 ACM 中配置和鏈接 OpenShift GitOps，我們可以將一組一個或多個託管 Cluster 註冊到 Argo CD 或 OpenShift GitOps Operator 的實例。

註冊後，我們可以使用從 ACM Hub Application Controller 的 Application 和 ApplicationSets 將我們需要部署的應用程式部署到這些叢集。然後，我們可以設置一個連續的 GitOps 環境，以在開發、暫存和生產環境中跨叢集自動化配置 Application 的一致性。

- 首先，我們需要創建 cluster sets 並將託管 clusters 添加到這些 cluster sets：

```
$ cat acmgitops/managedclusterset.yaml

apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterSet
metadata:
  name: all-openshift-clusters
  spec: {}
```

- 將託管叢集作為導入 Cluster 添加到 ClusterSet。您可以使用 [ACM Console](https://github.com/open-cluster-management/rhacm-docs/blob/2.4_stage/clusters/managedclustersets.adoc#creating-a-managedclustersetbinding-by-using-the-console) 或 [CLI](https://github.com/open-cluster-management/rhacm-docs/blob/2.4_stage/clusters/managedclustersets.adoc#adding-clusters-to-a-managedclusterset-by-using-the-command-line) 導入：

- 創建託管 Cluster 綁定到部署 Argo CD 或 OpenShift GitOps 的 namespace

```
$ cat managedclustersetbinding.yaml

apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterSetBinding
metadata:
  name: all-openshift-clusters
  namespace: openshift-gitops
spec:
  clusterSet: all-openshift-clusters

$ oc apply -f managedclustersetbinding.yaml
```

- 在託管叢集綁定中使用的 namespace 中，創建放置自定義資源以選擇一組託管叢集以註冊到 ArgoCD 或 OpenShift GitOps Operator instance：

```
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: all-openshift-clusters
  namespace: openshift-gitops
spec:
  predicates:
  - requiredClusterSelector:
      labelSelector:
        matchExpressions:
        - key: vendor
          operator: "In"
          values:
          - OpenShift
```

> 注意：只有 OpenShift 叢集註冊到 Argo CD 或 GitOps Operator instance，而不是其他 Kubernetes 叢集。

- 創建一個 GitOpsCluster 自定義資源以將託管叢集從放置註冊到 Argo CD 或 OpenShift GitOps 的指定的 instance：

```
apiVersion: apps.open-cluster-management.io/v1alpha1
kind: GitOpsCluster
metadata:
  name: argo-acm-clusters
  namespace: openshift-gitops
spec:
  argoServer:
    cluster: local-cluster
    argoNamespace: openshift-gitops
  placementRef:
    kind: Placement
    apiVersion: cluster.open-cluster-management.io/v1alpha1
    name: all-openshift-clusters
    namespace: openshift-gitops
```

這使 Argo CD instance 能夠將 Application 部署到任何 ACM Hub 託管叢集中。

正如我們從前面的示例中看到的，placementRef.name 被定義為 `all-openshift-clusters`，並被指定為安裝在 argoNamespace：openshift-gitops 中的 GitOps 實例的目標叢集。

另一方面，argoServer.cluster 規範需要 `local-cluster` 值，因為將使用部署在 OpenShift 叢集中的 OpenShift GitOps，該叢集也是安裝 ACM Hub 的位置。

- 幾分鐘後，我們在 ACM Hub 中生成了 GitOps Cluster CRD，我們將能夠直接從 Application 部分的 ACM Hub 控制台定義 Application 和ApplicationSet。

## 從 ACM Hub 部署 ArgoCD/OpenShift GitOps ApplicationSet

一旦我們通過 ACM Hub 中的 GitOps Cluster CRD 啟用了 OpenShift GitOps 和 ACM 之間的整合，我們就可以直接在 ACM Hub 中部署 ApplicationSet，在一個單一的頁面中管理所有 ArgoCD Application。

另一方面，我們還將受益於[ArgoCD ApplicationSets 的不同生成器的特性](https://argocd-applicationset.readthedocs.io/en/stable/Generators/)。

使用這些生成方式，我們可以從不同叢集中的單個  Repository 部署多個 Application，利用 ApplicationSet 為每個託管的 Cluster 的從Repository 中的配置不同對象及要部署的 Application。

讓我們在 ACM Hub 中生成 ApplicationSet。

- 使用 UI 為Application 集生成一個 ApplicationSet 示例：

![](https://rcarrata.com/images/acmappA.png)

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: acm-appsets
  namespace: openshift-gitops
spec:
  generators:
    - clusterDecisionResource:
        configMapRef: acm-placement
        labelSelector:
          matchLabels:
            cluster.open-cluster-management.io/placement: acm-appsets-placement
        requeueAfterSeconds: 180
  template:
    metadata:
      name: 'acm-appsets-'
    spec:
      destination:
        namespace: bgdk
        server: ''
      project: default
      source:
        path: apps/bgd/overlays/bgdk
        repoURL: 'https://github.com/yylin1/ns-apps/'
        targetRevision: single-app
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

注意：目標 namespace 可以是 openshift-gitops。BGDK 可能會更改，但它會以這種方式離開，因為我們需要放置一個目標 namespace，即ApplicationSet 本身不需要它（Applicatio bgdk 也不需要）

- 結果是在 OpenShift GitOps 中生成但由 ACM Hub 管理的 ApplicationSet：

![](https://rcarrata.com/images/acmappB.png)

正如我們所看到的，被分配到兩個不同的 Cluster ，`bm-germany` 這 `local-cluster` 將是 Application 部署的地方，由 ApplicationSet 管理

Application 在之前定義 ApplicationSet 期間為每個叢集生成了與定義為 acm-appsets-placement 的 Placement 匹配的 ApplicationSet。還可以匹配 Cluster 進行 label，而不僅僅依賴於 Placement 對象。

- 在生成的 Application 中，每個Application都有自己的Application、Placement 和 Cluster，我們可以檢查：

![](https://rcarrata.com/images/acmappC.png)

因為我們可以檢查 ArgoCD Application 是否正確部署並由 BM-Germany 叢集中的 ACM AppSets 的 ApplicationSet 自動管理。此外，另一個 ArgoCD Application 將用於在與 Placement 匹配的另一個 Cluster 中部署另一個 Application 。


![](https://rcarrata.com/images/acmappD.png)

正如我們之前所描述的，兩個 ArgoCD Application 是由與定義的 Placement 匹配的 ApplicationSet 生成的。

- 在 OpenShift GitOps / ArgoCD argo-controller 實例中，ACM 生成的 ApplicationSet 也生成了兩個 Argo Application ，並且為與 Placement 匹配的 ClusterSet 中管理的每個 Cluster 生成了每個 ArgoCD Application ：

![](https://rcarrata.com/images/acmappE.png)

> 注意：檢查指向在早期步驟中定義的不同託管 Cluster 的目標。

- 每個 Argo ApplicationSet 管理每個託管 Cluster 中的Application ，例如在 BM-Germany  Cluster 中部署 BGDK Application 。

![](https://rcarrata.com/images/acmappF.png)

此 Application 將在託管 Cluster 中部署Application 清單，在這種情況下部署 bgdk Application 清單（路由、服務、部署等）。

- 在 ArgoCD/OpenShift GitOps 的設置中，在 ACM 使用 ClusterSet 管理的這些叢集。

![](https://rcarrata.com/images/acmappG.png)

這些是由 ACM Hub 中生成的 GitOps CRD 自動生成和管理的，它與託管 Cluster 對應。

## Reference

- https://github.com/RedHat-EMEA-SSA-Team/ns-gitops/
- https://rcarrata.com/openshift/argo-and-acm/