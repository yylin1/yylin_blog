---
title: "OpenShift 啟用 GPU Operator Dashboard"
slug:  "gpu"
url: "openshift/gpu-utilization"
date: 2022-06-28T21:40:40+08:00
draft: false
description: "本篇介紹如何在安裝 NVIDIA GPU Operator 後啟用 console-plugin 於 OpenShift Console 顯示 GPU Utilization。"
categories:
  - GPU
tags:
  - OpenShift
  - GPU
---

## 先決條件：
- Bastion 安裝 Helm
- 當前 OpenShift 版本為 4.10+
- Nvidia GPU Operator 已經完成安裝

## 啟用 NVIDIA GPU Operator Usage 資訊

1. 添加 helm repo:

```
$ helm repo add rh-ecosystem-edge https://rh-ecosystem-edge.github.io/console-plugin-nvidia-gpu
```

2. 更新 repo:

```
$ helm repo update
```

3. 安裝 `helm chart` 於預設 NVIDIA GPU Operator namespace:

```
$ helm install -n nvidia-gpu-operator console-plugin-nvidia-gpu rh-ecosystem-edge/console-plugin-nvidia-gpu

$ kubectl -n nvidia-gpu-operator get all -l app.kubernetes.io/name=console-plugin-nvidia-gpu


# 啟用 plugin 執行以下 command: 

$ kubectl patch consoles.operator.openshift.io cluster --patch '[{"op": "add", "path": "/spec/plugins/-", "value": "console-plugin-nvidia-gpu" }]' --type=json
```

4. 查看部署的資源：

```
$ oc -n nvidia-gpu-operator get all -l app.kubernetes.io/name=console-plugin-nvidia-gpu
```

5. 驗證 plugins 是否已指定

```
$ oc get consoles.operator.openshift.io cluster --output=jsonpath="{.spec.plugins}"
```
- 如果未指定，則運行以下 command 以啟用 plugin：

```
$ oc patch consoles.operator.openshift.io cluster --patch '{ "spec": { "plugins": ["console-plugin-nvidia-gpu"] } }' --type=merge
```

- 如果指定，則運行以下 command 以啟用 plugin：

```
$ oc patch consoles.operator.openshift.io cluster --patch '[{"op": "add", "path": "/spec/plugins/-", "value": "console-plugin-nvidia-gpu" }]' --type=json
```

在 OCP Web Console 頁面中（Home > Overview）就可以查閱 GPU utilization:

![](https://docs.nvidia.com/datacenter/cloud-native/_images/gpu_overview_dashboard1.png)

---

## Reference
- https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/openshift/enable-gpu-op-dashboard.html
