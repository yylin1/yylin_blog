---
title: "CI/CD: Tekton Pipeline 實戰"
slug:  "tekton"
url: "openshift/pipelines"
date: 2022-06-21T21:40:40+08:00
draft: false
description: "本篇介紹 Tekton 是一個基於 Kuberentes 原生的 CI/CD 工具，經常分別和常見工具 Jenkine, Jenkins X, Spinnaker做比較，這邊會針對Tekton介紹概念後，並如何在 OpenShift 中部署 OpenShift Pipeline，及做其他開源工具差異情境比較。"
categories:
  - CI/CD
tags:
  - OpenShift
  - CICD
  - Tekton
---

OpenShift Pipelines 是一個基於 Kubernetes 資源的雲塊的持續和持續交付（持續集成和持續交付，簡稱 CI/CD）的解決方案。它通過執行執行的細節，使用 Tekton 進行跨平台的自動部署。Tekton 引入了多種標準的自定義資源定義 (CRD)，定義可跨 Kubernetes 分佈用於 CI/CD 管道。

## 主要特性

- OpenShift Pipelines 是一個無服務器的 CI/CD 系統，它在獨立的容器中運行 Pipelines，以及所有需要的依賴組件。
- OpenShift Pipelines 是為開發微服務架構的非中心化團隊設計的。
- OpenShift Pipelines 使用標準 CI（pipeline）定義，這些與現有的 Kubernetes 工具集成擴展可擴展和擴展，可讓您定義和擴展 Kubernetes。
- 您可以通過 OpenShift Pipelines 使用 Kubernetes （如 Source-to-Image (S2I)、Buildah、Buildpacks 和 Kaniko）構建鏡像，這些工具可以移植到任何 Kubernetes 平台。
- 您可以使用 OpenShift Container Platform 開發運行（Developer Console）來創建 Tekton 資源，查看 Pipeline 的日誌，並管理 OpenShift Container Platform 設計空間中的管道。

成功運行 Pipeline 結果:

![](https://www.redhat.com/architect/sites/default/files/styles/embed_large/public/2022-06/4-pipeline.png?itok=Eex7RxG9)

失敗運行 Pipeline 結果：

![](https://www.redhat.com/architect/sites/default/files/styles/embed_large/public/2022-06/5-ci-dev-pipeline-failed.png?itok=dzFYWfKl)

