
# Instructions for schedulingOrchestration requirements

- [Instructions for schedulingOrchestration requirements](#instructions-for-schedulingorchestration-requirements)
  - [gang\_scheduling](#gang_scheduling)
    - [Prerequisites](#prerequisites)
    - [Procedure](#procedure)
  - [pod\_autoscaling](#pod_autoscaling)
    - [Prerequisites](#prerequisites-1)
    - [Procedure](#procedure-1)


## gang_scheduling

### Prerequisites

* A MKS 1.34 cluster with at least 1 node.

### Procedure

1. Install volcano

Instructions from https://volcano.sh/en/docs/installation/

```bash
helm repo add volcano-sh https://volcano-sh.github.io/helm-charts

helm repo update

helm install volcano volcano-sh/volcano -n volcano-system --create-namespace
```

2. Create the following sample job demonstrating gang scheduling capabilities:

```yaml
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  name: lavaflow-training
spec:
  minAvailable: 3   # gang size
  schedulerName: volcano
  queue: default
  tasks:
    - name: master
      replicas: 1
      template:
        spec:
          containers:
            - name: master
              image: busybox
              command: ["sh", "-c", "echo I am the lava master; sleep 10"]
          restartPolicy: Never

    - name: worker
      replicas: 2
      template:
        spec:
          containers:
            - name: worker
              image: busybox
              command: ["sh", "-c", "echo I am a lava worker; sleep 10"]
          restartPolicy: Never
```

3. Watch the event until the completion of the vulcano job:

```
...
  Warning  Unschedulable  17s                volcano  0/0 tasks in gang unschedulable: pod group is not ready, 3 minAvailable
  Warning  Unschedulable  16s                volcano  0/3 tasks in gang unschedulable: pod group is not ready, 3 Binding, 3 minAvailable
  Normal   Scheduled      12s (x3 over 15s)  volcano  pod group is ready
...
```

```bash
kubectl get vcjob,podgroup
NAME                                     STATUS      MINAVAILABLE   RUNNINGS   AGE
job.batch.volcano.sh/lavaflow-training   Completed   3                         110s
```

## pod_autoscaling

### Prerequisites

* A MKS 1.34 cluster with at least 1 node.
* The NVIDIA dynamic resource allocation (DRA) driver, see [dra.md](./dra.md).
* The NVIDIA DGCM metrics exporter and a prometheus stack. See [the associated guide](https://help.ovhcloud.com/csm/fr-public-cloud-kubernetes-monitoring-gpu-application?id=kb_article_view&sysparm_article=KB0055257).

### Procedure

1. Deploy the following resources:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: gpu-intensive-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: gpu-intensive-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Pods
      pods:
        metric:
          name: DCGM_FI_DEV_GPU_UTIL
        target:
          type: AverageValue
          averageValue: 5

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-intensive-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gpu-intensive
  template:
    metadata:
      labels:
        app: gpu-intensive
    spec:
      containers:
      - name: gpu-intensive-container
        image: k8s.gcr.io/cuda-vector-add:v0.1
        command: ["/bin/bash", "-c"]
        args: ["while true; do ./vectorAdd; done"]
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
---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: gpu-task-utilization
spec:
  groups:
    - name: gpu-rule-group
      rules:
        - record: cuda_gpu
          expr: DCGM_FI_DEV_GPU_UTIL
          labels:
            namespace: default
            deployment: gpu-intensive-app
```

2. Watch the deployment scale up to its maxpods configuration:

```
NAME                                                    REFERENCE                      TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/cpu-intensive-app   Deployment/cpu-intensive-app   cpu: 196%/50%   1         10        10         12m
```