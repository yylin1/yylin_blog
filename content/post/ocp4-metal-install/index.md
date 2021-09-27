---
title: "OpenShift 4.8.x UPI install on Bare metal"
date: 2021-09-26T14:54:40+08:00
draft: false
description: "本篇文章主要是說明 OpenShift 4.8 裸機伺服器(Bare-metal Server)部署過程與環境架構整理，並針對過程遇到問題透過記錄整理過程，延伸可能經常遇到的問題與探討。"
categories:
  - OpenShift
tags:
  - OCP 4.x 
---

> [註記] 由於硬體環境資源有限，本實驗是透過 [`Proxmox Environment`](https://pve.proxmox.com/wiki/Main_Page) 單一伺服器節點來模擬部署 `OpenShift` 叢集，並依據官網配置建議硬體需求(Server sizing)資源。
> [共同編輯]: [Kyle Bai](https://k2r2bai.com/)

## 部署版本說明：
- Proxmox Environment v6.1-7
    -  CPU: 8-Core, 16-Thread Unlocked / Memory: 128 GB
- OpenShift v4.8.x.

## Reference
- 參考官網文件 - [Installing a user-provisioned cluster on bare metal](https://docs.openshift.com/container-platform/4.8/installing/installing_bare_metal/installing-bare-metal.html)


## 事前準備
開始前，需要先準備以下資訊與要求。

### Download URLs

* [User-provisioned infrastructure](https://console.redhat.com/openshift/install/metal/user-provisioned): 可下載 OpenShift Installer、CLI 與 Pull secrets(需要登入 Red Hat 註冊帳號權限)
* [Public OpenShift 4.8 Mirror](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest-4.8/): 可下載 OpenShift Installer、CLI 與 OPM。
    * **openshift-client-linux.tar.gz**: 包含 oc 與 kubectl CLI 工具。
    * **openshift-install-linux.tar.gz**: 包含 openshift-install 工具。用於建立 OpenShift auth、igntion 等設定檔案。
    * **opm-linux.tar.gz**: 用於管理 Operator Hub 與 mirror 相關 images。
* [Public RHCOS 4.8 Mirror](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.8/latest/): 可下載 RHCOS 4.8 相關 VM images。
    * 若手動安裝以下載 `rhcos-live.x86_64.iso` 為主。若想要控管使用版本，請選擇有標示版本的 live iso。
        * FET vSphere 安裝請以這方式進行。
    * 若以 PXE 方式安裝，則下載以下映像檔:
        * rhcos-live-initramfs.x86_64.img
        * rhcos-live-kernel-x86_64
        * rhcos-live-rootfs.x86_64.img

> 過程中會透過 `wget` 下載，因此在有網路狀況下，不需要預先下載。

---

### Network and Hosts Info

* **OpenShift subnet CIDR**: 192.168.101.0/24
* **OpenShift Pod Network(CNI)**: 10.128.0.0/14
* **OpenShift Service Network**: 172.3.0.0/16
* **Domain**: lab.yiylin.internal

| Hosts      | IP               | CPU      | RAM    |
| --------   | --------         | -------- | ------ |
| bashtion   |  192.168.101.9   | 2 vCore  | 4G     |
| bootstrap	 |  192.168.101.10	| 4 vCore  | 8G     |
| master-1   |  192.168.101.21  | 4 vCore  | 8G     |
| master-2   |  192.168.191.22  | 4 vCore  | 8G     |
| master-3   |  192.168.191.23  | 4 vCore  | 8G     |
| worker-1   |  192.168.191.31  | 4 vCore  | 8G     |
| worker-2   |  192.168.191.32  | 4 vCore  | 8G     |
| infra-1    |  192.168.191.41  | 4 vCore  | 8G     |
| infra-2    |  192.168.191.42  | 4 vCore  | 8G     |
| infra-3    |  192.168.191.43  | 4 vCore  | 8G     |


> 這裡為了方便測試，將 PXE / HAProxy / DNS Server 等都放在 bastion上。
> [額外參考資訊]各元件節點最小資源可參考 [Minimum resource requirements](https://docs.openshift.com/container-platform/4.8/installing/installing_platform_agnostic/installing-platform-agnostic.html#minimum-resource-requirements_installing-platform-agnostic)。
> 請先透過 `Proxmox Environment` 開啟對應 `hardware` VM。

#### 補充 Network 說明
> 其中 `OpenShift subnet CIDR` 請修改為自己當前 Proxmox Environment 網路配置)
>![](https://i.imgur.com/8UnOcXL.jpg)


## Bastion
透過 Live CD 安裝 RHEL 8 版本(或使用 CentOS 8)，並確保網路設定正確。
> RHEL 需要確認 `subscription` 驗證，確保 repo 源可正常取得

### 事前準備
進入 `bastion` 機器，安裝 jq 與 wget 工具等:

```sh
$ sudo su -
$ yum install -y wget jq vim
```

關閉 bastion 防火牆:

```shell
$ systemctl stop firewalld
$ systemctl disable firewalld
```
&gt; 這邊方便測試，所以直接關閉 :D。

設定與關閉 SELinux:

```shell
$ vim /etc/selinux/config
SELINUX=disabled

$ setenforce 0
```
> 這邊方便測試實驗，所以直接關閉。

建立 SSH Keys:

```sh
$ ssh-keygen -t rsa
```
> 這邊方便測試，以 root 來建立。

導入以下環境變數:

```sh
export DOMAIN=lab.yiylin.internal
export OCP_NETWORK_CIDR=192.168.101.0/24 
```

下載 OpenShift Installer 與 CLI 工具，並解壓縮檔案到 `/usr/local/bin/` 底下:

```sh
# CLI
$ wget -c https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest-4.8/openshift-client-linux.tar.gz -O - | tar -xz
$ mv {oc,kubectl} /usr/local/bin/
$ oc version
Client Version: 4.8.2

# Installer
$ wget -c https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest-4.8/openshift-install-linux.tar.gz -O - | tar -xz
$ mv openshift-install /usr/local/bin/
$ openshift-install version
openshift-install 4.8.2
```

### 安裝與設定 DNS Server
安裝 DNS server 套件，這邊使用 `dnsmasq` 來提供服務:

```sh
$ yum -y install dnsmasq bind-utils
```
> 這部分也可參考 OCP 官網安裝 [BIND](https://docs.openshift.com/container-platform/4.8/installing/installing_bare_metal/installing-bare-metal.html#installation-dns-user-infra-example_installing-bare-metal)，關於不同 `DNS Server` 相關配置方法，會額外補充於後續文章介紹。

編輯 `/etc/dnsmasq.d/openshift.conf` 設定檔，以設定 A Record 與 PTR:

```sh
$ cat << "EOF" > /etc/dnsmasq.d/openshift.conf
server=8.8.8.8

domain=${DOMAIN},${OCP_NETWORK_CIDR},local

# A Records(api)
host-record=api.${DOMAIN}, 192.168.101.9
host-record=api-int.${DOMAIN},
host-record=api-int.${DOMAIN},192.168.101.9

# A Records(hosts)
host-record=bastion.${DOMAIN},192.168.101.9
host-record=bootstrap.${DOMAIN},192.168.101.10
host-record=master-1.${DOMAIN},192.168.101.21
host-record=master-2.${DOMAIN},192.168.101.22
host-record=master-3.${DOMAIN},192.168.101.23
host-record=worker-1.${DOMAIN},192.168.101.31
host-record=worker-2.${DOMAIN},192.168.101.32
host-record=infra-1.${DOMAIN},192.168.101.41
host-record=infra-2.${DOMAIN},192.168.101.42
host-record=infra-3.${DOMAIN},192.168.101.43

# A Records(etcd)
host-record=etcd-0.${DOMAIN},192.168.101.21
host-record=etcd-1.${DOMAIN},192.168.101.22
host-record=etcd-2.${DOMAIN},192.168.101.23

srv-host=_etcd-server-ssl._tcp.${DOMAIN},etcd-0.${DOMAIN},2380,0,10
srv-host=_etcd-server-ssl._tcp.${DOMAIN},etcd-1.${DOMAIN},2380,0,10
srv-host=_etcd-server-ssl._tcp.${DOMAIN},etcd-2.${DOMAIN},2380,0,10

# PTRs
address=/apps.${DOMAIN}/192.168.101.9
address=/.apps.${DOMAIN}/192.168.101.9
address=/api.${DOMAIN}/192.168.101.9
address=/api-int.${DOMAIN}/192.168.101.9
address=/bastion.${DOMAIN}/192.168.101.9
address=/bootstrap.${DOMAIN}/192.168.101.10
address=/master-1.${DOMAIN}/192.168.101.21
address=/master-2.${DOMAIN}/192.168.101.22
address=/master-3.${DOMAIN}/192.168.101.23
address=/etcd-0.${DOMAIN}/192.168.101.21
address=/etcd-1.${DOMAIN}/192.168.101.22
address=/etcd-2.${DOMAIN}/192.168.101.23
address=/worker-1.${DOMAIN}/192.168.101.31
address=/worker-2.${DOMAIN}/192.168.101.32
address=/infra-1.${DOMAIN}/192.168.101.41
address=/infra-2.${DOMAIN}/192.168.101.42
address=/infra-3.${DOMAIN}/192.168.101.43
EOF
```


啟動 dnsmasq 服務，並檢測 dnsmasq 設定是否正確:

```sh
$ systemctl enable dnsmasq; systemctl start dnsmasq
$ dnsmasq --test
dnsmasq: syntax check OK.
```

編輯 `/etc/resolv.conf` 檔案，修改 nameserver:

```sh
$ cat << "EOF" > /etc/resolv.conf
nameserver 127.0.0.1
EOF
```

驗證 DNS server 正反解:

```sh
# dig domain @dns_server
$ dig bastion.${DOMAIN} @10.77.110.39

# dig -x ip @dns_server
$ dig -x 10.77.110.39 @10.77.110.39
```

### 安裝與設定 HTTP Server
安裝與設定 httpd 套件:

```sh
$ yum -y install httpd
```

編輯 `/etc/httpd/conf/httpd.conf` 設定檔:

```sh
#Listen 12.34.56.78:80
Listen 8080	
```
&gt; 因為 80 是 Ingress-http 使用，除非將 HAProxy 拆成不同節點或用 External LB。

啟動 httpd 服務:

```sh
$ systemctl enable httpd; systemctl start httpd
```

### 產生 OpenShift Ignition fils 
建立放置 OpenShift Config 檔案的目錄:

```sh
$ cd $HOME ; mkdir $HOME/ocp4/ignition;cd $HOME/ocp4/ignition
```

建立與編輯 `install-config.yaml` 檔案:

```yaml
apiVersion: v1
baseDomain: yiylin.internal
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: lab
networking:
  clusterNetworks:
  - cidr: 10.128.0.0/14
    hostPrefix: 24
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.3.0.0/16
platform:
  none: {}
pullSecret: '{"auths":{ YOUR_PULL_SECRETE }}'
sshKey: "SSH_PUB_KEY"
```
> * Pull secret 可在 [Create Cluster by user provisioned](https://cloud.redhat.com/openshift/install/metal/user-provisioned) 取得。
> * Proxy 設定，參考 [Configuring the cluster-wide proxy during installation](https://docs.openshift.com/container-platform/4.8/installing/installing_bare_metal/installing-restricted-networks-bare-metal.html#installation-configure-proxy_installing-restricted-networks-bare-metal)。
> * SSH Key 請自行於 `bastion` 創建填寫，這裡不多做說明

產生 manifests 相關檔案:

```sh
$ cp install-config.yaml install-config.yaml.bak
$ openshift-install create manifests --dir=$HOME/ocp4/ignition
$ ls $HOME/ocp4/ignition
manifests  openshift
```

編輯 `manifests/cluster-scheduler-02-config.yml` 檔案，設定 `mastersSchedulable` 值，以確保 master 不作為工作負載節點:

```yaml
...
spec:
  mastersSchedulable: false
... 
```

產生 OCP 使用的 Ignition files:

```sh
$ openshift-install create ignition-configs --dir=$HOME/ocp4/ignition
$ ls $HOME/ocp4/ignition
auth  bootstrap.ign  master.ign  metadata.json  worker.ign
```

撰寫方面安裝 RHCOS 的腳本 `$HOME/ocp4/ignition/coreos-install.sh`:

```sh
#!/bin/bash

set -eu 

export DEVICE_NAME=${DEVICE_NAME:-/dev/sda}
export IGNITION_FILE=${IGNITION_FILE:-bootstrap.ign} # bootstrap.ign  master.ign  worker.ign 
export IGNITION_URL=${IGNITION_URL:-http://192.168.101.9:8080/ignition/${IGNITION_FILE}}

sudo coreos-installer install ${DEVICE_NAME} \
  --insecure-ignition \
  --ignition-url=${IGNITION_URL} \
  --firstboot-args rd.neednet=1 \
  --copy-network
```

複製 `*.ign` 與腳本 到 `/var/www/html/ignition` 目錄:

```sh
$ mkdir /var/www/html/ignition
$ cp -rp $HOME/ocp4/ignition/{*.ign,coreos-install.sh} /var/www/html/ignition
$ ls /var/www/html/ignition
bootstrap.ign  master.ign  worker.ign coreos-install.sh

$ chmod 664 -R /var/www/html/ignition/*
```

## Bootstrap
透過 PVE 建立 Bootstrap 節點，並掛載 RHCOS Live ISO，並進入 Console 執行以下操作。

首先進入系統後，先設定 Network 資訊:

```sh
$ sudo nmtui
```

設定網卡資訊:

![](https://i.imgur.com/tJFSkiO.png)

![](https://i.imgur.com/vAEtA2f.png)

![](https://i.imgur.com/qINQqLa.png)



&gt; 確保網卡 IP address 設定為 `xx.xx.xx.xx/24`，若沒有設定 Network mask 的話，就會用預設 `xx.xx.xx.xx/8`。

重新啟動網卡設定:

![](https://i.imgur.com/KMvs0UZ.png)


透過以下指令檢查設定結果:

```sh
$ ip -4 a
```
> 確認網路與 DNS 都能夠被解析到。

確認沒問題後，執行以下指令來安裝 RHCOS，並設定為 Bootstrap 節點:

```sh
$ curl http://192.168.101.9:8080/ignition/coreos-install.sh --output coreos-install.sh
$ chmod u+x coreos-install.sh
$ ./coreos-install.sh

# 安裝完成後
$ lsblk -f
$ reboot
```
> 安裝沒問題後，重新啟動機器，並將 CD-ROM 移除。

Bootstrap 開啟完成後，回到 `bastion` 節點執行以下指令來檢查部署狀況:

```sh
$ openshift-install wait-for bootstrap-complete --dir=$HOME/ocp4/ignition --log-level debug
DEBUG OpenShift Installer 4.8.2
DEBUG Built from commit a5ddd2dd6c72d8a5ea0a5f17acd8b964b6a3d1be
INFO Waiting up to 20m0s for the Kubernetes API at https://api.lab.yiylin.internal:6443...
INFO API v1.21.1+051ac4f up
INFO Waiting up to 30m0s for bootstrapping to complete...

# 確認 openshift-api-server 以及 machine-config-server LB 可以通
$ curl -k https://api-int.${DOMAIN}:6443
$ curl -k https://api-int.${DOMAIN}:22623/config/master | jq .
```

當看到此訊息時，就能並往下進行安裝 Masters。若等待過久，可以透過 bastion 進入 bootstrap 節點查看狀況:

```
$ ssh core@bootstrap.lab.yiylin.internal

core $ sudo crictl pods
core $ sudo crictl ps -a
core $ sudo journalctl -b -f -u bootkube.service
```

正常情況 Pods 要如以下:

```
POD ID              CREATED              STATE               NAME                                           NAMESPACE                             ATTEMPT             RUNTIME
8101e41b16f9e       52 seconds ago       Ready               bootstrap-kube-scheduler-localhost             kube-system                           0                   (default)
907d0a933771a       52 seconds ago       Ready               bootstrap-kube-controller-manager-localhost    kube-system                           0                   (default)
a8b709f7caeaa       52 seconds ago       Ready               bootstrap-kube-apiserver-localhost             kube-system                           0                   (default)
4eb793b3d1844       52 seconds ago       Ready               cloud-credential-operator-localhost            openshift-cloud-credential-operator   0                   (default)
49a1cdb2b25f9       52 seconds ago       Ready               bootstrap-cluster-version-operator-localhost   openshift-cluster-version             0                   (default)
c5098e42b5aa5       About a minute ago   Ready               bootstrap-machine-config-operator-localhost    default                               0                   (default)
e7a3dbc6e0bc0       About a minute ago   Ready               etcd-bootstrap-member-localhost                openshift-etcd                        0                   (default)
```

## Maters
透過 vCenter 建立 Master 節點，並掛載 RHCOS Live ISO，並進入 Console 執行以下操作。

首先進入系統後，先設定 Network 資訊:

```sh
$ sudo nmtui
```
&gt; 網路參考 Bootstrap 設定流程。

確認沒問題後，執行以下指令來安裝 RHCOS，並設定為 Masters 節點:

```sh
$ curl http://10.77.110.39:8080/ignition/coreos-install.sh --output coreos-install.sh
$ chmod u+x coreos-install.sh
$ export IGNITION_FILE=master.ign
$ ./coreos-install.sh

# 安裝完成後
$ lsblk -f
$ reboot
```
&gt; 安裝沒問題後，重新啟動機器，並將 CD-ROM 移除。

Masters 都開啟完成後，回到 `bastion` 節點執行以下指令來檢查部署狀況:

```sh
$ openshift-install wait-for bootstrap-complete --dir=$HOME/ocp4/ignition --log-level debug
...
INFO Waiting up to 30m0s for bootstrapping to complete...
DEBUG Bootstrap status: complete
INFO It is now safe to remove the bootstrap resources
INFO Time elapsed: 0s
```

在 `bastion` 執行以下指令來查看叢集狀況:

```sh
$ cp $HOME/ocp4/ignition/auth/kubeconfig $HOME/ocp4/kubeconfig
$ export KUBECONFIG=$HOME/ocp4/kubeconfig
$ oc get csr
NAME                                       AGE   SIGNERNAME                                    REQUESTOR                                                                         CONDITION
csr-9xqqd                                  25m   kubernetes.io/kubelet-serving                 system:node:etcd-1                                                                Approved,Issued
csr-gzvwr                                  27m   kubernetes.io/kubelet-serving                 system:node:etcd-0                                                                Approved,Issued
csr-j6npt                                  25m   kubernetes.io/kubelet-serving                 system:node:etcd-2                                                                Approved,Issued
csr-mxqkf                                  25m   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper         Approved,Issued
csr-tth9f                                  28m   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper         Approved,Issued
csr-zx9q8                                  25m   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper         Approved,Issued
system:openshift:openshift-authenticator   25m   kubernetes.io/kube-apiserver-client           system:serviceaccount:openshift-authentication-operator:authentication-operator   Approved,Issued

$ oc get no
NAME                           STATUS   ROLES    AGE   VERSION
master-1.lab.yiylin.internal   Ready    master   31m   v1.21.1+051ac4f
master-2.lab.yiylin.internal   Ready    master   15m   v1.21.1+051ac4f
master-3.lab.yiylin.internal   Ready    master   10m   v1.21.1+051ac4f
```

完成後，即可進行 Workers 與 Infras 部署，並移除 Bootstrap。若等待過久，可以透過 bastion 進入 masters 節點查看狀況:

```
$ ssh core@maste-1.lab.yiylin.internal
```

## Workers &amp; Infras
透過 vCenter 建立 Workers &amp; Infras 節點，並掛載 RHCOS Live ISO，並進入 Console 執行以下操作。

首先進入系統後，先設定 Network 資訊:

```sh
$ sudo nmtui
```
&gt; 網路參考 Bootstrap 設定流程。

確認沒問題後，執行以下指令來安裝 RHCOS，並設定為 Workers &amp; Infras 節點:

```sh
$ curl http://192.168.101.9:8080/ignition/coreos-install.sh --output coreos-install.sh
$ chmod u+x coreos-install.sh
$ export IGNITION_FILE=worker.ign
$ ./coreos-install.sh

# 安裝完成後
$ lsblk -f
$ reboot
```
> 安裝沒問題後，重新啟動機器，並將 CD-ROM 移除。

Workers &amp; Infras 都開啟完成後，就可以回到 `bastion` 節點執行以下指令來檢查部署狀況:

```sh
$ openshift-install wait-for install-complete --dir=$HOME/ocp4/ignition --log-level debug
DEBUG OpenShift Installer 4.8.2
DEBUG Built from commit a5ddd2dd6c72d8a5ea0a5f17acd8b964b6a3d1be
DEBUG Loading Install Config...
DEBUG   Loading SSH Key...
DEBUG   Loading Base Domain...
DEBUG     Loading Platform...
DEBUG   Loading Cluster Name...
DEBUG     Loading Base Domain...
DEBUG     Loading Platform...
DEBUG   Loading Networking...
DEBUG     Loading Platform...
DEBUG   Loading Pull Secret...
DEBUG   Loading Platform...
DEBUG Using Install Config loaded from state file
INFO Waiting up to 40m0s for the cluster at https://api.lab.yiylin.internal:6443 to initialize...
DEBUG Still waiting for the cluster to initialize: Some cluster operators are still updating: authentication, console, ingress, monitoring
```
> 出現 `DEBUG` 即可往下操作，因為這部分會需要下載 Images 會比較久。

開啟一個新的 Terminal，在 `bastion` 節點 Approve workers CSR:

```sh
$ export KUBECONFIG=$HOME/ocp4/kubeconfig
$ oc get csr -ojson | jq -r .items[] | select(.status == {} ) | .metadata.name | xargs oc adm certificate approve

# 確認機器都起來
$ oc get no
NAME       STATUS   ROLES    AGE   VERSION
infra0     Ready    worker   12m   v1.21.1+051ac4f
infra1     Ready    worker   12m   v1.21.1+051ac4f
infra2     Ready    worker   12m   v1.21.1+051ac4f
master0    Ready    master   11h   v1.21.1+051ac4f
master1    Ready    master   11h   v1.21.1+051ac4f
master2    Ready    master   11h   v1.21.1+051ac4f
worker0    Ready    worker   12m   v1.21.1+051ac4f
worker1    Ready    worker   12m   v1.21.1+051ac4f
worker2    Ready    worker   12m   v1.21.1+051ac4f
worker3    Ready    worker   12m   v1.21.1+051ac4f
```

### 搬移資源至 Infra nodes
接著要將 Worker 與 Infra 角色做拆分，確保系統與 Operator 服務不影響業務服務的負載，參考 [Creating infrastructure machine sets](https://docs.openshift.com/container-platform/4.8/machine_management/creating-infrastructure-machinesets.html#infrastructure-components_creating-infrastructure-machinesets)

設定 infra nodes 的 labels:

```sh
$ for i in {1..2}; do
  oc label node worker-${i} node-role.kubernetes.io/app= --overwrite
done

$ for i in {1..3}; do
    oc label node infra-${i} node-role.kubernetes.io/infra= --overwrite
    oc label node infra-${i} node-role.kubernetes.io/worker-
  done

$ oc get no
NAME                           STATUS   ROLES        AGE    VERSION
infra-1.lab.yiylin.internal    Ready    infra        31m    v1.21.1+051ac4f
infra-2.lab.yiylin.internal    Ready    infra        31m    v1.21.1+051ac4f
infra-3.lab.yiylin.internal    Ready    infra        31m    v1.21.1+051ac4f
master-1.lab.yiylin.internal   Ready    master       156m   v1.21.1+051ac4f
master-2.lab.yiylin.internal   Ready    master       140m   v1.21.1+051ac4f
master-3.lab.yiylin.internal   Ready    master       135m   v1.21.1+051ac4f
worker-1.lab.yiylin.internal   Ready    app,worker   31m    v1.21.1+051ac4f
worker-2.lab.yiylin.internal   Ready    app,worker   31m    v1.21.1+051ac4f
```

編輯 `Scheduler` 物件，並調整預設 `NodeSelector`:

```sh
$ oc edit scheduler cluster
...
spec:
  defaultNodeSelector: node-role.kubernetes.io/app= 
...
```

建立 Infra 機器的 Machine Config Pool 檔案:

```sh
$ cat &lt;&lt;EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: infra
spec:
  machineConfigSelector:
    matchExpressions:
    - key: machineconfiguration.openshift.io/role
      operator: In
      values:
      - worker
      - infra
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/infra:
EOF

$ oc get mcp
```
> 這時 Infra 節點會重新啟動。

搬移 Router 元件至 Infra 節點上:

```sh
$ oc edit IngressController default -n openshift-ingress-operator
...
spec:
  nodePlacement:
    nodeSelector:
      matchLabels:
        node-role.kubernetes.io/infra:
  replicas: 2
  
$ oc -n openshift-ingress get pod -o wide
```

搬移 Image Registry 元件至 Infra 節點上:

```sh
$ oc edit configs.imageregistry.operator.openshift.io/cluster
...
spec:
  nodeSelector:
    node-role.kubernetes.io/infra:
...

$ oc -n openshift-image-registry get pods -o wide 
```

搬移 Monitoring 元件至 Infra 節點上:

```sh
$ cat << "EOF" > oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |+
    alertmanagerMain:
      nodeSelector:
        node-role.kubernetes.io/infra:
    prometheusK8s:
      nodeSelector:
        node-role.kubernetes.io/infra:
    prometheusOperator:
      nodeSelector:
        node-role.kubernetes.io/infra:
    grafana:
      nodeSelector:
        node-role.kubernetes.io/infra:
    k8sPrometheusAdapter:
      nodeSelector:
        node-role.kubernetes.io/infra:
    kubeStateMetrics:
      nodeSelector:
        node-role.kubernetes.io/infra:
    telemeterClient:
      nodeSelector:
        node-role.kubernetes.io/infra:
    openshiftStateMetrics:
      nodeSelector:
        node-role.kubernetes.io/infra:
    thanosQuerier:
      nodeSelector:
        node-role.kubernetes.io/infra:
EOF

$ watch -n 0.1 oc -n openshift-monitoring get pod -o wide
```
&gt; 其他資源請參考 [Moving OpenShift Logging resources](https://docs.openshift.com/container-platform/4.8/machine_management/creating-infrastructure-machinesets.html#infrastructure-moving-logging_creating-infrastructure-machinesets) 搬移。

叢集初始化會需要一點時間，完成後就可以透過 Console 查看整體狀況。

## Web Console
透過 oc 取得 Web console 的 route URL:

```sh
$ oc -n openshift-console get route console -o=jsonpath={.spec.host}{\n}
console-openshift-console.apps.lab.yiylin.internal
```

取得 OpenShift 的 `kubeadmin` 密碼:

```sh
$ cat $HOME/ocp4/ignition/auth/kubeadmin-password
```

![](https://i.imgur.com/7ykeVUc.png)

---
## Terminal Login 


開啟一個新的 Terminal，在 bastion 節點配置 `KUBECONFIG`
```shell
$ export KUBECONFIG=$HOME/ocp4/kubeconfig
$ oc get no
```


## References
- https://github.com/openshift/machine-config-operator/blob/master/docs/custom-pools.md
- https://gist.github.com/kairen/894348c6281f33a981b528e69c3d06bf
- https://docs.openshift.com/container-platform/4.8/installing/installing_vsphere/installing-restricted-networks-vsphere.html
- [Networking Guide -  BIND (Berkeley Internet Name Domain)](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/networking_guide/sec-bind)
- [參考 IPI Prerequisites](https://docs.openshift.com/container-platform/4.8/installing/installing_bare_metal_ipi/ipi-install-prerequisites.html)

