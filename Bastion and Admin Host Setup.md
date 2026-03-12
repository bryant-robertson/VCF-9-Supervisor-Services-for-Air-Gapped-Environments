# Photon OS 5 Minimal — vSphere Supervisor / VKS Air-Gapped Bastion Host Setup

> **Reference:** [vsphere-tmm/vsphere-supervisor – air-gapped.md](https://github.com/vsphere-tmm/vsphere-supervisor/blob/main/airgapped/air-gapped.md)
> **Base OS:** VMware Photon OS 5 Minimal
> **Platform:** vSphere Supervisor with vSphere Kubernetes Service (VKS)
> **Node image delivery: Content Library OVA mode** — VKr OVAs imported directly into vCenter Content Library
> **Tool Versions (verified March 2026):** VCF CLI v9.0.0 · Tanzu CLI v1.5.4 · imgpkg v0.46.0 · yq v4.52.4 · kubectl v1.35.2

---

## Prerequisites
 
- Photon OS 5 Minimal VM with 2+ vCPU, 4 GB+ RAM, **200+ GB disk** (VKr OVAs are 5–10 GB each)
- Internet connectivity from the VM
- SSH client access
- Broadcom Support Portal account with **Product Administrator** role (required for OVA and YAML downloads from My Downloads)
 
---
