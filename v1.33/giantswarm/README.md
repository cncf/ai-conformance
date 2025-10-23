### Giant Swarm Platform

Giant Swarm Platform is a managed Kubernetes platform developed by [Giant Swarm](https://www.giantswarm.io).

### How to Reproduce

#### Create Cluster

First access the [Giant Swarm Platform](https://docs.giantswarm.io/getting-started/), and login to platform API.
After successful login, select [Create a  cluster]()  with the [specific values]().

# AI platform components

The following components should be installed to complete the AI setup:

## 1. NVIDIA GPU Operator

**Purpose**: Manages NVIDIA GPU resources in Kubernetes clusters.

**Installation via Giant Swarm App Platform**:
```sh
kubectl gs template app \
  --catalog giantswarm \
  --name gpu-operator \
  --cluster-name ai-tests \
  --target-namespace kube-system \
  --version 1.0.0 \
  --organization fer | kubectl apply -f -
```

## 2. NVIDIA DRA Driver GPU

**Purpose**: Provides Dynamic Resource Allocation (DRA) support for NVIDIA GPUs.

**Installation via Flux HelmRelease**:
```sh
# First create the NVIDIA Helm Repository
kubectl apply -f - <<EOF
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: nvidia
  namespace: org-fer
spec:
  interval: 1h
  url: https://helm.ngc.nvidia.com/nvidia
EOF

# Then create the HelmRelease
kubectl apply -f - <<EOF
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: ai-tests-nvidia-dra-driver-gpu
  namespace: org-fer
spec:
  interval: 5m
  chart:
    spec:
      chart: nvidia-dra-driver-gpu
      version: "25.3.0"
      sourceRef:
        kind: HelmRepository
        name: nvidia
  targetNamespace: kube-system
  kubeConfig:
    secretRef:
      name: ai-tests-kubeconfig
      key: value
  values:
    nvidiaDriverRoot: "/"
    resources:
      gpus:
        enabled: false
EOF
```

## 3. Kuberay Operator

**Purpose**: Manages Ray clusters for distributed AI/ML workloads.

**Installation via Giant Swarm App Platform**:
```sh
kubectl gs template app \
  --catalog giantswarm \
  --name kuberay-operator \
  --cluster-name ai-tests \
  --target-namespace kube-system \
  --version 1.0.0 \
  --organization fer | kubectl apply -f -
```

## 4. Kueue

**Purpose**: Provides job queueing and resource management for batch workloads.

**Installation via Flux HelmRelease**:
```sh
# First create the Kueue Helm Repository
kubectl apply -f - <<EOF
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: kueue
  namespace: org-fer
spec:
  interval: 1h
  type: oci
  url: oci://registry.k8s.io/kueue/charts
EOF

# Then create the HelmRelease
kubectl apply -f - <<EOF
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: ai-tests-kueue
  namespace: org-fer
spec:
  interval: 5m
  chart:
    spec:
      chart: kueue
      version: "0.14.1"
      sourceRef:
        kind: HelmRepository
        name: kueue
        namespace: org-fer
  targetNamespace: kube-system
  kubeConfig:
    secretRef:
      name: ai-tests-kubeconfig
      key: value
EOF
```

## 5. Gateway API CRDs

**Purpose**: Provides advanced traffic management capabilities for inference services.

**Installation via Giant Swarm App Platform**:
```sh
kubectl gs template app \
  --catalog giantswarm \
  --name gateway-api-crds \
  --cluster-name ai-tests \
  --target-namespace kube-system \
  --version 1.2.1 \
  --organization fer | kubectl apply -f -
```

## 6. AWS EFS CSI Driver

**Purpose**: Enables persistent storage using AWS Elastic File System for shared AI model storage.

**Installation via Giant Swarm App Platform**:
```sh
kubectl gs template app \
  --catalog giantswarm \
  --name aws-efs-csi-driver \
  --cluster-name ai-tests \
  --target-namespace kube-system \
  --version 2.1.5 \
  --organization fer | kubectl apply -f -
```

## 7. JobSet

**Purpose**: Manages sets of Jobs for distributed training workloads.

**Installation via Flux HelmRelease**:
```sh
kubectl apply -f - <<EOF
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: ai-tests-jobset
  namespace: org-fer
spec:
  interval: 5m
  chart:
    spec:
      chart: oci://registry.k8s.io/jobset/charts/jobset
      version: "0.10.1"
  targetNamespace: kube-system
  kubeConfig:
    secretRef:
      name: ai-tests-kubeconfig
      key: value
EOF
```

## 8. Kube Prometheus Stack

**Purpose**: Provides comprehensive monitoring and alerting for Kubernetes clusters.

**Installation via Giant Swarm App Platform**:
```sh
kubectl gs template app \
  --catalog giantswarm \
  --name kube-prometheus-stack \
  --cluster-name ai-tests \
  --target-namespace kube-system \
  --version 18.1.0 \
  --organization fer | kubectl apply -f -
```

## 9. Prometheus Adapter

**Purpose**: Enables custom metrics for Horizontal Pod Autoscaler, including AI/ML specific metrics.

**Installation via Flux HelmRelease**:
```sh
kubectl apply -f - <<EOF
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: ai-tests-prometheus-adapter
  namespace: org-fer
spec:
  interval: 5m
  chart:
    spec:
      chart: oci://ghcr.io/prometheus-community/charts/prometheus-adapter
  targetNamespace: kube-system
  kubeConfig:
    secretRef:
      name: ai-tests-kubeconfig
      key: value
  values:
    prometheus:
      url: http://kube-prometheus-stack-prometheus.kube-system.svc
    rules:
      default: true
      custom:
        - seriesQuery: 'vllm:num_requests_running'
          resources:
            overrides:
              namespace: {resource: "namespace"}
              pod: {resource: "pod"}
          name:
            matches: "vllm:num_requests_running"
            as: "vllm_num_requests_running"
          metricsQuery: sum(vllm:num_requests_running{<<.LabelMatchers>>}) by (<<.GroupBy>>)
EOF
```

## 10. Sonobuoy Configuration

**Purpose**: Applies PolicyExceptions and configurations needed for AI conformance testing.

**Installation**: Applied directly to the workload cluster using the kubeconfig:

```sh
# Download and apply the configuration
kubectl --kubeconfig=/path/to/workload-cluster-kubeconfig apply -f https://gist.githubusercontent.com/pipo02mix/80415c1182a5920af46a85c7adf90a8a/raw/d75d7593194fb2a3beba0549f946cb6f8a5a5f46/sonobuoy-rews.yaml
```

All these components work together to provide a complete AI/ML platform on Kubernetes with GPU support, workload management, monitoring, and conformance testing capabilities.

#### Run conformance Test by Sonobuoy

Login to the control-plane of the cluster created by Giant Swarm Platform.

Start the conformance tests:

```sh
sonobuoy run --plugin https://raw.githubusercontent.com/pipo02mix/ai-conformance/c0f5f45e131445e1cf833276ca66e251b1b200e9/sonobuoy-plugin.yaml
````

Monitor the conformance tests by tracking the sonobuoy logs, and wait for the line: "no-exit was specified, sonobuoy is now blocking"

```sh
stern -n sonobuoy sonobuoy
```

Retrieve result:

```sh
outfile=$(sonobuoy retrieve)
sonobuoy results $outfile
```
