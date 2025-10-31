# Secure Accelerator Access Tests

**MUST**: Ensure that access to accelerators from within containers is properly isolated and mediated by the Kubernetes resource management framework (device plugin or DRA) and container runtime, preventing unauthorized access or interference between workloads.

## Tests

### Test 1: Verify Isolated GPU Access via Device Plugin

**Step 1**: Prepare the test environment, including:

- Creating a Kubernetes 1.33 cluster
- [Adding a GPU node pool and install the NVIDIA GPU Operator](https://docs.giantswarm.io/tutorials/fleet-management/cluster-management/gpu/)

**Step 2 [Accessible]**: Deploy a Pod on a node with available accelerator(s), and ensure the container within the Pod explicitly requests accelerator resources. Inside the running container, execute a command to detect the accelerator device. This command should succeed and output the model of the accelerator device currently used by the container.

```bash
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test-accessible
  namespace: default
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
  tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
EOF

$ kubectl wait --for=condition=Ready pod/gpu-test-accessible --timeout=300s

$ kubectl exec gpu-test-accessible -- nvidia-smi --query-gpu=name --format=csv,noheader
Tesla T4
```

**Expected Result**: The command should successfully return the GPU model name, confirming that the container has proper access to the GPU through the device plugin framework.

**Step 3 [Isolation]**: Deploy two Pods on the same node, each requesting different GPU resources (if multiple GPUs are available) or the same GPU with resource limits. Verify that each Pod can only access its allocated GPU resources and cannot interfere with the other Pod's GPU access.

```shell
# Deploy first Pod requesting GPU 0
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test-pod1
  namespace: default
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
  tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
EOF

# Deploy second Pod requesting GPU 1 (if available)
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test-pod2
  namespace: default
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
  tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
EOF

$ kubectl wait --for=condition=Ready pod/gpu-test-pod1 --timeout=300s
$ kubectl wait --for=condition=Ready pod/gpu-test-pod2 --timeout=300s

# Verify isolation - each Pod should only see its allocated GPU
$ kubectl exec gpu-test-pod1 -- nvidia-smi -L
GPU 0: Tesla T4 (UUID: GPU-dabc57c1-250b-2979-2b6a-7fd7d9574143)

$ kubectl exec gpu-test-pod2 -- nvidia-smi -L
GPU 0: Tesla T4 (UUID: GPU-18705848-fd64-920c-22c5-e2f1a3d5a7c1)

# Verify that each Pod cannot access the other's GPU context
$ kubectl exec gpu-test-pod1 -- nvidia-smi --query-compute-apps=pid,process_name,gpu_uuid --format=csv
```

**Expected Result**: Each Pod should only see and be able to access its allocated GPU. The CUDA_VISIBLE_DEVICES environment variable and the device plugin should ensure proper isolation between workloads.

### Test 2: Verify Unauthorized Access Prevention

**Step 1**: Deploy a Pod without GPU resource requests and verify that it cannot access GPU devices directly.

```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test-unauthorized
  namespace: default
spec:
  containers:
  - name: cuda-container
    image: nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda11.7.1-ubuntu20.04
    command: ["sleep", "3600"]
    # Note: No GPU resources requested
  restartPolicy: Never
  tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
EOF

$ kubectl wait --for=condition=Ready pod/gpu-test-unauthorized --timeout=300s

# Attempt to access GPU - this should fail
$ kubectl exec gpu-test-unauthorized -- nvidia-smi
OCI runtime exec failed: exec failed: unable to start container process: exec: "nvidia-smi": executable file not found in $PATH: unknown.
```

**Expected Result**: The Pod without GPU resource requests should not have access to GPU devices. The nvidia-smi command should fail, demonstrating that the device plugin properly mediates access.

**Step 2**: Verify that containers cannot bypass the device plugin by accessing GPU devices directly through device files.

```shell
# Check if GPU device files are accessible
$ kubectl exec gpu-test-unauthorized -- ls -la /dev/nvidia*
ls: cannot access '/dev/nvidia*': No such file or directory

# Verify that the container runtime has not mounted GPU devices
$ kubectl exec gpu-test-unauthorized -- ls -la /dev/ | grep nvidia
# Should return empty
```

**Expected Result**: GPU device files should not be available in containers that haven't requested GPU resources through the Kubernetes resource management framework.
