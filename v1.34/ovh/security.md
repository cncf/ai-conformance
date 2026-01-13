
# Instructions for security requirements

- [Instructions for security requirements](#instructions-for-security-requirements)
  - [secure\_accelerator\_access](#secure_accelerator_access)
    - [Prerequisites](#prerequisites)
    - [Procedure](#procedure)
      - [Test 1: Test that DRA works as expected](#test-1-test-that-dra-works-as-expected)
      - [Test 2: Test that devices are mapped to the expected containers](#test-2-test-that-devices-are-mapped-to-the-expected-containers)

## secure_accelerator_access

### Prerequisites

* A MKS 1.34 cluster with at least 1 GPU node.
* The NVIDIA GPU operator and DRA device plugin installed and configured. See [dra.md](dra.md) for installation instructions.

### Procedure

#### Test 1: Test that DRA works as expected

1. Create the following kubernetes resources:

```yaml
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
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pytorch-cuda-check
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: pytorch-cuda-check
  template:
    metadata:
      labels:
        app: pytorch-cuda-check
    spec:
      containers:
        - name: pytorch-cuda-check
          image: nvcr.io/nvidia/pytorch:25.12-py3
          command: ["/bin/sh", "-c"]
          args:
            - |
              while true; do
                python3 -c 'import torch; print(str(torch.cuda.device_count()) + " GPUs available")'
                sleep 30
              done
          resources:
            claims:
              - name: single-gpu
      resourceClaims:
        - name: single-gpu
          resourceClaimTemplateName: gpu-claim-template

```

```bash
kubectl create -f /tmp/security.yaml
```

2. Check that the deployment is running and showing the expected output (1 GPU available)

```bash
kubectl logs -l app=pytorch-cuda-check
1 GPUs available
```

3. Remove the resource claim from the deployment

```bash
kubectl patch deployment pytorch-cuda-check -p '{"spec":{"template":{"spec":{"containers":[{"name":"pytorch-cuda-check","resources":null}]}}}}'
```

4. Check that the GPU is not detected anymore

```bash
kubectl logs -l app=pytorch-cuda-check
0 GPUs available
```

#### Test 2: Test that devices are mapped to the expected containers

This test is implemented in the e2e tests in the kubernetes repository.

   1. Clone the kubernetes repository

```bash
git clone --depth 1 --branch v1.34.1 https://github.com/kubernetes/kubernetes
cd kubernetes
```

2. Run the e2e test

```bash
go run github.com/onsi/ginkgo/v2/ginkgo -focus='must map configs and devices to the right containers' ./test/e2e
  I0105 15:51:33.792937 524920 test_context.go:565] The --provider flag is not set. Continuing as if --provider=skeleton had been used.
  I0105 15:51:33.793517  524920 e2e.go:109] Starting e2e run "5f34385f-b87a-45c7-8d1e-5afffc646ba2" on Ginkgo node 1
Running Suite: Kubernetes e2e suite - /home/mhubert/kubernetes/test/e2e
=======================================================================
Random Seed: 1767624672 - will randomize all specs

Will run 1 of 7137 specs
...

Ran 1 of 7137 Specs in 27.495 seconds
SUCCESS! -- 1 Passed | 0 Failed | 0 Pending | 7136 Skipped
PASS

Ginkgo ran 1 suite in 48.787260498s
Test Suite Passed
```