---
title: "單節點 OpenShift 部署與應用探討"
slug:  "openshift"
url: "openshift/sno-installer"
date: 2022-09-24T21:40:40+08:00
draft: false
description: "所謂的單節點（Single Node OpenShift，SNO），就是在單臺實體或虛擬伺服器部署 All-in-One 的 OpenShift 叢集功能，可用於 OpenShift 控制與執行使用者工作負載，但 SNO 部署適合哪些應用場景？是我們值得探討與規劃的。"
categories:
  - OpenShift
tags:
  - OpenShift
---

OpenShift 4.9 開始正式推出提供單節點部署（Single Node OpenShift，SNO），以支援小型、全功能的企業級Kubernetes叢集的應用，常見客戶 POC 應用需求如資料中心伺服器資源有限情況下，單節點 OpenShift 部署型態能夠更容易處理，可協助企業擴充既有應用程式開發與部署規模，以及管理相關工作流程，更支援邊緣資料資料中心的執行需求。

單節點部署可以使用 RHACM 或者在線的安裝引導進行安装，此篇以安裝引導進行 SNO 部署說明：


## 1. 準備好本地的安裝環境：

- 本文章使用 [Red Hat Virtualization](https://access.redhat.com/zh_CN/content/4218151)虛擬化環境，部署 SNO 叢集的最低配置要求如 `Prerequisite` 下表資訊
- 生成 SSH 密鑰(用於SSH登陸)
- 配置部署的虛擬機 IP 或使用DHCP配置叢集網路及 DNS 伺服器解析

### Prerequisite

**主機資源要求**

| CPU | Memory | Disk |
| - | - | - |
| 8vCPU | 16GB | 120GB |

> Requirements for installing OpenShift on a single node - [minimum resource requirements](https://docs.openshift.com/container-platform/4.10/installing/installing_sno/install-sno-preparing-to-install-sno.html#install-sno-requirements-for-installing-on-a-single-node_install-sno-preparing)

### **DNS 設定**


| Usage | FQDN | 
| - | - | 
| Kubernetes API | api.<cluster_name>.<base_domain> | 
| Internal API | api-int.<cluster_name>.<base_domain> | 
| Ingress route | \*.apps.<cluster_name>.<base_domain> |

- 參考 How to try out single-node OpenShift from Red Hat 安裝指南：
{{< youtube QFf0yVAHQKc >}}

## 2. 登入安裝 [console.redhat.com](https://console.redhat.com/), 創建 OpenShift 單節點部署

### Installation - Assisted Install

- **開啟 https://console.redhat.com，選擇 OpenShift**
![](https://i.imgur.com/Qzz1mj4.png)


- **點擊上方 Create Cluster**
![](https://i.imgur.com/mNKB6GQ.png)


- **選擇 Datacenter 並點擊 Create Cluster**
![](https://i.imgur.com/tGSYnXs.png)


- **輸入 Cluster 相關參數，勾選 Install single node OpenShift**
![](https://i.imgur.com/oPwZ5kx.png)


- **Host network 部分選擇 Static network configuration**
![](https://i.imgur.com/wV0rcAv.png)


- **輸入相關網路參數**
> 此部分範例設置為 192.168.10.0/24 網段

![](https://i.imgur.com/SgPDkJ5.png)
![](https://i.imgur.com/X9XE6Ev.png)


- **點選 Add host**
![](https://i.imgur.com/vhYpStz.png)


- **選擇 Minimal image file，並輸入對應 ssh public key 內容**
![](https://i.imgur.com/zJwFOr9.png)


- **下載 Discovery ISO**
![](https://i.imgur.com/1N1LkT7.png)


- **啟動 VM 並掛載該 ISO，等待其自動安裝與設定，完成後應該看到 Host Inventory 顯示如下**
![](https://i.imgur.com/QngzQ70.png)


- **針對錯誤部分進行修改，此處問題為 hostname 不能為 localhost**
![](https://i.imgur.com/vEYp79z.png)
![](https://i.imgur.com/TI0hOOP.png)


- **確認沒問題後即可進行下一步**
![](https://i.imgur.com/yzBx0w6.png)


- **最後檢查一遍設定，確認無誤後點擊 Install cluster**
![](https://i.imgur.com/sxKA3O1.png)


- **等待安裝完畢**

> 單節點安裝時間大約 40 分鐘

![](https://i.imgur.com/rE8Ykdw.png)
- ![](https://i.imgur.com/5hl2Gqc.png)

- **安裝完成後即可根據以下連線資訊使用 Single Node OCP**

![](https://i.imgur.com/Bvn1ERs.png)


- **最終登入 OCP 查看狀態**


![](https://i.imgur.com/tZG1QkE.png)
![](https://i.imgur.com/BidpalP.jpg)





## Reference
1. [Demo: How to try out single-node OpenShift from Red Hat](https://www.youtube.com/watch?v=QFf0yVAHQKc&ab_channel=OpenShift)
2. [Preparing to install on a single node](https://docs.openshift.com/container-platform/4.10/installing/installing_sno/install-sno-preparing-to-install-sno.html)
3. [OpenShift 4.10 - Installing OpenShift on a single node](https://docs.openshift.com/container-platform/4.10/installing/installing_sno/install-sno-installing-sno.html)
4. [OpenShift Virtualization on a Single Node Cluster](https://www.youtube.com/watch?v=PE8W8OKJoXc&ab_channel=OpenShift)
5. [Meet single node OpenShift: Our newest small OpenShift footprint for edge architectures](https://www.redhat.com/en/blog/meet-single-node-openshift-our-smallest-openshift-footprint-edge-architectures)
6. https://github.com/lees07/tech-docs/blob/master/e1-sno-by-assisted-installer.md
7. https://www.youtube.com/watch?v=leJa9HmvdI0&ab_channel=RyanNix

