---
title: "NVIDIA GPU Operator on OpenShift4"
slug:  "gpu-operator-on-ocp4"
url: "openshift/gpu-operator"
date: 2022-02-28T21:40:40+08:00
draft: false
description: "本篇介紹如何在 OpenShift 與 GPU節點上，透過GPU Operator 快速部署 GPU Driver"
categories:
  - OpenShift
tags:
  - GPU
  - OpenShift
---

Nvidia GPU Operator v1.9 on OpenShift 4.9.9 包含以上版本，安裝不用再進行額外權限配置]
## OpenShift 4.9.9 或更高的版本 [1]
針對 driver toolkit 取消必要安裝要求:
- Set up an entitlement
- Mirror the RPM packages in a disconnected environment
- Configure a proxy to access the package repository

[1] https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/openshift/steps-overview.html#entitlement-free-supported-versions

## OpenShift 4.9.8 與以下版本 [2]
需手動操作獲取 OCP 憑證，創建 MachineConfig 來認證 OCP 叢集，擴大授權 Images 使用權限範圍，來安裝 Nvidia Operator :
1. 從 Red Hat Customer Portal 下載 Red Hat OpenShift Container Platform 訂閱憑證 (啟用權限需要登入 OCP 憑證）。
2. 創建一個 MachineConfig 啟用訂閱管理平台並提供有效訂閱憑證。等待 MachineConfigOperator 重啟節點並完成 MachineConfig。
3. 驗證叢集所有節點更新權限是否正常。
> 補充 - NVIDIA GPU Operator 安裝會部署幾個 Pod 服務，用於管理和啟用 GPU 在 OpenShift 中運作。其中一些 Pod 需要 OpenShift 使用一些非 Universal Base Image (UBI)  默認授權的 Images。必須在 OpenShift Cluster 中啟用信任的授權 Images，來啟動 NVIDIA GPU 驅動程式的容器運行。

[1] https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/openshift/steps-overview.html#entitlement-free-supported-versions
[2] https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/openshift/cluster-entitlement.html#enabling-a-cluster-wide-entitlemenent

## Install Operator - Node Feature Discovery

![](https://i.imgur.com/3J8zoZU.png)
![](https://i.imgur.com/NDepxYJ.png)


```bash=
[lab-user@bastion ~]$ oc get no
NAME                                         STATUS   ROLES    AGE     VERSION
ip-10-0-135-51.us-east-2.compute.internal    Ready    worker   3h5m    v1.22.3+e790d7f
ip-10-0-142-219.us-east-2.compute.internal   Ready    master   3h14m   v1.22.3+e790d7f
ip-10-0-167-35.us-east-2.compute.internal    Ready    worker   3h5m    v1.22.3+e790d7f
ip-10-0-186-251.us-east-2.compute.internal   Ready    master   3h14m   v1.22.3+e790d7f
ip-10-0-213-103.us-east-2.compute.internal   Ready    master   3h14m   v1.22.3+e790d7f
```


![](https://i.imgur.com/456JizI.png)


- 要驗證實例是否已創建，請運行：

```
[lab-user@bastion ~]$ oc get pods -n openshift-nfd
NAME                                      READY   STATUS    RESTARTS   AGE
nfd-controller-manager-6f65f47cf6-tg6gj   2/2     Running   0          24m
nfd-master-d7cqw                          1/1     Running   0          35s
nfd-master-j42m9                          1/1     Running   0          35s
nfd-master-r64nv                          1/1     Running   0          35s
nfd-worker-24tzn                          1/1     Running   0          35s
nfd-worker-5rsg2                          1/1     Running   0          35s
```

- 成功的部署會顯示一個Running狀態。


## Installing the NVIDIA GPU Operator

With the Node Feature Discovery Operator installed you can continue with the final step and install the NVIDIA GPU Operator.

As a cluster administrator, you can install the NVIDIA GPU Operator using the OpenShift Container Platform CLI or the web console.

![](https://i.imgur.com/zSZnOQc.png)
![](https://i.imgur.com/H5mLoWH.png)
![](https://i.imgur.com/gV3G6kq.png)


```bash=
kind: ClusterPolicy
apiVersion: nvidia.com/v1
metadata:
  name: gpu-cluster-policy
spec:
  dcgmExporter:
    config:
      name: ''
  dcgm:
    enabled: true
  daemonsets: {}
  devicePlugin: {}
  driver:
    enabled: true
    use_ocp_driver_toolkit: true
    repoConfig:
      configMapName: ''
    certConfig:
      name: ''
    licensingConfig:
      nlsEnabled: false
      configMapName: ''
    virtualTopology:
      config: ''
  gfd: {}
  migManager:
    enabled: true
  nodeStatusExporter:
    enabled: true
  operator:
    defaultRuntime: crio
    deployGFD: true
    initContainer: {}
  mig:
    strategy: single
  toolkit:
    enabled: true
  validator:
    plugin:
      env:
        - name: WITH_WORKLOAD
          value: 'true'

```

Create the ClusterPolicy custom resource. This CRD will create several OCP resources. It will evaluate all the labels for the each node in the cluster and look for this:


![](https://i.imgur.com/k4xdsx8.png)


![](https://i.imgur.com/45ygkxL.png)

```
$ oc project nvidia-gpu-operator 
$ oc get pod -o wide
```

## Validating the GPU availability

```bash=
[lab-user@bastion ~]$ oc get pod | grep nvidia-device-plugin-daemonset
nvidia-device-plugin-daemonset-bspfh                 1/1     Running     0          21m
nvidia-device-plugin-daemonset-n62dm                 1/1     Running     0          21m
[lab-user@bastion ~]$ oc exec -ti nvidia-device-plugin-daemonset-bspfh -- nvidia-smi
Defaulted container "nvidia-device-plugin-ctr" out of: nvidia-device-plugin-ctr, toolkit-validation (init)
Fri Jan 28 06:47:21 2022
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 470.82.01    Driver Version: 470.82.01    CUDA Version: 11.4     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Tesla V100-SXM2...  On   | 00000000:00:1E.0 Off |                    0 |
| N/A   32C    P0    23W / 300W |      0MiB / 16160MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+

...
...
$ oc exec -ti nvidia-device-plugin-daemonset-n62dm -- nvidia-smi
Defaulted container "nvidia-device-plugin-ctr" out of: nvidia-device-plugin-ctr, toolkit-validation (init)
Fri Jan 28 06:47:44 2022
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 470.82.01    Driver Version: 470.82.01    CUDA Version: 11.4     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Tesla V100-SXM2...  On   | 00000000:00:1E.0 Off |                    0 |
| N/A   26C    P0    24W / 300W |      0MiB / 16160MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

## Running a sample GPU Application

Run a simple CUDA VectorAdd sample, which adds two vectors together to ensure the GPUs have bootstrapped correctly.


1. Run the following:

```
cat << EOF | oc create -f -

apiVersion: v1
kind: Pod
metadata:
  name: cuda-vectoradd
spec:
 restartPolicy: OnFailure
 containers:
 - name: cuda-vectoradd
   image: "nvidia/samples:vectoradd-cuda11.2.1"
   resources:
     limits:
       nvidia.com/gpu: 1
EOF
pod/cuda-vectoradd created
```

2. Check the logs of the container:

```
[lab-user@bastion ~]$ oc logs cuda-vectoradd
[Vector addition of 50000 elements]
Copy input data from the host memory to the CUDA device
CUDA kernel launch with 196 blocks of 256 threads
Copy output data from the CUDA device to the host memory
Test PASSED
Done
```

## Getting information about the GPU¶

The nvidia-smi shows memory usage, GPU utilization and the temperature of the GPU. Test the GPU access by running the popular nvidia-smi command within the pod.

To view GPU utilization, run nvidia-smi from a pod in the GPU Operator daemonset.

1. Change to the nvidia-gpu-operator project:

```
$ oc project nvidia-gpu-operator
```

2. Run the following command to view these new pods:

```
$ oc get pod -owide -lopenshift.driver-toolkit=true 
```


```
NAME                                                 READY   STATUS    RESTARTS   AGE   IP             NODE                                        NOMINATED NODE   READINESS GATES
nvidia-driver-daemonset-49.84.202201102104-0-gl557   2/2     Running   0          26m   10.131.0.106   ip-10-0-167-35.us-east-2.compute.internal   <none>           <none>
nvidia-driver-daemonset-49.84.202201102104-0-k9sg5   2/2     Running   0          26m   10.128.2.17    ip-10-0-135-51.us-east-2.compute.internal   <none>           <none>
```

3. Run the nvidia-smi command within the pod: 

```
$ oc exec -it nvidia-driver-daemonset-48.84.202110270303-0-9df9j -- nvidia-smi
```
