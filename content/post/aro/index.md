---
title: "從零開始快速建置 - Microsoft Azure Red Hat OpenShift (ARO)"
slug:  "aro"
url: "openshift/aro"
date: 2022-03-13T21:40:40+08:00
draft: false
description: "本篇介紹如何在 Azure 上快速配置ARO4，並透過指定參數與 script 快速建置OCP，過程還測試更換指定域名，分享測試過程經驗"
categories:
  - Cloud Service Provider
tags:
  - Azure
  - OpenShift
  - ARO
---

ARO4 深入探討 - Microsoft Azure Red Hat OpenShift 4

讓我們深入挖掘！

Microsoft Azure Red Hat OpenShift 服務支持部署完全託管的 OpenShift 集群。

Azure Red Hat OpenShift 由 Red Hat 和 Microsoft 聯合設計、運營和支持，以提供集成的支持體驗。沒有虛擬機可以運行，也不需要打補丁。主節點、基礎架構和應用程序節點由 Red Hat 和 Microsoft 代表您進行修補、更新和監控。您的 Azure Red Hat OpenShift 集群已部署到您的 Azure 訂閱中，並包含在您的 Azure 賬單中。

在 OpenShift 4 上部署 Azure Red Hat 時，整個集群都包含在一個虛擬網絡中。在這個虛擬網絡中，您的主節點和工作節點都位於各自的子網中。每個子網都使用一個內部負載均衡器和一個公共負載均衡器。

這是有關 Azure Red Hat OpenShift 4 的官方圖表（可在 ARO4 Microsoft 頁面中找到）：


![](https://rcarrata.com/images/aro4-networking-diagram.png)


- 關於 ARO4 部署和管理的網絡和資源的更多詳細信息，請查看[ARO 圖詳細信息 - 官方文檔](https://docs.microsoft.com/en-us/azure/openshift/concepts-networking#networking-components)

讓我們安裝我們的第一個 ARO4 集群！

## Azure 帳戶先決條件
首先，我們需要在 Azure 帳戶中設置幾項內容，例如生成 ServicePrincipals、增加限制以及定義要使用的區域。
- 按照[配置 Azure 帳戶先決條件](https://docs.openshift.com/container-platform/latest/installing/installing_azure/installing-azure-account.html)來定義和分配適當的 RBAC 權限並增加對 Azure 帳戶的限制。


## 為 ARO 安裝配置 Azure 基礎結構先決條件

當我們準備好上一步後，就可以在 Azure 中為我們的 ARO4 集群生成基礎資源先決條件了：

- 定義 ARO4 安裝的基本參數：
```
export LOCATION=eastus
export RESOURCEGROUP=aro-rg
export CLUSTER=rcarrata
export VNET_CIDR="10.0.0.0/22"
export MASTER_SUBNET_CIDR="10.0.0.0/23"
export WORKER_SUBNET_CIDR="10.0.2.0/23"
```
- 使用 az cli 登錄到 Azure：

```
az login
```
注意：當登錄彈出時，您需要在 Azure Dashboard 中使用您的憑據進行身份驗證。

- 註冊資源提供者：

```
az provider register -n Microsoft.RedHatOpenShift --wait
az provider register -n Microsoft.Compute --wait
az provider register -n Microsoft.Storage --wait
```


# 建立 Azure Red Hat OpenShift 4 叢集

## 開始之前

Azure Red Hat OpenShift 至少需要 40 個核心，才能建立和執行 OpenShift 叢集。 新 Azure 訂用帳戶的預設 Azure 資源配額不符合這項需求。  若要要求增加資源限制，請參閱標準配額：[VM 系列的增加限制](https://docs.microsoft.com/zh-tw/azure/azure-portal/supportability/per-vm-quota-requests)。

- 例如，若要檢查最小支援的虛擬機器系列 SKU 「標準 DSv3」的目前訂用帳戶配額：

```
LOCATION=eastus
az vm list-usage -l $LOCATION \
--query "[?contains(name.value, 'standardDSv3Family')]" \
-o table
```

```
CurrentValue    Limit    LocalName
--------------  -------  --------------------------
40              1000     Standard DSv3 Family vCPUs
```


## 驗證權限

```
Azure CLI quickstart:
export GUID=hzdnk
export CLIENT_ID=bdf5480a-c661-451f-a2ce-81e448ce0ba8
export PASSWORD=Zz842xzC.7GWZS_xVZ3oQvg1~7PtoIcKVA
export TENANT=1ce7852f-dcf3-42bc-afe6-3bf81ab984fb
export SUBSCRIPTION=ede7f891-835c-4128-af5b-0e53848e54e7
export RESOURCEGROUP=openenv-hzdnk

curl -L https://aka.ms/InstallAzureCli | bash
az login --service-principal -u $CLIENT_ID -p $PASSWORD --tenant $TENANT

[
  {
    "cloudName": "AzureCloud",
    "homeTenantId": "1ce7852f-dcf3-42bc-afe6-3bf81ab984fb",
    "id": "ede7f891-835c-4128-af5b-0e53848e54e7",
    "isDefault": true,
    "managedByTenants": [
      {
        "tenantId": "b5ce0030-ec42-4a62-bc94-3025993e790c"
      }
    ],
    "name": "RHPDS Subscription - OpenTLC Tenant",
    "state": "Enabled",
    "tenantId": "1ce7852f-dcf3-42bc-afe6-3bf81ab984fb",
    "user": {
      "name": "bdf5480a-c661-451f-a2ce-81e448ce0ba8",
      "type": "servicePrincipal"
    }
  }
]
```




- 創建資源組為 ARO4 對象（如 vnet 和子網，以及自己的 ARO4 對象）分配資源：
```
az group create --name $RESOURCEGROUP --location $LOCATION
```

- 創建虛擬網絡：

```
az network vnet create --resource-group $RESOURCEGROUP \
  --name aro-vnet --address-prefixes $VNET_CIDR
```

- 為主節點添加一個空子網：

```
az network vnet subnet create \
    --resource-group $RESOURCEGROUP \
    --vnet-name aro-vnet \
    --name master-subnet \
    --address-prefixes $MASTER_SUBNET_CIDR \
    --service-endpoints Microsoft.ContainerRegistry
```


- 為工作節點添加一個空子網：

```
az network vnet subnet create \
    --resource-group $RESOURCEGROUP \
    --vnet-name aro-vnet \
    --name worker-subnet \
    --address-prefixes $WORKER_SUBNET_CIDR \
    --service-endpoints Microsoft.ContainerRegistry
```

- 在主子網上禁用子網專用終結點策略：

```
az network vnet subnet update \
    --name master-subnet \
    --resource-group $RESOURCEGROUP \
    --vnet-name aro-vnet \
    --disable-private-link-service-network-policies true
```

## 安裝 ARO4

- 使用 azure cli 創建 ARO 集群：
```
echo "Creating ARO Cluster... Please wait 40mins"
az aro create --resource-group $RESOURCEGROUP \
    --name $CLUSTER --vnet aro-vnet  \
    --master-subnet master-subnet \
    --worker-subnet worker-subnet \
    --pull-secret @pull-secret.txt
```
注意：安裝需要一個有效的 pull-secret。請訪問您的[Cloud OpenShift Doshboard](https://rcarrata.com/openshift/aro4/cloud.redhat.com/openshift/)並獲取您的 pull-secret 令牌。

- 然後 az aro cli 將在 Azure Dashboard 中預配一個 ARO 對象：

![](https://i.imgur.com/I1PyWMK.png)

- 自動地，它使用用於安裝的 Azure 對象創建了一個額外的 Azure 資源組，由 RH 和 MSFT 的 ARO SRE 管理：

![](https://i.imgur.com/wjJ4OUi.png)

- 在此資源組中，我們生成了提供和配置 ARO4 集群所需的資源：

![](https://rcarrata.com/images/aro4_3.png)

## 訪問 API 和控制台
- 大約 40 分鐘後。我們將使用控制台和 API 準備好 ARO4 集群：

[](https://rcarrata.com/images/aro4_4.png)

- 要訪問集群，請列出集群的憑據：

```
echo "List credentials for ARO Cluster"
az aro list-credentials \
    --name $CLUSTER \
    --resource-group $RESOURCEGROUP
echo ""
```

- 要顯示 ARO4 控制台：

```
echo "List console for ARO cluster"
az aro show \
    --name $CLUSTER \
    --resource-group $RESOURCEGROUP \
    --query "consoleProfile.url" -o tsv
```

- 檢查 ARO4 API：

```
apiServer=$(az aro show -g $RESOURCEGROUP -n $CLUSTER --query apiserverProfile.url -o tsv)
echo "This is the API for your cluster: $apiServer"
```

- 查詢 ARO Cluster 狀態

```
[root@bastion ~]# az aro list -o table
Name       ResourceGroup    Location    ProvisioningState    WorkerCount    URL
---------  ---------------  ----------  -------------------  -------------  -----------------------------------------------------------------
aro-demo  openenv-hdpnp    eastus      Succeeded            3              https://console-openshift-console.apps.yylin.io/
```
