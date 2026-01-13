
# Instructions for operator requirements

- [Instructions for operator requirements](#instructions-for-operator-requirements)
  - [robust\_controller](#robust_controller)
    - [Prerequisites](#prerequisites)
    - [Procedure](#procedure)

## robust_controller

### Prerequisites

* A MKS 1.34 cluster with at least one worker node.
* The NVIDIA GPU operator installed and configured.


### Procedure

Source: [Official Ray Documentation](https://docs.ray.io/en/releases-2.53.0/cluster/kubernetes/getting-started.html)

1. Install the Operator using helm

```bash
helm repo add kuberay https://ray-project.github.io/kuberay-helm/
helm repo update

helm install kuberay-operator kuberay/kuberay-operator --version 1.5.1
```

2. Check that the operator pod is running.

```bash
kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
kuberay-operator-56bcbffd6b-vms85   1/1     Running   0          17s
```

3. Deploy a Ray job to validate the installation.

```bash
kubectl logs -l=job-name=rayjob-sample
2026-01-05 01:09:00,628 INFO worker.py:1694 -- Connecting to existing Ray cluster at address: 10.2.0.42:6379...
2026-01-05 01:09:00,643 INFO worker.py:1879 -- Connected to Ray cluster. View the dashboard at 10.2.0.42:8265 
test_counter got 1
test_counter got 2
test_counter got 3
test_counter got 4
test_counter got 5
2026-01-05 01:09:08,704 SUCC cli.py:65 -- -----------------------------------
2026-01-05 01:09:08,704 SUCC cli.py:66 -- Job 'rayjob-sample-rkx9b' succeeded
2026-01-05 01:09:08,704 SUCC cli.py:67 -- -----------------------------------
```