# Red Hat OpenShift AI : GPU Platform

Production-pattern OpenShift AI 3.4.0 platform, self-hosted on bare-metal ESXi, GTX 1660 Super GPU passthrough, DeepSeek R1 1.5B w8a8 serving at 12–17 tok/s, no cloud dependency.


---



| Component | Notes |
|-----------|-------|
| OpenShift 4.21 UPI | Connected UPI · self-hosted HAProxy + BIND9 |
| NVIDIA GPU Operator | Driver 580.126.20 · DCGM metrics confirmed |
| RHOAI 3.4.0 | DataScienceCluster reconciled · dashboard accessible |
| MinIO AIStor | In-cluster S3 · 899 GB capacity |
| KServe RawDeployment | InferenceService Ready=True |
| DeepSeek R1 1.5B w8a8 | 12–17 tok/s · chain-of-thought reasoning confirmed |

---

## Hardware

### ESXi Host : Dell Precision Workstation T5810

| Component | Specification |
|-----------|---------------|
| CPU | Intel Xeon E5-2620 v4 · 8 cores · 16 threads · 2.10 GHz base · 3.00 GHz turbo · Broadwell-EP 14nm |
| RAM | 96 GB ECC DDR4 |
| Storage | 128 GB SSD (ESXi OS) · 2 TB HDD (VM storage) |
| GPU | NVIDIA GeForce GTX 1660 Super · TU116 · Turing (SM 7.5) · 6 GB GDDR6 |
| Hypervisor | VMware ESXi 7.0 |

### VMware Workstation Pro Host : Lenovo Legion 5 16IRX9

| Component | Specification |
|-----------|---------------|
| CPU | Intel Core i7-14650HX · 16 cores · 24 threads · 2.20 GHz base · Raptor Lake-HX |
| RAM | 64 GB DDR5 |
| Storage | 2 TB NVMe |
| GPU | NVIDIA GeForce RTX 4060 Laptop · 8 GB GDDR6 *(not used for cluster workloads)* |
| Hypervisor | VMware Workstation Pro (Windows) |

---

<img width="6048" height="5548" alt="IMG_5489 - Copy (2)" src="https://github.com/user-attachments/assets/da09e065-4a11-4047-a7a8-ad9c406814c7" />

---

## Cluster Topology

| Role | Count | Hypervisor | Host |
|------|-------|------------|------|
| Master nodes | 3× RHCOS VMs | VMware Workstation Pro | Lenovo Legion Laptop |
| Worker nodes | 3× RHCOS VMs | ESXi 7.0 | Dell Workstation |
| GPU worker (worker1) | 1× RHCOS VM | ESXi 7.0 · DirectPath I/O | Dell Workstation |

## Cluster Specifications

**OpenShift:** 4.21 · Connected UPI · Self-hosted HAProxy + BIND9
**RHOAI:** 3.4.0 Self-Managed

| Node | Role | CPU | RAM | Storage | Notes |
|------|------|-----|-----|---------|-------|
| master1.openshift-ai.huzaifa.lab | control-plane | 8 cores | 16 GB | 100 GB | Nested · Workstation Pro |
| master2.openshift-ai.huzaifa.lab | control-plane | 8 cores | 16 GB | 100 GB | Nested · Workstation Pro |
| master3.openshift-ai.huzaifa.lab | control-plane | 8 cores | 16 GB | 100 GB | Nested · Workstation Pro |
| worker1.openshift-ai.huzaifa.lab | worker · GPU | 16 cores | 64 GB | 200 GB | ESXi · GTX 1660 Super passthrough |
| worker2.openshift-ai.huzaifa.lab | worker | 8 cores | 24 GB | 200 GB | ESXi |
| worker3.openshift-ai.huzaifa.lab | worker | 8 cores | 24 GB | 200 GB | ESXi |
| ocp-svc.huzaifa.lab | bastion · LB · DNS | 4 cores | 8 GB | 150 GB | RHEL 10 · Workstation Pro |

> OpenShift reports slightly lower values than provisioned due to RHCOS kernel and system process reservations : expected behaviour, not a misconfiguration.

---

<img width="949" height="504" alt="image" src="https://github.com/user-attachments/assets/e0332e2d-55fb-4dbe-b84e-cd3128ecc0b3" />

---
<img width="494" height="439" alt="image" src="https://github.com/user-attachments/assets/e1788fec-0ae5-4651-a9ef-504a0936b574" />

---
<img width="1891" height="1014" alt="image" src="https://github.com/user-attachments/assets/b6b91083-fa30-49a2-8ad7-d43309f57e2f" />

---

<img width="859" height="511" alt="image" src="https://github.com/user-attachments/assets/32f5b78a-ad3b-4d48-b32c-d8d98739681b" />

---
<img width="949" height="494" alt="image" src="https://github.com/user-attachments/assets/655774a4-43b6-419a-8b86-440a079b3318" />

---
<img width="907" height="352" alt="image" src="https://github.com/user-attachments/assets/1aef2494-2a6c-4f73-b475-7ec6b05792ec" />

## Architecture

### Topology

1. **Hybrid hypervisor** : Master nodes run nested inside VMware Workstation Pro on a Windows laptop. Worker nodes run on bare-metal ESXi on a separate physical workstation. The GPU-bearing worker (worker1) is on bare-metal ESXi , the viable path for PCI passthrough on consumer hardware.

2. **Connected UPI** : Self-hosted HAProxy and BIND9 on a dedicated RHEL 10 bastion node. Mirrors the install pattern used in regulated environments where infrastructure is explicitly managed rather than abstracted away.

3. **GPU passthrough** : GTX 1660 Super passed through from ESXi to worker1 via VMware DirectPath I/O. Hypervisor detection suppressed via `hypervisor.cpuid.v0 = FALSE` in the VM config. Memory fully reserved , passthrough devices DMA directly into guest memory and cannot be swapped.

4. **NFD + NVIDIA GPU Operator** : Node Feature Discovery labels the GPU node automatically. The GPU Operator builds and loads the NVIDIA kernel module against the running RHCOS kernel via a privileged DaemonSet , the only supported path for driver installation on an immutable RHCOS node. Driver 580.126.20 confirmed via `nvidia-smi` inside the guest.

5. **OpenShift AI** : RHOAI 3.4.0 installed via OperatorHub. `DataScienceCluster` CR created explicitly to trigger component provisioning , operator running alone is insufficient. KServe configured in RawDeployment mode with ModelMesh disabled. Hardware profile `gtx1660super-gpu` maps the GPU to RHOAI workloads.

6. **Storage** : Local Storage Operator provisions raw ESXi disks as PersistentVolumes across worker nodes. MinIO AIStor deployed as in-cluster S3 , model artifacts stored in bucket `ai-initial`, pulled by the KServe storage initializer on pod start.

7. **Model serving** : DeepSeek R1 1.5B w8a8 deployed via KServe InferenceService backed by the vLLM NVIDIA GPU ServingRuntime. CUDA graph capture disabled (`--enforce-eager`) , required on 6 GB VRAM where the graph pool alone would exhaust available memory. Inference confirmed at 12–17 tok/s via OpenAI-compatible endpoint.

### Openshift Cluster Setup

- **Static IP allocation:** All nodes assigned static IPs prior to installation. No DHCP dependency ensures predictable DNS resolution and HAProxy backend targeting across reboots.
- **Bridged networking on Workstation Pro:** Master VMs placed directly on the physical LAN segment alongside ESXi workers. Required for OVN-Kubernetes and the API VIP to function across the hybrid topology.
- **Bastion node (RHEL 10):** Single VM serving three roles:
  - **HAProxy:** Layer 4 load balancer, API :6443, MCS :22623, ingress :80/:443
  - **BIND9:** Authoritative DNS for `openshift-ai.huzaifa.lab` — forward + reverse zones, wildcard `*.apps`
  - **Apache httpd:** Served RHCOS ignition files during UPI bootstrap
- **UPI install flow:** `install-config.yaml` → manifests → ignition configs → ISO boot → bootstrap → masters → workers → bootstrap removed
- **Configuration files:** Redacted `install-config.yaml`, `haproxy.cfg`, `named.conf`, zone files in [`infra/`](infra/)
### AI Platorm Stack

| Layer | Operator / Tool | Version | Notes |
|-------|----------------|---------|-------|
| Node Feature Discovery | NFD Operator | 4.21.0 | Auto-labels GPU nodes · feeds GPU Operator |
| GPU Enablement | NVIDIA GPU Operator | 26.3.2 | Driver 580.126.20 · DaemonSet on worker1 |
| GPU Monitoring | DCGM Exporter | 26.3.2 | Live metrics via Prometheus · DCGM_FI_DEV_GPU_UTIL |
| GPU Sharing | NVIDIA Device Plugin | 26.3.2 | Time-slicing · 2 schedulable slices per GPU |
| AI Platform | Red Hat OpenShift AI | 3.4.0 | DataScienceCluster reconciled · dashboard live |
| Model Serving | KServe · vLLM ServingRuntime | v0.18.0 | InferenceService Ready=True · RawDeployment mode |
| Object Storage | MinIO AIStor Operator | 2026.4.13 | In-cluster S3 · 899 GB capacity |
| Persistent Storage | Local Storage Operator | 4.21.0 | LocalVolumeSet · 3 PVs across worker nodes |
| Service Mesh | OpenShift Service Mesh | 3.2.0 | Required by RHOAI gateway components |

---

## Storage

Two storage layers: local persistent volumes for stateful workloads, and MinIO AIStor as the in-cluster S3-compatible object store for model artifacts.

### Local Storage Operator
From ESXI host and attach 900GB disk from the datastore to each worker (thin provisioning does the magic of allocating more than available, since I am only using worker 1 mainly for AI workloads,I am safe here)

Raw disks on each worker node are provisioned as PersistentVolumes using the Local Storage Operator. A `LocalVolumeSet` watches nodes labelled `storage=local` and automatically creates PVs from any available disk : rotational or non-rotational.

Label worker nodes before applying the manifest:

```bash
oc label node worker1.openshift-ai.huzaifa.lab storage=local
oc label node worker2.openshift-ai.huzaifa.lab storage=local
oc label node worker3.openshift-ai.huzaifa.lab storage=local
```

Manifests: [`manifests/storage/localvolumeset.yaml`](manifests/storage/localvolumeset.yaml) · [`manifests/storage/storageclass.yaml`](manifests/storage/storageclass.yaml)

`WaitForFirstConsumer` binding mode on the StorageClass ensures PVs are only bound when a pod is scheduled : critical for GPU workloads that must land on a specific node.

Verify provisioning:

```bash
oc get pv
# NAME                CAPACITY   STATUS      CLAIM
# local-pv-6b052930   900Gi      Available
# local-pv-8e2008e1   900Gi      Bound       aistor/0-ai-minio-pool-0-0
# local-pv-a79cf2c2   900Gi      Bound       rhoai-model-registries/model-catalog-postgres
```

### MinIO AIStor

MinIO AIStor is the in-cluster S3-compatible object store used as the model artifact backend for RHOAI. MinIO Community Edition was archived February 2026 : AIStor Free tier (launched December 2025) is the current viable path for standalone in-cluster S3 on RHOAI.

**Why AIStor over NooBaa/MCG:** NooBaa is designed for multi-cloud object gateway use cases and adds significant operational overhead. AIStor is a straight S3 implementation that integrates directly with RHOAI data connections and the KServe storage initializer with no additional configuration.

- **Operator version:** `v2026.4.13`
- **Namespace:** `aistor`
- **Internal S3 endpoint:** `https://minio.aistor.svc.cluster.local:9000`
- **Console:** `https://minio-console-aistor.apps.openshift-ai.huzaifa.lab`
- **Capacity:** 900 GiB · backed by `local-pv-8e2008e1`

Manifests: [`manifests/storage/minio-objectstore.yaml`](manifests/storage/minio-objectstore.yaml) · [`manifests/storage/minio-policybinding.yaml`](manifests/storage/minio-policybinding.yaml)

Retrieve admin credentials:

```bash
oc get secret ai-minio-generated -n aistor \
  -o jsonpath='{.data.config\.env}' | base64 -d | grep -E "ROOT_USER|ROOT_PASSWORD"
```

<img width="956" height="510" alt="image" src="https://github.com/user-attachments/assets/1396d01b-8e68-4ea0-a34f-c800ec1a65c5" />


Model artifact bucket layout:

```
ai-initial/
└── models/
    └── deepseek-r1-distill-qwen-1.5b/
        ├── config.json
        ├── generation_config.json
        ├── model.safetensors        # 2,245,331,488 bytes (~2.1 GB)
        ├── recipe.yaml
        ├── special_tokens_map.json
        ├── tokenizer.json
        └── tokenizer_config.json
```

---

## OpenShift AI

### Operator Installation

Installed via OperatorHub → Red Hat OpenShift AI → channel `fast` → version `3.4.0`. The operator deploys into `redhat-ods-operator` and manages two CRs: `DSCInitialization` (bootstrap) and `DataScienceCluster` (component configuration).

```bash
oc get csv -n redhat-ods-operator | grep rhods
# rhods-operator.3.4.0   3.4.0   Succeeded

oc get pods -n redhat-ods-operator
# redhat-ods-operator-*   3/3   Running
```

### DSCInitialization

Created automatically by the operator on first install. Configures the applications namespace, monitoring namespace, and trusted CA bundle.

Key fields from live cluster:

```yaml
spec:
  applicationsNamespace: redhat-ods-applications
  monitoring:
    managementState: Managed
    namespace: redhat-ods-monitoring
  trustedCABundle:
    managementState: Managed
```

Status: `phase: Ready`

### DataScienceCluster

The `DataScienceCluster` CR is the primary configuration object. It must be created explicitly : the operator running alone does not provision the dashboard, KServe, or any other component until this CR exists.

Manifest: [`manifests/rhoai/datasciencecluster.yaml`](manifests/rhoai/datasciencecluster.yaml)

Key component decisions:

- **`modelsAsService: Removed`** : disables ModelMesh multi-model serving. Required to enable KServe single-model mode, which is the only mode that supports direct GPU assignment per InferenceService.
- **`rawDeploymentServiceConfig: Headless`** : KServe creates standard Kubernetes Deployments instead of Knative Serving resources. No Knative or Istio dependency. Direct Deployment object control.
- **`trainer: Managed`** : enabled but `TrainerReady=False` is expected; requires the JobSet operator which is not installed. All components required for model serving are Ready.

Verify component status:

```bash
oc get datasciencecluster default-dsc \
  -o jsonpath='{.status.conditions[*].type}' | tr ' ' '\n'
# KserveReady          → True
# DashboardReady       → True
# RayReady             → True
# ModelRegistryReady   → True
# TrustyAIReady        → True
# WorkbenchesReady     → True
# TrainerReady         → False  (expected : JobSet not installed)
```

### Namespace Setup

The project namespace requires two labels before deploying a model:

```bash
oc new-project deepseek-reasoning-demo
oc label namespace deepseek-reasoning-demo \
  opendatahub.io/dashboard=true \
  modelmesh-enabled=false
```

`modelmesh-enabled=false` forces KServe single-model serving mode. Without it the dashboard defaults to ModelMesh and GPU assignment is not possible.

Manifest: [`manifests/model/namespace.yaml`](manifests/model/namespace.yaml)

### Dashboard

```
https://rh-ai.apps.openshift-ai.huzaifa.lab
```

Authentication via OpenShift OAuth. Hardware profiles, serving runtimes, and model deployments are managed through the dashboard or via CLI against `redhat-ods-applications`.

<img width="950" height="490" alt="image" src="https://github.com/user-attachments/assets/f0711c50-c320-4a65-a25b-0a1eb65b1dc5" />



### Hardware Profile

Hardware profiles replace Accelerator Profiles (deprecated in RHOAI 2.19). The `gtx1660super-gpu` profile exposes the GTX 1660 Super to RHOAI workloads and constrains resource requests to what the node can actually provide.

Manifest: [`manifests/rhoai/hardwareprofile.yaml`](manifests/rhoai/hardwareprofile.yaml)

```bash
oc get hardwareprofile gtx1660super-gpu -n redhat-ods-applications
```
<img width="938" height="409" alt="image" src="https://github.com/user-attachments/assets/8271b8bd-d0ad-4a0e-8417-10faaf58ca54" />

<img width="812" height="404" alt="image" src="https://github.com/user-attachments/assets/574f3dc7-a416-49fe-acb3-b190a612950d" />

---

## GPU

### PCI Passthrough : ESXi Configuration

The GTX 1660 Super is passed through from the ESXi host to `worker1` using VMware DirectPath I/O. This gives the RHCOS VM direct hardware access : a requirement for NVIDIA driver loading and CUDA initialisation on an immutable RHCOS node.

**Step 1 : Enable VT-d / IOMMU** in BIOS on the Dell T5810.

**Step 2 : Enable passthrough in vSphere:**
Navigate to host → Configure → Hardware → PCI Devices → enable passthrough on the GTX 1660 Super. Reboot the host.

**Step 3 : Disable nested virtualisation on the worker1 VM:**
PCI passthrough and nested hardware-assisted virtualisation cannot coexist on the same VM.

```
VM Settings → CPU → uncheck "Expose hardware-assisted virtualization to the guest OS"
```

**Step 4 : Reserve all VM memory:**
Passthrough devices DMA directly into guest memory. The hypervisor cannot swap or balloon that memory : it must be fully pinned.

```
VM Settings → Memory → enable "Reserve all guest memory (All locked)"
```

**Step 5 : Suppress hypervisor detection:**
NVIDIA GeForce drivers refuse to load inside a VM unless the hypervisor signature is hidden. Add via vSphere → VM Options → Advanced → Edit Configuration:

```
hypervisor.cpuid.v0 = "FALSE"
```

**Step 6 : Add PCI device to VM** and power on. Verify inside the guest:

```bash
lspci | grep -i nvidia
# 13:00.0 VGA compatible controller: NVIDIA Corporation TU116 [GeForce GTX 1660 SUPER]
```

### NVIDIA GPU Operator

Manages the full NVIDIA software stack on RHCOS nodes: driver, container toolkit, device plugin, DCGM exporter, and MIG manager. The operator builds and loads a kernel module matched to the running RHCOS kernel version via a privileged DaemonSet : the only supported mechanism for getting an NVIDIA driver onto an immutable RHCOS node.

Manifest: [`manifests/gpu/clusterpolicy.yaml`](manifests/gpu/clusterpolicy.yaml)

Verify the stack is running:

```bash
oc get pods -n nvidia-gpu-operator
# nvidia-driver-daemonset-*            2/2   Running
# nvidia-container-toolkit-daemonset-* 1/1   Running
# nvidia-device-plugin-daemonset-*     2/2   Running
# nvidia-dcgm-exporter-*               1/1   Running
# nvidia-operator-validator-*          1/1   Running
# gpu-feature-discovery-*              1/1   Running
```

Verify GPU is schedulable:

```bash
oc describe node worker1.openshift-ai.huzaifa.lab | grep "nvidia.com/gpu"
# Capacity:    nvidia.com/gpu: 2
# Allocatable: nvidia.com/gpu: 2
```

Verify driver inside the node:

```bash
oc debug node/worker1.openshift-ai.huzaifa.lab
chroot /host
nvidia-smi
# NVIDIA-SMI 580.126.20   Driver Version: 580.126.20
# GTX 1660 SUPER | 00000000:13:00.0
```

### GPU Time-Slicing

Time-slicing exposes a single physical GPU as multiple schedulable resources. Two slices are configured : two pods can concurrently request `nvidia.com/gpu: 1` from the same card. No memory isolation, shared fault domain.

**MIG (Multi-Instance GPU)** partitions a GPU into isolated instances with dedicated compute and memory. Hardware-enforced isolation, no interference between workloads. The correct enterprise choice : but requires Ampere architecture (SM ≥ 8.0). The GTX 1660 Super is Turing (SM 7.5). MIG is not available on this hardware.

**MPS (Multi-Process Service)** shares a single CUDA context across multiple processes, reducing context switch overhead. Documented as supported on Kubernetes. However, there is a known open issue specific to OpenShift: MPS test pods report success but only one process executes on the GPU at a time : the sharing does not actually occur. Red Hat and NVIDIA have an open item on this. MPS was evaluated and ruled out before implementation, not discovered as a failure mid-deployment.

**Why time-slicing over MIG and MPS:**

**Time-slicing** was the remaining viable option. It round-robins GPU access between processes at a configurable interval. No memory isolation, shared fault domain : a pod crash can affect co-located workloads. Acceptable for a single-user lab where both slices are unlikely to be active simultaneously. Two slices are configured, matching the hardware profile `maxCount` and leaving room for a second workload without contention at idle.




Manifest: [`manifests/gpu/time-slicing-config.yaml`](manifests/gpu/time-slicing-config.yaml)

Apply and patch the ClusterPolicy to reference it:

```bash
oc apply -f manifests/gpu/time-slicing-config.yaml

oc patch clusterpolicy gpu-cluster-policy \
  -n nvidia-gpu-operator \
  --type=merge \
  -p '{"spec":{"devicePlugin":{"config":{"name":"time-slicing-config","default":"any"}}}}'

oc get node worker1.openshift-ai.huzaifa.lab \
  -o jsonpath='{.status.allocatable.nvidia\.com/gpu}'
```
Verify
<img width="1053" height="420" alt="image" src="https://github.com/user-attachments/assets/ab91c3c5-7e04-4d08-b0a9-bd367df0f5ea" />



### GPU Monitoring : Prometheus Metrics

DCGM Exporter exposes GPU metrics to the OpenShift Prometheus stack via a `ServiceMonitor`. Enabled by `serviceMonitor.enabled: true` in the ClusterPolicy : no additional configuration required.

Key PromQL queries:

```promql
# GPU utilisation (%)
DCGM_FI_DEV_GPU_UTIL{Hostname=~"worker1.*"}

# GPU memory used (bytes)
DCGM_FI_DEV_FB_USED{Hostname=~"worker1.*"}

# GPU power draw (watts)
DCGM_FI_DEV_POWER_USAGE{Hostname=~"worker1.*"}

# GPU temperature (celsius)
DCGM_FI_DEV_GPU_TEMP{Hostname=~"worker1.*"}
```
## Results:
---
GPU utilisation (%)
<img width="1825" height="643" alt="image" src="https://github.com/user-attachments/assets/6ec9061f-f3ec-49b7-93ee-42bfd8262534" />

GPU memory used (bytes)
<img width="910" height="320" alt="image" src="https://github.com/user-attachments/assets/2b9a299e-f906-4fa9-89b2-517564faa391" />

GPU power draw (watts)
<img width="907" height="320" alt="image" src="https://github.com/user-attachments/assets/ae8f34a4-8d22-4bc1-9974-cf7afb95e4fc" />

GPU temperature (celsius)
<img width="905" height="329" alt="image" src="https://github.com/user-attachments/assets/539d33bd-bbe7-4280-8e3c-f3ad88899420" />

---


Access via OpenShift Console → Observe → Metrics, or via the Prometheus API:

```bash
PROM=$(oc get route prometheus-k8s -n openshift-monitoring -o jsonpath='{.spec.host}')
TOKEN=$(oc create token prometheus-k8s -n openshift-monitoring)

curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$PROM/api/v1/query?query=DCGM_FI_DEV_GPU_UTIL" | python3 -m json.tool
```

---

## Model Deployment

### Model Selection

**Model:** `RedHatAI/DeepSeek-R1-Distill-Qwen-1.5B-quantized.w8a8`

Red Hat-provided quantized variant with pre-validated accuracy. w8a8 quantizes both weights and activations to INT8, using `CutlassInt8ScaledMMLinearKernel` which runs on CUDA cores with no tensor core requirement : viable on Turing SM 7.5. The 1.5B scale fits the 6 GB VRAM constraint: weights load at 2.13 GiB leaving 2.46 GiB for KV cache.

| Model | Weights VRAM | Fits 6 GB |
|-------|-------------|-----------|
| DeepSeek R1 7B w8a8 | ~7–8 GB | No |
| DeepSeek R1 1.5B FP16 | ~3.2 GB | Marginal |
| DeepSeek R1 1.5B w8a8 | 2.13 GB | Yes : 2.46 GB KV headroom |

### Model Download : HuggingFace to MinIO

The model is pulled directly into the cluster using a Kubernetes Job. No local workstation download required.

```bash
# Grant anyuid SCC : required for pip install as root
oc adm policy add-scc-to-user anyuid \
  -z model-downloader-sa -n deepseek-reasoning-demo
```

Manifest: [`manifests/model/model-download-job.yaml`](manifests/model/model-download-job.yaml)

```bash
oc apply -f manifests/model/model-download-job.yaml
oc logs -f job/model-downloader -n deepseek-reasoning-demo
# Fetching 9 files: 100% 9/9
# Done.
```

### Data Connection

Created via Dashboard → deepseek-reasoning-demo → Connections → Add connection → S3-compatible. Creates a Kubernetes Secret referenced by the InferenceService `storage.key` field.

```
Name:     minio-models
Endpoint: https://minio.aistor.svc.cluster.local:9000
Bucket:   ai-initial
Region:   us-east-1
```

<img width="943" height="254" alt="image" src="https://github.com/user-attachments/assets/d654fb5b-bda4-4d7b-a8b4-a42a359559b6" />


### ServingRuntime

Manifest: [`manifests/model/servingruntime.yaml`](manifests/model/servingruntime.yaml)

Created automatically by the RHOAI dashboard. Defines the vLLM container image and runtime args. The `{{.Name}}` template token is replaced with the InferenceService name at deploy time.

### InferenceService

Manifest: [`manifests/model/inferenceservice.yaml`](manifests/model/inferenceservice.yaml)

**RHOAI 3.4 critical finding : args propagation:**

InferenceService `args` and ServingRuntime `args` are not reconciled back to the Deployment after initial creation. The Deployment object is the authoritative source of container args. To change vLLM args post-deployment, patch the Deployment directly:

```bash
oc patch deployment deepseek-r1-distill-qwen-15b-predictor \
  -n deepseek-reasoning-demo \
  --type=json \
  -p '[{"op":"replace","path":"/spec/template/spec/containers/0/args","value":[
    "--port=8080",
    "--model=/mnt/models",
    "--served-model-name=deepseek-r1-distill-qwen-15b",
    "--max-model-len=2048",
    "--gpu-memory-utilization=0.85",
    "--dtype=auto",
    "--max-num-seqs=4",
    "--enforce-eager"
  ]}]'
```

### Resource Usage

| Resource | Request | Limit | Runtime |
|----------|---------|-------|---------|
| CPU | 1 core | 2 cores | ~0.9 cores during inference |
| Memory | 2 GiB | 6 GiB | ~3.5 GiB |
| GPU | 1 | 1 | 1 (full device) |
| GPU VRAM | - | 6 GB | ~4.6 GB (weights + KV cache) |

VRAM breakdown:

```
Weights:    2.13 GiB  (w8a8 INT8)
KV cache:   2.46 GiB  (91,968 tokens · 2048 ctx)
───────────────────────────────
Used:      ~4.59 GiB
Usable:     5.61 GiB
Headroom:  ~1.02 GiB
```

### Verification

```bash
# InferenceService ready
oc get inferenceservice deepseek-r1-distill-qwen-15b \
  -n deepseek-reasoning-demo \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
# True

# Confirm vLLM startup
oc logs <pod-name> -n deepseek-reasoning-demo -c kserve-container | head -20
# enforce_eager=True
# Model loading took 2.13 GiB memory
# Available KV cache memory: 2.46 GiB
# Application startup complete.
```

### Live Inference Test

**Question:** What is 15% of 280?

**Raw API response** — full JSON including chain-of-thought:

```bash
curl -k -X POST \
  "https://deepseek-r1-distill-qwen-15b-deepseek-reasoning-demo.apps.openshift-ai.huzaifa.lab/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-r1-distill-qwen-15b",
    "messages": [{"role":"user","content":"What is 15% of 280?"}],
    "max_tokens": 256,
    "temperature": 0.6
  }' | python3 -m json.tool
```

DeepSeek R1 is a reasoning model — it thinks before answering. The `<think>` block in the response contains the internal chain-of-thought process, visible in the raw output:

```
"content": "First, I need to understand what the question is asking...
15% is equivalent to 0.15. Next, I'll multiply the decimal by 280...
Therefore, 15% of 280 is 42.\n</think>\n\nTo find 15% of 280..."
```

**Stripped response** — answer only, chain-of-thought removed:

```bash
curl -k -s -X POST \
  "https://deepseek-r1-distill-qwen-15b-deepseek-reasoning-demo.apps.openshift-ai.huzaifa.lab/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-r1-distill-qwen-15b",
    "messages": [{"role":"user","content":"What is 15% of 280?"}],
    "max_tokens": 256,
    "temperature": 0.6
  }' | python3 -c "
import sys, json, re
r = json.load(sys.stdin)
msg = r['choices'][0]['message']['content']
msg = re.sub(r'<think>.*?</think>', '', msg, flags=re.DOTALL).strip()
usage = r['usage']
print('Answer:', msg)
print(f'Tokens — prompt: {usage[\"prompt_tokens\"]} · completion: {usage[\"completion_tokens\"]} · total: {usage[\"total_tokens\"]}')
"
```

```
Answer: To find 15% of 280, follow these steps:

1. Convert the percentage to a decimal: 15% = 0.15
2. Multiply: 0.15 × 280 = 42

Answer: 42

Tokens — prompt: 17 · completion: 222 · total: 239
```

**Confirmed:** HTTP 200 · finish_reason: stop · OpenAI-compatible endpoint · running on GTX 1660 Super via KServe RawDeployment on RHOAI 3.4.0.

<img width="892" height="373" alt="image" src="https://github.com/user-attachments/assets/b9fd2d4c-1825-4edc-aba1-3b754fe0dbec" />

---

## Frontend

A custom chat frontend is deployed as a static HTML application inside the cluster. Running it in-cluster avoids CORS issues that arise when loading HTML from a local filesystem or a different origin.

Source: [`manifests/ai-chat-frontend/index.html`](manifests/ai-chat-frontend/index.html)

Features: OpenAI-compatible API · chain-of-thought display · web search augmentation · live tok/s stats · conversation history

### Deployment

```bash
# Create ConfigMap from HTML file
oc create configmap deepseek-frontend \
  -n deepseek-reasoning-demo \
  --from-file=index.html=manifests/ai-chat-frontend/index.html
```

Manifest: [`manifests/ai-chat-frontend/frontend-deployment.yaml`](manifests/ai-chat-frontend/frontend-deployment.yaml)

```bash
oc apply -f manifests/ai-chat-frontend/frontend-deployment.yaml

oc get route deepseek-chat-ui -n deepseek-reasoning-demo
# deepseek-chat-ui-deepseek-reasoning-demo.apps.openshift-ai.huzaifa.lab
```

Busybox httpd was used over nginx : nginx subPath ConfigMap mounts fail under RHCOS SCC restrictions. Busybox mounts the ConfigMap as a directory and serves it directly, which works cleanly under the restricted SCC.

### Configuration

| Field | Value |
|-------|-------|
| Endpoint | `https://deepseek-r1-distill-qwen-15b-deepseek-reasoning-demo.apps.openshift-ai.huzaifa.lab` |
| Model | `deepseek-r1-distill-qwen-15b` |
| Max Tokens | `512` |
| Temperature | `0.6` |

Max tokens is set to 512, not 2048. The model context window is 2048 total (prompt + completion). Setting it to 2048 causes a `VLLMValidationError` when the prompt itself consumes tokens.

### RHOAI Native Playground

RHOAI 3.4 includes a native chat playground : no separate frontend deployment required.

```
Dashboard → AI hub → Models → deepseek-r1-distill-qwen-1.5b → Open in Playground
```

The InferenceService must carry the label `opendatahub.io/genai-asset: "true"` to appear in the AI hub model list. This is already set in [`manifests/model/inferenceservice.yaml`](manifests/model/inferenceservice.yaml).

### Update after HTML changes

```bash
oc delete configmap deepseek-frontend -n deepseek-reasoning-demo
oc create configmap deepseek-frontend \
  -n deepseek-reasoning-demo \
  --from-file=index.html=manifests/ai-chat-frontend/index.html
oc rollout restart deployment/deepseek-chat-ui -n deepseek-reasoning-demo
```

---
DeepSeek R1 is a reasoning model, it generates an internal chain-of-thought before producing a final answer. 

This is visible in the UI as the thinking block above the response. The reasoning process itself consumes tokens, so `max_tokens` controls the combined budget for both thinking and the final answer. 

On the 1.5B distilled variant, the model does not self-regulate verbosity, it will reason at length regardless of question complexity. For simple questions, set `max_tokens` between 300–600 to allow the reasoning to complete and the answer to appear.

For complex technical questions the reasoning adds genuine value and can be shown or hidden via the Think toggle.

<img width="949" height="494" alt="image" src="https://github.com/user-attachments/assets/d48d5959-25c7-42e6-b371-f4fe2315af46" />

