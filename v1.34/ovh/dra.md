
# Instructions for accelerators requirements

- [Instructions for accelerators requirements](#instructions-for-accelerators-requirements)
  - [dra\_support](#dra_support)
    - [Prerequisites](#prerequisites)
    - [Procedure](#procedure)

## dra_support

### Prerequisites

* A MKS 1.34 cluster with at least 1 GPU node.

### Procedure

1. Install the GPU Operator

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update
helm install --create-namespace --namespace gpu-operator gpu-operator nvidia/gpu-operator
```

2. Install the nvidia device plugin

```bash
helm upgrade -i --create-namespace --namespace nvidia nvidia-dra-driver-gpu nvidia/nvidia-dra-driver-gpu \
  --set controller.affinity=null \
  --set gpuResourcesEnabledOverride=true \
  --set nvidiaDriverRoot=/run/nvidia/driver \
  --set kubeletPlugin.tolerations[0].key=nvidia.com/gpu \
  --set kubeletPlugin.tolerations[0].operator=Exists \
  --set kubeletPlugin.tolerations[0].effect=NoSchedule
```

Notes:
* In the future, this driver will be included in the NVIDIA GPU Operator directly.

3. Check for default device classes

```bash
kubectl get deviceclasses.resource.k8s.io 
NAME                                        AGE
compute-domain-daemon.nvidia.com            6m2s
compute-domain-default-channel.nvidia.com   6m2s
gpu.nvidia.com                              6m2s
mig.nvidia.com                              6m2s
```

4. Deploy the test job

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dra-gpu-example
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dra-gpu-example
  template:
    metadata:
      labels:
        app: dra-gpu-example
    spec:
      containers:
      - name: ctr
        image: ubuntu:24.04
        command: ["bash", "-c"]
        args: ["while [ 1 ]; do date; echo $(nvidia-smi -L || echo Waiting...); sleep 60; done"]
        resources:
          claims:
          - name: single-gpu
      resourceClaims:
      - name: single-gpu
        resourceClaimTemplateName: gpu-claim-template
      tolerations:
      - key: "nvidia.com/gpu"
        operator: "Exists"
        effect: "NoSchedule"
---
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: gpu-claim-template
spec:
  spec:
    devices:
      requests:
      - name: single-gpu
        exactly:
          deviceClassName: gpu.nvidia.com
          allocationMode: ExactCount
          count: 1
```

5. Check the logs of the test job


Expected output:

```bash
kubectl logs deployments/dra-gpu-example
Tue Dec 30 16:46:13 UTC 2025
GPU 0: Tesla V100-PCIE-16GB (UUID: GPU-f113d62f-398b-84ac-835b-63f08c5db5b6)
```
