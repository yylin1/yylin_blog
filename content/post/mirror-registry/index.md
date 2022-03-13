---
title: "Disconnected Environment - Introducing Mirror Registry for Red Hat OpenShift"
date: 2022-03-07T00:53:25+08:00
draft: false
url: "openshift/mirror-registry-for-ocp4-install"
description: "本篇文章介紹，如何透過 `Mirror Regristry ` 一座輕量精簡的容器鏡像倉庫(Regristry)，為使用者提供了在沒有 Regristry 的離線環境中，提供託管 OpenShift 4 安裝所需的相關容器鏡像。"
categories:
  - OpenShift
tags:
  - Registry
---

本文件如何安裝 OpenShift 4.8 版本至虛擬機器上，這邊將以 RHEL 環境來進行測試。

- RHEL 8 or Fedora machine with podman v3.3 installed
- Fully qualified domain name for the Quay service (must resolve via DNS, or at least /etc/hosts)
- Passwordless sudo access on the target host (rootless install tbd)
- Key-based SSH connectivity on the target host (will be set up automatically for local installs, in case of remote hosts see here)
- make (only if compiling your own installer)


當我們的基礎設施處於離線環境時，需要創建容器鏡像倉庫(Registry)來託管安裝 OpenShift Container Platform 所需的相關鏡像檔，來正常部署 OpenShift 叢集。

這是有關 mirror registry for Red Hat OpenShift OpenShift 4 的官方流程圖（可在 Red Hat Blog 頁面中找到）：

![](https://i.imgur.com/49Qc3LU.png)


## 事前準備



---


## Reference

- https://cloud.redhat.com/blog/introducing-mirror-registry-for-red-hat-openshift
- https://access.redhat.com/support/policy/updates/openshift#omr
- https://www.youtube.com/watch?v=j5e4OT71N0A
- https://cloud.redhat.com/blog/introducing-mirror-registry-for-red-hat-openshift


