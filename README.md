# Red Hat OpenShift AI — GPU Platform

Production-pattern OpenShift AI 3.4.0 cluster on bare-metal ESXi with GPU passthrough,
KServe model serving, and a verified DeepSeek R1 inference endpoint.
No managed control plane. No cloud GPU credits. No abstractions hiding the failure modes.

---

## Platform Status

| Component | Status | Verified |
|-----------|--------|----------|
| OpenShift 4.21 UPI | Running | Connected, self-hosted DNS + LB |
| NVIDIA GPU Operator | Running | Driver 580.126.20 · DCGM metrics live |
| RHOAI 3.4.0 | Running | DataScienceCluster reconciled |
| MinIO AIStor | Running | In-cluster S3 · 899GB capacity |
| KServe RawDeployment | Running | InferenceService Ready=True |
| DeepSeek R1 1.5B w8a8 | Serving | 12–17 tok/s · chain-of-thought confirmed |

---

## Architecture

![Architecture Diagram](architecture/diagram.svg)

### Infrastructure topology

**Hybrid hypervisor — deliberate, not default.**
Master nodes run nested inside VMware Workstation Pro on a Windows laptop.
Worker nodes run on bare-metal ESXi on a separate physical workstation.
The GPU-bearing worker (worker1) is on bare-metal ESXi — the only viable path
for PCI passthrough on consumer hardware.

**Connected UPI — not IPI.**
Self-hosted HAProxy and BIND9. Mirrors the install pattern in regulated environments
where infrastructure is explicitly managed rather than abstracted away.
Full control over DNS and load-balancing at the cost of manual provisioning.

### Hardware

| Role | Hardware | Notes |
|------|----------|-------|
| ESXi host | Dell Precision T5810 | ESXi 7.0 |
| GPU | NVIDIA GTX 1660 Super | TU116 · Turing · 6GB GDDR6 · PCI passthrough |
| Worker nodes | 3× RHCOS VMs on ESXi | worker1 has GPU passthrough |
| Master nodes | 3× RHCOS VMs | Nested on VMware Workstation Pro |

---

## Model Serving

**Model:** `RedHatAI/DeepSeek-R1-Distill-Qwen-1.5B-quantized.w8a8`
**Runtime:** vLLM 0.18.0 · KServe RawDeployment · OpenAI-compatible API

### Inference endpoint

```bash
curl -k -X POST \
  "https://deepseek-r1-distill-qwen-15b-deepseek-reasoning-demo.apps.<CLUSTER_DOMAIN>/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-r1-distill-qwen-15b",
    "messages": [{"role":"user","content":"Explain KV cache in transformer inference."}],
    "max_tokens": 512,
    "temperature": 0.6
  }'
```

### GPU configuration (GTX 1660 Super — 6GB Turing)
