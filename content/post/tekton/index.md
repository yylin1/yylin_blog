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

![](https://www.redhat.com/cms/managed-files/container-platforms-pipelines.png)

## 主要特性

- OpenShift Pipelines 是一個無服務器的 CI/CD 系統，它在獨立的容器中運行 Pipelines，以及所有需要的依賴組件。
- OpenShift Pipelines 是為開發微服務架構的非中心化團隊設計的。
- OpenShift Pipelines 使用標準 CI（pipeline）定義，這些與現有的 Kubernetes 工具集成擴展可擴展和擴展，可讓您定義和擴展 Kubernetes。
- 您可以通過 OpenShift Pipelines 使用 Kubernetes （如 Source-to-Image (S2I)、Buildah、Buildpacks 和 Kaniko）構建鏡像，這些工具可以移植到任何 Kubernetes 平台。
- 您可以使用 OpenShift Container Platform 開發運行（Developer Console）來創建 Tekton 資源，查看 Pipeline 的日誌，並管理 OpenShift Container Platform 設計空間中的管道。

在 Tekton pipeline 中有以下幾個主要的組成要素，分別是：
- PipelineResource
- Task & ClusterTask
- TaskRun
- Pipeline
- PipelineRun

## PipelineResource

PipelineResource 簡單來說可以作為 task 的 input or output，而每個 task 可以有多個 input & output。

## Syntax

PipelineResource 的定義中會有以下必要資訊：

- apiVersion：目前固定是 tekton.dev/v1alpha1

- kind：因為這是 CRD，所以是 PipelineResource

- metadata：用來辨識此 TaskRun 用的資訊，例如 name

- sepc：使用 resource 的詳細資訊(例如：路徑、位址)

- type：用來指定 resource type，目前支援 git, pullRequest, image, cluster, storage, cloudevent … 等等

其他選填項目：

- params：不同的 resource type 可能會有的不同額外參數資訊

## Resource Type
有了以上概念後，接著要知道的是 PipelineResources 共有以下幾種類型：

- Git Resource
- Pull Request Resource
- Image Resource
- Cluster Resource
- Storage Resource
- Cloud Event Resource

以下就針對比較常用的 Git & Image resource 說明，其他的部份可以參考官網的詳細文件。

### Git Resource

一般的 git repository，作為 task input 時，Tekton 執行 task 前會將程式碼 clone 回來，因此這邊就必須注意 git repository 存取的權限問題，若是 private repository 就要額外提供 credential 資訊才可以正常運作

以下是一個標準的 Git PipelineResource 的定義：

```
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: wizzbang-git
  namespace: default
spec:
  type: git
  params:
    - name: url
      value: https://github.com/wizzbangcorp/wizzbang.git
    # 可用 branch, tag, commit SHA or ref
    # 沒指定就會拉 master branch
    - name: revision
      value: master
      # value: some_awesome_feature
      # value: refs/pull/52525/head
```

## Task & ClusterTask

`Task`(& `ClusterTask`) 中包含了一連串的 step，通常是使用者要用來執行 CI flow，而這些工作會在單一個 pod 中以多個 container 的形式逐一完成。

Task & ClusterTask 兩者的不同在於 Task 是屬於 namespace level，而 ClusterTask 是屬於 cluster level

而在 Task 的定義中，最重要的部份有以下三個項目：

- Input
- Output
- Steps

以下是一個 task 的標準定義內容：
```
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: deploy-using-kubectl
spec:
  inputs:
    resources:
      - name: source
        type: git
      - name: image
        type: image
    params:
      - name: path
        type: string
        description: Path to the manifest to apply
      - name: yamlPathToImage
        type: string
        description:
          The path to the image to replace in the yaml manifest (arg to yq)
  steps:
    # step 中可以定義多個執行工作，會依照順序執行
    - name: replace-image
      image: mikefarah/yq
      command: ["yq"]
      args:
        - "w"
        - "-i"
        - "$(inputs.params.path)"
        - "$(inputs.params.yamlPathToImage)"
        - "$(inputs.resources.image.url)"
    - name: run-kubectl
      image: lachlanevenson/k8s-kubectl
      command: ["kubectl"]
      args:
        - "apply"
        - "-f"
        - "$(inputs.params.path)"
  # 此 volume 在 task 中沒有用到，只是一個範例而已
  # 用以表示可以在 task 中定義 volume 並使用
  volumes:
    - name: example-volume
      emptyDir: {}
```

## TaskRun

定義了 task 之後，Tekton 並不會主動執行任何 task，這時候就必須要搭配 TaskRun 才可以讓 task 真正的執行指定工作。

以下是一個標準的 TaskRun 定義：

```
# 以下幾個(apiVersion, kind, metadata, spec)是必要資訊
apiVersion: tekton.dev/v1alpha1
kind: TaskRun
metadata:
  name: build-docker-image-from-git-source-task-run
spec:
  serviceAccount: robot-docker-basic
  # 指定到已經預先定義好的 Task
  taskRef:
    name: build-docker-image-from-git-source
  inputs:
    resources:
      - name: docker-source
        # 指定到已經預先定義好的 PipelineResource
        resourceRef:
          name: git-tekton-test
    params:
      - name: pathToDockerFile
        value: Dockerfile
      - name: pathToContext
        value: /workspace/docker-source/examples/microservices/leeroy-web
  outputs:
    resources:
      - name: builtImage
        resourceRef:
          name: image-tekton-test

```

## Pipeline

Pipeline 其實可以把它簡單思考為前面 Task 的集合，有順序性的排列，並透過之後介紹的 PipelineRun 來運作。


成功運行 Pipeline 結果:

![](https://www.redhat.com/architect/sites/default/files/styles/embed_large/public/2022-06/4-pipeline.png?itok=Eex7RxG9)

失敗運行 Pipeline 結果：

![](https://www.redhat.com/architect/sites/default/files/styles/embed_large/public/2022-06/5-ci-dev-pipeline-failed.png?itok=dzFYWfKl)

## 結語

以上內容(PipelineResource, Task, TaskRun, Pipeline, PipelineRun) 是 Tekton 中執行工作的必要元素，實際上執行的 CI/CD 工作都會與這幾個部份有關。

Tekton 將所有的基本元素拆分成一個一個的 k8s CRD(Custom Resource Definition)，如果是稍微複雜一點的 CI/CD 工作，可能就會需要定義不少個 CRD 才能完成，而且在設計上相對於其他的 CI server(例如：GitLab CI, Drone CI)可能不是這麼直覺；但這樣的設計提供了以下優點：

- 原生的 k8s 使用經驗，不需要額外學習其他語法

- 定義好的 CRD(PipelineResource, Task, Pipeline) 可以被重複利用

- 原生整合 k8s

若是未來有考慮 workload 都跑在 k8s 上的使用者，在選擇 CI/CD 的工具時或許可以將 Tekton 考慮進行。

接著可能會面臨到的問題可能是，如果希望作到 GitOps，光是以上項目好像不夠還有相關元件支援性。


## Reference

- [pipeline/examples](https://github.com/tektoncd/pipeline/tree/main/examples)
- [PipelineResources](https://github.com/tektoncd/pipeline/blob/main/docs/resources.md)