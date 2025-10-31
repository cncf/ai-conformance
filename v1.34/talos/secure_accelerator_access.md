# Omni Secure Accelerator Access

**MUST**: Ensure that access to accelerators from within containers is properly isolated and mediated by the Kubernetes resource management framework (device plugin or DRA) and container runtime, preventing unauthorized access or interference between workloads.

## Tests

### Test 1: Verify Isolated GPU Access via Device Plugin

**Step 1**: Prepare the test environment, including:

- Create a Kubernetes 1.33 cluster
- [Add a GPU node and install the NVIDIA device plugin](https://docs.siderolabs.com/talos/v1.11/configure-your-talos-cluster/hardware-and-drivers/nvidia-gpu-proprietary)

**Step 2 [Accessible]**: Deploy a Pod on a node with available accelerator(s), and ensure the container within the Pod explicitly requests accelerator resources.

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: gpu-accessible
  namespace: kube-system
spec:
  containers:
  - name: cuda-container
    image: nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda11.7.1-ubuntu20.04
    command: ["sleep", "3600"]
    resources:
      limits:
        nvidia.com/gpu: 1
  restartPolicy: Never
  runtimeClassName: nvidia
EOF
```

Validate the GPU is available

```bash
kubectl exec gpu-test-accessible -- nvidia-smi -L
GPU 0: Quadro P1000 (UUID: GPU-ca8f98c8-e647-af32-3402-a255bc380664)
```

**Expected Result**: The command should successfully return the GPU information.

**Step 3 [Isolation]**: Deploy two Pods on the same node, each requesting different GPU resources. Verify that only 1 pod is scheduled if only 1 GPU is available on the node or each pod only has access to the GPU it requested.

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod1
  namespace: kube-system
spec:
  containers:
  - name: cuda-container
    image: nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda11.7.1-ubuntu20.04
    command: ["sleep", "3600"]
    resources:
      limits:
        nvidia.com/gpu: 1
    env:
    - name: CUDA_VISIBLE_DEVICES
      value: "0"
  restartPolicy: Never
  runtimeClassName: nvidia
EOF
```
Deploy second workload

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata
  name: gpu-pod2
  namespace: kube-system
spec:
  containers:
  - name: cuda-container
    image: nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda11.7.1-ubuntu20.04
    command: ["sleep", "3600"]
    resources:
      limits:
        nvidia.com/gpu: 1
    env:
    - name: CUDA_VISIBLE_DEVICES
      value: "1"
  restartPolicy: Never
  runtimeClassName: nvidia
EOF
```

Verify isolation - each Pod should only see its allocated GPU

```bash
kubectl exec gpu-pod1 -- nvidia-smi -L
GPU 0: Quadro P1000 (UUID: GPU-ca8f98c8-e647-af32-3402-a255bc380664)
```

If only 1 GPU is available the second pod will not be scheduled.
```bash
kubectl describe po gpu-pod2 | grep FailedScheduling
Warning  FailedScheduling  10m    default-scheduler  0/3 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 2 Insufficient nvidia.com/gpu. no new claims to deallocate, preemption: 0/3 nodes are available: 1 No preemption victims found for incoming pod, 2 Preemption is not helpful for scheduling.
```

If 2 GPUs are available on the node then the 2nd pod shows the GPU it has access to.
```bash
kubectl exec gpu-pod2 -- nvidia-smi -L
GPU 0: Quadro P1000 (UUID: GPU-4b85a003-881c-4eb8-bea2-014b0ed809d1)
```

**Expected Result**: Each Pod should only see and be able to access its allocated GPU. The CUDA_VISIBLE_DEVICES environment variable and the device plugin should ensure proper isolation between workloads.

### Test 2: Verify Unauthorized Access Prevention

**Step 1**: Deploy a Pod without GPU resource requests and verify that it cannot access GPU devices directly.

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: gpu-unauthorized
  namespace: default
spec:
  containers:
  - name: cuda-container
    image: nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda11.7.1-ubuntu20.04
    command: ["sleep", "3600"]
  restartPolicy: Never
EOF
```

Attempt to access GPU - this should fail

```bash
kubectl exec gpu-unauthorized -- nvidia-smi
OCI runtime exec failed: exec failed: unable to start container process: exec: "nvidia-smi": executable file not found in $PATH: unknown.
```

**Expected Result**: The Pod without GPU resource requests should not have access to GPU devices. The nvidia-smi command should fail, demonstrating that the device plugin properly mediates access.

**Step 2**: Verify that containers cannot bypass the device plugin by accessing GPU devices directly through device files.

```bash
kubectl exec gpu-unauthorized -- ls -la /dev/nvidia*
ls: cannot access '/dev/nvidia0': No such file or directory
ls: cannot access '/dev/nvidia-caps': No such file or directory
ls: cannot access '/dev/nvidiactl': No such file or directory
ls: cannot access '/dev/nvidia-modeset': No such file or directory
ls: cannot access '/dev/nvidia-uvm': No such file or directory
ls: cannot access '/dev/nvidia-uvm-tools': No such file or directory
command terminated with exit code 2
```

Verify that the container runtime has not mounted GPU devices

```bash
$ kubectl exec gpu-test-unauthorized -- ls -la /dev/ | grep nvidia
# empty
```

**Expected Result**: GPU device files should not be available in containers that haven't requested GPU resources through the Kubernetes resource management framework.
