---
title: "Disconnected Environment - Introducing Mirror Registry for Red Hat OpenShift"
date: 2022-03-07T00:53:25+08:00
draft: false
url: "openshift/mirror-registry-for-ocp4-install"
description: "本篇文章介紹，如何透過 `Mirror Regristry` 一座輕量精簡的容器鏡像倉庫(Regristry)，為使用者提供了在沒有 Regristry 的離線環境中，提供託管 OpenShift 4 安裝所需的相關容器鏡像。"
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

## 情境與使用目標

OpenShift 通常是在受控制離線網路的情況下運行 Production Cluster。而客戶要使用 Red Hat Quay 在 OpenShift 上運行優勢包含，有更多擴充性與部署元件配置整合性。要在離線環境安裝 OpenShift 前提條件，需要一個存放的安裝所需的相關鏡像的鏡像倉庫 (e.g., Quay, Harbor, Docker registry)，這邊就會有雞生蛋蛋生雞問題。

解決方案/目標：

Red Hat 提供一個 “mirror registry”工具，只針對 OpenShift 部署的引導(Bootstrap)鏡像倉庫，透過自動化腳本安裝程序，提供在 RHEL 8 (or Fedora) 的系統環境，快速部署一個精簡版的 Quay ，提供保存下載特定版本 OpenShift、OpenShiftHub 鏡像檔。mirror registry 提供針對離線環境或只是單純 PoC OpenShift 情境使用的 Registry，可以透過自動化腳本，快速部署設置安裝單節點(all-in-one)的 Quay ，所需要的相關手動繁瑣設定，例如：完整網域名稱 FQDN、使用者自定義 SSL/TLS 憑證、訪問權權限 SSH key 及運行的環境選擇。

情境: 離線環境，安裝 OpenShift 運行流程:

1. 在可連線外部網路環境中，透過 mirror registry 部署第一座容器鏡像倉庫(Online Mirror) 進行 OpenShift、OpenShiftHub 必要鏡像檔存放。
2. 同樣在離線環境中，透過 mirror registry 部署第二座容器鏡像倉庫(Air-gapped Mirror)，從 Online Mirror 拷貝並要的映像檔存放於 Air-gapped Mirror
3. 透過 Air-gapped Mirror 部署 OpenShift Production/Infra Cluster
4. OpenShift 安裝 Quay Operator 提供內部服務及應用所需的鏡像存放

## 事前準備

- 從 [OpenShift console Downloads](https://console.redhat.com/openshift/downloads#tool-mirror-registry) 下載最新版本的 `mirror-registry.tar.gz`
- 準備 pull-secret.json 檔案


## 安裝

- 在當前環境安裝 mirror registry 

```
$ vim /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

127.0.0.1 registry.yylin.demolab

$ export HOSTNAME="registry.yylin.demolab"
$ ./mirror-registry install --quayHostname ${HOSTNAME} --ssh-key <~/.ssh/my_id_rsa>
```

- 最後輸出顯示 registry host 與登入資訊
```
INFO[2022-03-01 00:52:38] Quay installed successfully, permanent data are stored in /etc/quay-install
INFO[2022-03-01 00:52:38] Quay is available at https://registry.yylin.demolab:8443 with credentials (init, xxxxxxxxxxxxxxxxxxxxxx)
```

- 產生 Registry Basic Auth 之認證資訊
```
# (init, xxxxxxxxxxxxxxxxxxxxxx)
echo -n '<username>:<password>' | base64 -w0 
```

- 在 pull-secret 加入本地 Registry 認證資訊
```
    "registry.yylin.demolab:8443": {
       "auth": "aW5pdDo3TzlNV1hBMDhQcTJKZGh0M0M2elZJNU5EMWlsYkJ3NA=="
    }
```

- 驗證登入registry

```
$ podman login -u init --authfile pull-secret.json registry.yylin.demolab:8443
Error: authenticating creds for "registry.yylin.demolab:8443": pinging container registry registry.yylin.demolab:8443: Get "https://registry.yylin.demolab:8443/v2/": x509: certificate signed by unknown authority (possibly because of "crypto/rsa: verification error" while trying to verify candidate authority certificate "bastion.redhat.kubedev.org")
```

- 安裝更新 quay 憑證

> mirror registry 預設會配置好 CA 憑證，請將 PEM 文件格式添加憑證到系統中信任的 CA 列表中，這邊複製到 /usr/share/pki/ca-trust-source/anchors/ 目錄中

```
$ cp /etc/quay-install/quay-rootCA/rootCA.pem /usr/share/pki/ca-trust-source/anchors/rootCA.cert
```

> 更新系統範圍的信任儲存配置，請使用 update-ca-trust 命令：

```
$ update-ca-trust
```

- 再次登入mirror registry - Quay 確認憑證狀態是不是已經更新

```
$ podman login -u init --authfile pull-secret.json bastion.redhat.kubedev.org:8443
Password:
Login Succeeded!
```


## 開始備份 OpenShift images

1. 建立備份路徑
 
```
$ mkdir -p $HOME/openshift4/registry/images
```

2. 匯入環境變數

```
# vim upgrade-env
export OCP_RELEASE=$(oc version -o json  --client | jq -r '.releaseClientVersion') 
export LOCAL_REGISTRY="registry.yylin.demolab:8443"
export LOCAL_REPOSITORY="ocp4/openshift4"
export PRODUCT_REPO="openshift-release-dev"

# /註記/ 請確認 pull-secret.json 裡面有包含 mirror rigistry 資訊
export LOCAL_SECRET_JSON=$HOME/mirror-registry/pull-secret.json  

export RELEASE_NAME="ocp-release"
export ARCHITECTURE=x86_64

# 指定 images 存放路徑
export REMOVABLE_MEDIA_PATH="$HOME/openshift4/registry/images"

# 匯入環境變數
source upgrade-env
```

3. 備份 Image

-  /#1/ Review the images and configuration manifests to mirror:
 
```
$ oc adm release mirror -a ${LOCAL_SECRET_JSON}       --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE}      --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}      --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE} --dry-run
```

```
imageContentSources:
- mirrors:
  - registry.yylin.demolab:8443/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - registry.yylin.demolab:8443/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
```

- /#2/ Mirror the images to a directory on the removable media:

```
$ oc adm release mirror -a ${LOCAL_SECRET_JSON} --to-dir=${REMOVABLE_MEDIA_PATH}/mirror quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE}
```

```
info: Mirroring completed in 3m6.3s (66.22MB/s)

Success
Update image:  openshift/release:4.10.0-rc.3-x86_64

To upload local images to a registry, run:

    oc image mirror --from-dir=/root/openshift4/registry/images/mirror 'file://openshift/release:4.10.0-rc.3-x86_64*' REGISTRY/REPOSITORY

Configmap signature file /root/openshift4/registry/images/mirror/config/signature-sha256-3d4ada825f4aa4d2.yaml created
```

- /#3/ Take the media to the restricted network environment and upload the images to the local container registry:

```
$ oc image mirror -a ${LOCAL_SECRET_JSON} --from-dir=${REMOVABLE_MEDIA_PATH}/mirror "file://openshift/release:${OCP_RELEASE}*" ${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} 
```

```
phase 0:
  registry.redhat.demolab:8443 ocp4/openshift4 blobs=334 mounts=0 manifests=163 shared=0

info: Planning completed in 2.12s
uploading: registry.redhat.demolab:8443/ocp4/openshift4 sha256:7253c5d7bf339222334565e0ec7ac88dd017dc9b9ec01654d26e8dda07408548 81.38MiB
uploading: registry.redhat.demolab:8443/ocp4/openshift4 sha256:8e549eaeb17902a2a516d03776112ad7a9bfd10cd2d1c298fe31b1c15d3b4166 154.4MiB
uploading: registry.redhat.demolab:8443/ocp4/openshift4 sha256:11456a2e94a652afe44f2645d83a051160119fd953b2459167cc88232adbf9ed 31.45MiB
uploading: registry.redhat.demolab:8443/ocp4/openshift4 sha256:4ceffbad995c4d6e76142c6b46e7f2e06fcd2ddb6eb2c3624cc0244ae018356f 106.9MiB
...
...
sha256:2d80cc5beffc10607a53eb6c013e87d03e267cdf0b8fc678b40acf5432957714 registry.redhat.demolab:8443/ocp4/openshift4:4.10.0-rc.3-x86_64-multus-networkpolicy
sha256:df3b0ec40395ea460fbf2728ca7adff79dbaebddcffce003e88b3fb9cb2c9759 registry.redhat.demolab:8443/ocp4/openshift4:4.10.0-rc.3-x86_64-must-gather
sha256:a340ea3d86560de8cf9ee2c1afe42ad24e5eefc9903a0073abe9dc54815bf710 registry.redhat.demolab:8443/ocp4/openshift4:4.10.0-rc.3-x86_64-console-operator
sha256:f532e7f50cadfc757a6d277ab5476a31a4f24aac0afc615da6ff9f7ce5e2b538 registry.redhat.demolab:8443/ocp4/openshift4:4.10.0-rc.3-x86_64-cluster-image-registry-operator
info: Mirroring completed in 1m17.77s (5.572MB/s)
```

- /#4/ Directly push the release images to the local registry by using following command:

```
$ oc adm release mirror -a ${LOCAL_SECRET_JSON} --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} --apply-release-image-signature
```

> [更多配置請參考 Mirroring the OpenShift Container Platform image repository](https://docs.openshift.com/container-platform/4.10/openshift_images/samples-operator-alt-registry.html)

## mirror 後結果
![](https://i.imgur.com/APzRuyc.png)

---

## Reference

- [Mirroring on a local host with mirror registry for Red Hat OpenShift](https://docs.openshift.com/container-platform/4.10/installing/disconnected_install/installing-mirroring-creating-registry.html#mirror-registry-localhost_installing-mirroring-creating-registry)
- https://cloud.redhat.com/blog/introducing-mirror-registry-for-red-hat-openshift
- https://access.redhat.com/support/policy/updates/openshift#omr
- https://www.youtube.com/watch?v=j5e4OT71N0A
