# Omni AI Kubernetes cluster

[Omni](https://siderolabs.com/omni) allows you to create a Kubernetes cluster with any machine in any environment. For this guide you will need to provide your own machine with an NVIDIA accelerator connected to Omni.

You will also need an Omni instance which you can sign up for at [Omni SaaS](https://siderolabs.com/omni-signup)

## Create a cluster

Follow [the documentation](https://docs.siderolabs.com/omni/omni-cluster-setup/registering-machines/register-machines-with-omni) to connect at least 1 machine to Omni.

Identify the machine with an AI accelerator to use as a single node cluster.

```
omnictl get machines
```
Export the machine as a variable for a cluster template.

```bash
export MACHINE_UUID=0564bde4-47b8-c3bd-79e6-63e7a2badf27
```

Create a machine template with necessary patches.

```bash
cat << EOF > cluster-template.yaml
kind: Cluster
name: ai-conformance
kubernetes:
  version: v1.34.1
talos:
  version: v1.11.3
patches:
  - idOverride: 500-10f35187-7376-4ff5-bcf5-cf1033cb2095
    annotations:
      name: "nvidia modules and runtime"
    inline:
      cluster:
        allowSchedulingOnControlPlanes: true
        inlineManifests:
          - contents: |-
              apiVersion: node.k8s.io/v1
              kind: RuntimeClass
              metadata:
                name: nvidia
              handler: nvidia
            name: nvidia-runtime
      machine:
        kernel:
          modules:
            - name: nvidia
            - name: nvidia_uvm
            - name: nvidia_drm
            - name: nvidia_modeset
        sysctls:
          net.core.bpf_jit_harden: 1
---
kind: ControlPlane
machines:
  - ${MACHINE_UUID}
---
kind: Workers
---
kind: Machine
systemExtensions:
  - siderolabs/nonfree-kmod-nvidia-production
  - siderolabs/nvidia-container-toolkit-production
name: ${MACHINE_UUID}
EOF
```
Apply the cluster template to create a Kubernetes cluster.

```bash
omnictl cluster template sync -f cluster-template.yaml
```

## NVIDIA device plugin

When the cluster is ready download the kubeconfig file

```bash
ominctl kubeconfig -c ai-conformance
```

Install the NVIDIA device plugin

```bash
helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
helm repo update
helm install nvidia-device-plugin nvdp/nvidia-device-plugin --version=0.14.5 --set=runtimeClassName=nvidia --namespace kube-system
```

## Deploy traefik Gateway API

Insatll Gateway API CDRs

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml
kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/v3.5/docs/content/reference/dynamic-configuration/kubernetes-gateway-rbac.yml
```
Install Traefik via Helm

```bash
cat << EOF > values.yaml
providers:
  kubernetesGateway:
    enabled: true
EOF

helm repo add traefik https://traefik.github.io/charts
helm repo update
helm upgrade --install traefik traefik/traefik \
  -n traefik --create-namespace \
  -f values.yaml
```

## Deploy Kueue

Install Kueue via `kubectl`

```bash
kubectl apply --server-side -f https://github.com/kubernetes-sigs/kueue/releases/download/v0.14.2/manifests.yaml
```



## Deploy Kuberay operator

Install Ray via Kuberay via the helm chart

```bash
helm repo add kuberay https://ray-project.github.io/kuberay-helm/
helm repo update

helm install kuberay-operator kuberay/kuberay-operator --version 1.4.2 --namespace kube-system
```

Create a raycluster

```bash
helm install raycluster kuberay/ray-cluster --version 1.4.2 --namespace kube-system
```
