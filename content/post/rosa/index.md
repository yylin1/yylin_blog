---
title: "Red Hat OpenShift Platform on AWS (ROSA) 快速部署上手"
slug:  "rosa"
url: "openshift/rosa"
date: 2021-09-30T14:54:40+08:00
draft: false
description: "本篇文章主要分享如何透過 ROSA 完全託管叢集資源配置，使基礎設施團隊在AWS上能更快速使用 Red Hat OpenShift 服務。"
categories:
  - Cloud Service Provider 
tags:
  - AWS
  - OpenShift
  - ROSA
---


## 事前準備：

開始前，需要先準備以下資訊與要求。

### 啟用 ROSA

> 開始配置 AWS 服務前，使用者必須以 [IAM](https://docs.openshift.com/rosa/rosa_getting_started/rosa-aws-prereqs.html#rosa-policy-iam_prerequisites) 身份(參考文件: [Customer Requirements](https://docs.openshift.com/rosa/rosa_getting_started/rosa-aws-prereqs.html#rosa-customer-requirements_prerequisites))

登入 AWS 管理控制台，確認當前使用者`已啟用 Red Hat OpenShift`服務
![](https://i.imgur.com/lbpYgG6.png)

### 下載 AWS CLI 與 OpenShift CLI 工具:


**URL Dowload**
- 選擇當前環境下載對應`CLI` ( [Command-line interface (CLI) tools](https://console.redhat.com/openshift/downloads))
    
    ![](https://i.imgur.com/c64k4mB.png)

or 

**Command Dowload**
- 下載 `AWS/OpenShift CLI` 工具，並解壓縮檔案到 /usr/local/bin/ 底下:

```bash
# CLI
$ wget -c https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/rosa/latest/rosa-linux.tar.gz -O - | tar -xz
$ mv rosa /usr/local/bin/
$ rosa version
1.1.1

$ wget -c https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest-4.8/openshift-client-linux.tar.gz -O - | tar -xz
$ mv {oc,kubectl} /usr/local/bin/
$ oc version
Client Version: 4.8.2
$ oc version
Client Version: 4.8.10
Server Version: 4.8.12
Kubernetes Version: v1.21.1+d8043e1
```

> 創建 Red Hat 帳號
> OpenShift 由 Red Hat SaaS 提供，因此您需要創建一個 Red Hat 帳戶。
> https://cloud.redhat.com/


---

## 透過 ROSA 指令檢查憑證與資源配置

這邊以 Red Hat 部署 ROSA 文件部署為主 ([Red Hat OpenShift Service on AWS - Creating a ROSA cluster](https://docs.openshift.com/rosa/rosa_getting_started/rosa-creating-cluster.html))

首先，使用 `ROSA CLI` 檢查 AWS 憑證 

```bash
$ rosa verify permissions
I: Validating SCP policies...
I: AWS SCP policies ok
```

第一次登錄時，你會需要一個 Token 來登入你的 Red Hat 帳號
> 需要創建一組 Red Hat 帳號

確認登入訊息：
```bash
$ rosa login
I: Logged in as 'xxxx@mail.com' on 'https://api.openshift.com'
```

配置完成後可以透過 `rosa whoami` 來檢查你目前所有登錄憑證與狀態
> Red Hat 帳號會附帶 OpenShift Cluster Manager (OCM) 資訊與你的 AWS 帳戶顯示。

```
$ rosa whoami
AWS Account ID:               xxxxxxxxxxxx
AWS Default Region:           us-west-2
AWS ARN:                      arn:aws:iam::xxxxxxxxxxxx:user/user
OCM API:                      https://api.openshift.com
OCM Account ID:               xxxxxxxxxxxxxxxxxxxxxx
OCM Account Name:             xxxxxx
OCM Account Username:         xxxxx@xxxxx.com
OCM Account Email:            xxxxx@xxxxx.com
OCM Organization ID:         xxxxxxxxxxxxxxxxxxxxxx
OCM Organization Name:        Red Hat
OCM Organization External ID: xxxxxxxx
```


以上檢查沒問題後，接下運行 `rosa init` 運行指令來確保 ROSA 配置與相關資源狀態沒問題

```
$ rosa init
I: Logged in as 'xxxxxx@mail.com' on 'https://api.openshift.com'
I: Validating AWS credentials...
I: AWS credentials are valid!
I: Validating SCP policies...
I: AWS SCP policies ok
I: Validating AWS quota...
I: AWS quota ok. If cluster installation fails, validate actual AWS resource usage against https://docs.openshift.com/rosa/rosa_getting_started/rosa-required-aws-service-quotas.html
I: Ensuring cluster administrator user 'osdCcsAdmin'...
I: Admin user 'osdCcsAdmin' already exists!
I: Validating SCP policies for 'osdCcsAdmin'...
I: AWS SCP policies ok
I: Validating cluster creation...
I: Cluster creation valid
I: Verifying whether OpenShift command-line tool is available...
I: Current OpenShift Client Version: 4.8.10
```

> 需要驗證當前環境，是否已經登入 AWS並確認透過`rosa init` 來確認 AWS 資源與配置是否能足夠創建 OpenShift


### 申請 AWS 帳戶配額

在配置 ROSA 時作為預先檢查 `$ rosa verify quota --region=${cluster}` 叢集名稱。如果您在剛剛部署但未執行任何操作的帳戶上運行它，您將收到錯誤訊息 EC2 `quota 配置不足`。

```bash
$ rosa verify quota --region=us-west-2
I: Validating AWS quota...
E: Insufficient AWS quotas
E: Service quota is insufficient for the following service quota codes:
- Service ec2 quota code L-1216C47A Running On-Demand Standard (A, C, D, H, I, M, R, T, Z) instances not valid, expected quota of at least 100, but got 5
```

這邊需要額外申請 AWS EC2 限制增加配額
![](https://i.imgur.com/X2PtefK.png)

需要提供申請需求與相關資訊，才能申請限制配額成功

![](https://i.imgur.com/fMprN3I.png)

相關配額資訊參考
- [AWS service quotas](https://docs.aws.amazon.com/general/latest/gr/aws_service_limits.html)
- [Required AWS service quotas](https://docs.openshift.com/rosa/rosa_getting_started/rosa-required-aws-service-quotas.html)

![](https://i.imgur.com/4K3BWLp.png)



以上完成申請後，再次驗證

```bash
$ rosa verify quota --region=us-west-2
I: Validating AWS quota...
I: AWS quota ok. If cluster installation fails, validate actual AWS resource usage against https://docs.openshift.com/rosa/rosa_getting_started/rosa-required-aws-service-quotas.htm
```
---

## 透過 ROSA 指令快速創建 Red Hat OpenShift

- 透過 `rosa create cluster` 開始部署 OpenShift Cluster (部署Cluster 大約需要30-40分鐘)

```bash
$ rosa create cluster --cluster-name=rosacluster
I: Creating cluster 'rosacluster'
I: To view a list of clusters and their status, run 'rosa list clusters'
I: Cluster 'rosacluster' has been created.
I: Once the cluster is installed you will need to add an Identity Provider before you can login into the cluster. See 'rosa create idp --help' for more information.
I: To determine when your cluster is Ready, run 'rosa describe cluster -c newsblogcluster'.
I: To watch your cluster installation logs, run 'rosa logs install -c newsblogcluster --watch'.
Name:                       newsblogcluster
ID:                         XXXXXXXXXXXXXXXXXXXX
External ID:                
OpenShift Version:          
Channel Group:              stable
DNS:                        newsblogcluster.phnh.p1.openshiftapps.com
AWS Account:                123456789012
API URL:                    
Console URL:                
Region:                     xxxxxxxxxxx
Multi-AZ:                   false
Nodes:
 - Master:                  3
 - Infra:                   2
 - Compute:                 2 (m5.xlarge)
Network:
 - Service CIDR:            172.30.0.0/16
 - Machine CIDR:            10.0.0.0/16
 - Pod CIDR:                10.128.0.0/14
 - Host Prefix:             /23
State:                      pending (Preparing account)
Private:                    No
Created:                    xxxxxxxxxxxxxxxx
Details Page:               https://cloud.redhat.com/openshift/details/XXXXXXXXXXXXXXXXXXXX
```
執行命令後，您可以透過 `rosa describe cluster` 或 `rosa logs install` 指令檢查 OpenShift Cluster 狀態。您還可以從 URL 訪問 OpenShift Console 進行確認。

![](https://i.imgur.com/DvBaXFH.png)
![](https://i.imgur.com/KkZJ3zn.png)

![](https://i.imgur.com/LUfY9n9.png)

```
$ rosa create admin -c rosacluster
W: It is recommended to add an identity provider to login to this cluster. See 'rosa create idp --help' for more information.
I: Admin account has been added to cluster 'rosacluster'.
I: Please securely store this generated password. If you lose this password you can delete and recreate the cluster admin user.
I: To login, run the following command:

   oc login https://api.rosacluster.hffh.p1.openshiftapps.com:6443 --username cluster-admin --password fTwWy-rYJJU-wUua7-W3G6Z

I: It may take up to a minute for the account to become active.
```


```bash
$ oc login https://api.rosacluster.hffh.p1.openshiftapps.com:6443 --username cluster-admin --password fTwWy-rYJJU-wUua7-W3G6Z
Login successful.

You have access to 87 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default".
```

```bash
$ oc get no
NAME                                         STATUS   ROLES          AGE   VERSION
ip-10-0-153-95.us-west-2.compute.internal    Ready    worker         42m   v1.21.1+d8043e1
ip-10-0-171-119.us-west-2.compute.internal   Ready    infra,worker   17m   v1.21.1+d8043e1
ip-10-0-175-165.us-west-2.compute.internal   Ready    infra,worker   18m   v1.21.1+d8043e1
ip-10-0-202-234.us-west-2.compute.internal   Ready    master         49m   v1.21.1+d8043e1
ip-10-0-239-231.us-west-2.compute.internal   Ready    master         49m   v1.21.1+d8043e1
ip-10-0-242-134.us-west-2.compute.internal   Ready    master         49m   v1.21.1+d8043e1
ip-10-0-252-208.us-west-2.compute.internal   Ready    worker         42m   v1.21.1+d8043e1
```


## 刪除 Cluster

`rosa delete cluster` 可以使用指令刪除使用 ROSA 創建的cluster。順便說一下，集群的刪除在大約 10 分鐘內完成。
```bash
$ rosa delete cluster --cluster=rosacluster
? Are you sure you want to delete cluster rosacluster? Yes
I: Cluster 'rosacluster' will start uninstalling now
I: To watch your cluster uninstallation logs, run 'rosa logs uninstall -c rosacluster --watch
```

## Reference

- rosaworkshop.io
