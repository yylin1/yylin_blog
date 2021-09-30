---
title: "OpenShift Intsall Troubleshooting Notes"
date: 2021-09-29T00:53:25+08:00
draft: false
description: "本篇文章主要是整理 OpenShift 4.x 安裝遇到問題與相關除錯常用介紹。"
categories:
  - OpenShift
tags:
  - Troubleshooting
---

延伸上一篇 OCP UPI 部署([OpenShift 4.8.x UPI install on Bare metal](https://blog.yylin.io/p/openshift-4.8.x-upi-install-on-bare-metal))過程遇到問題彙整。

> 備註: 由於資源有限情況下，透過單一伺服器(主機)透過 VM 模擬實際節點硬體資源環境，這邊主要透過檢測與除錯，來確保 OpenShift 部署成功。

## 部署常見問題整理 (持續更新)

### 超時等待 `bootstrap` 節點啟動

- 發現特定 Pod 持續`RunningNotReady` 狀態

```bash=
W0830 11:16:37.087497   35214 reflector.go:436] k8s.io/client-go/tools/watch/informerwatcher.go:146: watch of *v1.ConfigMap ended with: very short watch: k8s.io/client-go/tools/watch/informerwatcher.go:146: Unexpected watch close - watch lasted less than a second and no items received
apiserver - > RunningNotReady
Aug 30 03:42:37 bootstrap.lab.yiylin.internal bootkube.sh[644199]:         Pod Status:openshift-kube-apiserver/kube-apiserver        RunningNotReady
```

> 此情境請透過收集`bootstrap` 節點日誌，來確認環境問題

```bash
$ ./openshift-install gather bootstrap --dir=<installation_directory> 
...
INFO Pulling debug logs from the bootstrap machine
INFO Bootstrap gather logs captured here "<installation_directory>/log-bundle-<timestamp>.tar.gz"
```
- 蒐集包含 bootstrap process / bootstrap 上所有 Pod/Container 的 log
- 下載完後解壓縮 `.tar.gz` 檔案，針對可能部署或未啟動的`Pod`檢查log錯誤狀態，來找到對應的錯誤資訊

> 補充說明 bootstrap 主要分成兩個動作
> 1. 安裝自己跟建立自己的 cluster(Pod)
> 2. 引導安裝 master x 3 
> 3. master bootstrap 全部安裝完，然後下 wait for complete 確認最終才可移除 bootstrap node

#### Reference: 

- [Gathering logs from a failed installation](https://docs.openshift.com/container-platform/4.8/installing/installing-troubleshooting.html#installation-bootstrap-gather_installing-troubleshooting)
- [Waiting for the bootstrap process to complete](https://docs.openshift.com/container-platform/4.8/installing/installing_bare_metal/installing-bare-metal.html#installation-installing-bare-metal_installing-bare-metal)

### Master Node 部署不起來，檢查並驗證 etcd 運行之硬碟是否正常運作 ?

- 大部分 master 無法建置情境，通常可能會有原因是硬碟速度問題(硬碟壞軌)
> 造成叢集有時候部署成功有時候會失敗的情況

透過簡單運行 `fio` 來測試 etcd benchamrk performance 檢測
```bash
$ oc debug node/<master_node>
[...]
sh-4.4# chroot /host bash
[root@<master_node> /]# podman run --volume /var/lib/etcd:/var/lib/etcd:Z quay.io/openshift-scale/etcd-perf
```

#### Reference:
-  [How to Use 'fio' to Check Etcd Disk Performance in OCP
](https://access.redhat.com/solutions/4885641)
-  [[參考影片說明] OpenShift 4 如何使用 fio 驗證 etcd 運行之硬碟是否可以正常運作 / Use 'fio' to Check Etcd Disk Performance](https://www.youtube.com/watch?v=6qjsh9J3ndM&ab_channel=PingChunHuang)
- [Hardware requirements and recommendations](https://www.ibm.com/docs/en/cloud-private/3.2.x?topic=requirements-hardware-recommendations)


## 補充|檢查 - 除錯流程

1.  部署過程都透過 bootkube 確認部署狀態資訊，log 會不斷顯示錯誤然後 loop 確認環節，嘗試能不能從他錯誤訊息找出規則(也就是他跳錯誤的地方，通常是在 log restart之前)

2.  嘗試 SSH 到 Master Node，透過 `netstat -plnt` 檢查三個服務，這三個通常只要好，這台 Master 就沒問題 (6443| 22623|2379|2380)，分別是 OpenShift API/Kubernetes API/ETCD/Machine Config Operator

3. 如果以上幾個 Pod 有問題，透過 `crictl ps` 跟 `crictl log` 去查看 Container Log，問題有可能是連續的錯誤，所以可能是 API 有問題，可能是 Controller/Scheduler 引起，細節需要透過上面提到方式，下載節點所有 Log 日誌來比對檢查，找到正確的錯誤資訊來排除錯誤

## Reference:
- [OpenShift Troubleshooting Resources](https://connect.redhat.com/en/blog/openshift-troubleshooting-resources)
