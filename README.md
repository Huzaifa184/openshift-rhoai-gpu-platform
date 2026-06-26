# Red Hat OpenShift AI : GPU Platform

Production-pattern OpenShift AI 3.4.0 cluster on bare-metal ESXi with GPU passthrough,
KServe model serving, and a verified DeepSeek R1 inference endpoint.
No managed control plane. No cloud GPU credits. No abstractions hiding the failure modes.

---

## Platform Status

| Component | Status | Comments |
|-----------|--------|----------|
| OpenShift 4.21 UPI | Running | Connected, self-hosted DNS + LB |
| NVIDIA GPU Operator | Running | Driver 580.126.20 · DCGM metrics live |
| RHOAI 3.4.0 | Running | DataScienceCluster reconciled |
| MinIO AIStor | Running | In-cluster S3 · 899GB capacity |
| KServe RawDeployment | Running | InferenceService Ready=True |
| DeepSeek R1 1.5B w8a8 | Serving | 12–17 tok/s · chain-of-thought confirmed |

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

### Laptop : VMware Workstation Pro Host

| Component | Specification |
|-----------|---------------|
| CPU | Intel Core i7-14650HX · 16 cores · 24 threads · 2.20 GHz base · Raptor Lake-HX |
| RAM | 64 GB DDR5 |
| Storage | 2 TB NVMe |
| GPU | NVIDIA GeForce RTX 4060 Laptop · 8 GB GDDR6 *(not used for cluster workloads)* |
| Hypervisor | VMware Workstation Pro (Windows) |

---

<img width="6048" height="5548" alt="IMG_5489 - Copy (2)" src="https://github.com/user-attachments/assets/da09e065-4a11-4047-a7a8-ad9c406814c7" />



## Cluster Topology

| Role | Count | Hypervisor | Host |
|------|-------|------------|------|
| Master nodes | 3× RHCOS VMs | VMware Workstation Pro | Laptop |
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

> OpenShift reports slightly lower values than provisioned due to RHCOS kernel and
> system process reservations — expected behaviour, not a misconfiguration.
> system process reservations — this is expected behaviour, not a misconfiguration.
