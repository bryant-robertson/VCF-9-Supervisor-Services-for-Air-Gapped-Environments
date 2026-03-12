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
- Broadcom Support Portal account 
 
---

## Part 1 — Initial Photon OS 5 Setup
 
### Step 1.1 — First Boot Console Login
 
Log in at the VM console. The default username is `root`.
 
```
Username: root
Password: <password set during install or OVF deployment>
```
 
---
 
### Step 1.2 — Enable Root SSH Login
 
Root SSH is disabled by default on Photon OS 5 Minimal.
 
```bash
vi /etc/ssh/sshd_config +32
```
 
Set or confirm the following line:
 
```
PermitRootLogin yes
```
 
Save (`:wq`) and restart SSH:
 
```bash
systemctl restart sshd
```
 
Verify from your workstation: `ssh root@<BASTION_IP>`
 
> **Security note:** Acceptable for a temporary download-only bastion. Disable or switch to key-based auth when the download phase is complete.
 
---
 
### Step 1.3 — Update the OS
 
Photon OS uses `tdnf` (Tiny DNF) as its package manager.
 
```bash
tdnf update -y
reboot
```
 
Log back in after the reboot.
 
---
 
### Step 1.4 — Set Hostname (Optional)
> This step is optional if the Hostname set during the Photon OS 5 deployment doesn't need to be changed.
 
```bash
hostnamectl set-hostname bastion
```
 
> A short hostname is fine for the Bastion Host. Only the Admin Host needs a proper FQDN since it communicates with vCenter, the Supervisor, and the Enterprise registry.
 
---
 
### Step 1.5 — Configure Static IP (Optional)
> This step is optional if the IP address set during the initial installation of Photon OS 5 doesn't need to change.
 
```bash
vi /etc/systemd/network/10-static.network
```
 
```ini
[Match]
Name=ens160
 
[Network]
Address=192.168.10.50/24
Gateway=192.168.10.1
DNS=8.8.8.8
DNS=8.8.4.4
```
 
```bash
systemctl restart systemd-networkd
ping -c 3 google.com
```
 
---
 
### Step 1.6 — Install Base Utilities
 
```bash
tdnf install -y \
  wget curl tar unzip git vim \
  openssl jq rsync \
  tmux
```
 
| Package | Why |
|---|---|
| `wget` `curl` | Downloading binaries and manifests |
| `tar` `unzip` | Extracting downloaded archives |
| `git` | Cloning config repos if needed |
| `vim` | Editing YAML manifests |
| `openssl` | Certificate inspection |
| `jq` | JSON parsing in scripts |
| `rsync` | Transferring the airgap bundle to the Admin Host |
| `tmux` | Keeping long-running downloads alive (`screen` is not available in Photon OS 5 repos) |
 
> **Photon OS 5 notes:** For network diagnostics use `ip addr` and `ss -tulnp`. For DNS lookups use `getent hosts <hostname>`. Neither `bind-utils` nor `net-tools` are available in the Photon OS 5 repos.
 
---
 
## Part 2 — Install imgpkg
 
`imgpkg` (Carvel) copies OCI image bundles to and from registries. Used on the Bastion to pull the Standard Package bundle and on the Admin Host to push it to the Enterprise registry.
 
```bash
IMGPKG_VERSION="v0.46.0"
 
wget https://github.com/carvel-dev/imgpkg/releases/download/${IMGPKG_VERSION}/imgpkg-linux-amd64 \
  -O /usr/local/bin/imgpkg
 
chmod +x /usr/local/bin/imgpkg
imgpkg version
```
 
---
 
## Part 3 — Install the CLI (VCF CLI or Tanzu CLI)
 
The right CLI depends on your vSphere version:
 
| vSphere Version | Primary CLI | Notes |
|---|---|---|
| **vSphere 9 / VCF 9.x** | **VCF CLI** (`vcf`) | Replaces both `tanzu` and `kubectl-vsphere` for VKS cluster operations. Tanzu CLI still needed for package management — see Part 5. |
| **vSphere 8.x** | **Tanzu CLI** (`tanzu`) | Use with `kubectl-vsphere` plugin for Supervisor auth. |
 
---
 
### Option A — VCF CLI for vSphere 9 / VCF 9.x
 
```bash
mkdir -p /opt/vcf && cd /opt/vcf
 
wget https://packages.broadcom.com/artifactory/vcf-distro/vcf-cli/linux/amd64/v9.0.0/vcf-cli.tar.gz
 
tar -xvf vcf-cli.tar.gz
mv vcf-cli-linux_amd64 /usr/local/bin/vcf
chmod +x /usr/local/bin/vcf
 
vcf version
```
 
Download the VCF CLI plugin bundle for Linux amd64 directly — **do not** use `vcf plugin download-bundle` as it downloads plugins for every OS and architecture and will exhaust disk space on a 200 GB VM:
 
```bash
wget https://packages.broadcom.com/artifactory/vcf-distro/vcf-cli-plugins/v9.0.0/linux/amd64/plugins.tar.gz \
  -O /opt/vcf/vcf-plugins-linux-amd64.tar.gz
 
ls -lh /opt/vcf/vcf-plugins-linux-amd64.tar.gz
```
 
> On the Admin Host, install plugins with:
> ```bash
> mkdir -p /opt/vcf/plugins && tar -xzvf vcf-plugins-linux-amd64.tar.gz -C /opt/vcf/plugins/
> vcf plugin install all --local-source /opt/vcf/plugins/
> ```
 
---
 
### Option B — Tanzu CLI for vSphere 8.x
 
```bash
mkdir -p /opt/tanzu && cd /opt/tanzu
 
TANZU_CLI_VERSION="v1.5.4"
 
wget https://github.com/vmware-tanzu/tanzu-cli/releases/download/${TANZU_CLI_VERSION}/tanzu-cli-linux-amd64.tar.gz
 
tar -xvf tanzu-cli-linux-amd64.tar.gz
# The archive extracts into a versioned subdirectory: v1.5.4/tanzu-cli-linux_amd64
install -m 0755 ${TANZU_CLI_VERSION}/tanzu-cli-linux_amd64 /usr/local/bin/tanzu
tanzu version
```
 
Download the vSphere plugin bundle for offline installation on the Admin Host:
 
```bash
tanzu plugin download-bundle \
  --group vmware-vsphere/default \
  --to-tar /opt/tanzu/tanzu-vsphere-plugins.tar.gz
```
 
> **kubectl-vsphere note:** This binary is served directly from the Supervisor at runtime. On the Admin Host, download it from `https://<SUPERVISOR-IP>/wcp/plugin/linux-amd64/vsphere-plugin.zip`. It cannot be pre-staged from the Bastion Host.
 
---
 
## Part 4 — Tanzu CLI for Package Management (vSphere 9 also needs this)
 
Even on vSphere 9 with the VCF CLI, the **Tanzu CLI is still required for Tanzu Package operations** on VKS Clusters (`tanzu package repository add`, `tanzu package install`). If you followed Option A above, also install the Tanzu CLI:
 
```bash
mkdir -p /opt/tanzu && cd /opt/tanzu
TANZU_CLI_VERSION="v1.5.4"
wget https://github.com/vmware-tanzu/tanzu-cli/releases/download/${TANZU_CLI_VERSION}/tanzu-cli-linux-amd64.tar.gz
tar -xvf tanzu-cli-linux-amd64.tar.gz
 
# The archive extracts into a versioned subdirectory, e.g. v1.5.4/tanzu-cli-linux_amd64
install -m 0755 ${TANZU_CLI_VERSION}/tanzu-cli-linux_amd64 /usr/local/bin/tanzu
 
tanzu version
```
 
---
 
## Part 5 — Install yq
 
```bash
YQ_VERSION="v4.52.4"
 
wget https://github.com/mikefarah/yq/releases/download/${YQ_VERSION}/yq_linux_amd64 \
  -O /usr/local/bin/yq
 
chmod +x /usr/local/bin/yq
yq --version
```
 
---
 
## Part 6 — Install kubectl
 
```bash
KUBECTL_VERSION=$(curl -L -s https://dl.k8s.io/release/stable.txt)
echo "Downloading kubectl ${KUBECTL_VERSION}"
 
wget https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl \
  -O /usr/local/bin/kubectl
 
chmod +x /usr/local/bin/kubectl
kubectl version --client
```
 
> **Version matching:** VKS 3.6 supports VKr v1.32 and v1.33. kubectl must be within ±1 minor version of your cluster's Kubernetes version. kubectl v1.35 is compatible with clusters running v1.32–v1.34. To pin a specific version replace the `stable.txt` URL with e.g. `v1.33.0`.
 
---
 
## Part 7 — Verify All Tools
 
```bash
echo "=== Bastion Host Tool Verification ==="
echo "imgpkg:  $(imgpkg version 2>&1 | head -1)"
echo "yq:      $(yq --version 2>&1)"
echo "kubectl: $(kubectl version --client --short 2>&1)"
vcf version   2>/dev/null && echo "vcf:     $(vcf version 2>&1 | head -1)"   || true
tanzu version 2>/dev/null && echo "tanzu:   $(tanzu version 2>&1 | head -1)" || true
```
 
---
 
## Part 8 — Create Download Directory Structure
 
```bash
mkdir -p /opt/airgap/{vkr-ovas,vks-service,tanzu-packages,supervisor-services,binaries,vcf-cli,tanzu-cli}
```
 
| Directory | Contents |
|---|---|
| `vkr-ovas/` | VKr OVA files (one subdirectory per K8s version + OS flavor) |
| `vks-service/` | VKS service registration YAML |
| `tanzu-packages/` | VKS Standard Package repository bundle tar |
| `supervisor-services/` | Supervisor Service OVAs and image bundle tars |
| `binaries/` | kubectl, imgpkg, yq |
| `vcf-cli/` | VCF CLI binary + plugin bundle (vSphere 9) |
| `tanzu-cli/` | Tanzu CLI binary + plugin bundle |
 
---
 
## Part 9 — Download VKr OVA Files
 
VKr OVAs are the OS+Kubernetes node images VKS uses to provision cluster VMs. In Content Library mode these are imported directly into a **local vCenter Content Library** — no registry push required.
 
### Step 9.1 — Identify Required VKr Versions
 
VKS 3.6 supports **VKr v1.32 and v1.33**. Download OVAs for each Kubernetes minor version you intend to offer. Both Photon OS and Ubuntu flavors are available — you cannot mix OS types within a single VKS Cluster. Photon OS is Broadcom's recommended default.
 
### Step 9.2 — Download from the Broadcom Support Portal
 
```
https://support.broadcom.com/
  → My Downloads
  → Search: "vSphere Kubernetes Release"
  → Expand the desired VKr version
  → Download all four files per VKr:
      <distro-name>.ovf
      <distro-name>.vmdk
      <distro-name>.mf    (required if using OVF security policy)
      <distro-name>.cert  (required if using OVF security policy)
```
 
> **Critical — record the distribution name string now.** The Content Library item name on the Admin Host must exactly match this string — for example `v1.33.1+vmware.2-photon-fips.1`. Copy it to a notes file before closing the browser tab. It is required when creating the Content Library item and cannot be easily recovered later.
 
Stage the downloaded files:
 
```bash
mkdir -p /opt/airgap/vkr-ovas/v1.33-photon
mkdir -p /opt/airgap/vkr-ovas/v1.32-photon
 
mv ~/photon-5-kube-v1.33*.{ovf,vmdk,mf,cert} /opt/airgap/vkr-ovas/v1.33-photon/ 2>/dev/null || true
mv ~/photon-5-kube-v1.32*.{ovf,vmdk,mf,cert} /opt/airgap/vkr-ovas/v1.32-photon/ 2>/dev/null || true
 
ls -lh /opt/airgap/vkr-ovas/
```
 
---
 
## Part 10 — Stage the VKS Service YAML
 
VKS is registered with vCenter asynchronously using a service definition YAML — no full vCenter update required.
 
If you downloaded `vks-3.6.0-package.yaml` from the Broadcom Support Portal, move it into the staging directory:
 
```
https://support.broadcom.com/
  → My Downloads
  → Search: "vSphere Kubernetes Service"
  → Expand vSphere Kubernetes Service 3.6.0
  → Download: vks-3.6.0-package.yaml (if not already on hand)
```
 
```bash
mv ~/vks-3.6.0-package.yaml /opt/airgap/vks-service/
ls -lh /opt/airgap/vks-service/
```
 
**Prerequisites confirmed from the VKS 3.6.0 package YAML:**
 
| Requirement | Detail |
|---|---|
| Supervisor Kubernetes version | `> 1.30.0` (vCenter 9.0 or vCenter 8.0 Update 3g+) |
| **HA Supervisor required** | VKS 3.6.0 will not install on a non-HA (single control plane) Supervisor |
| Minimum upgrade source | VKS 3.3.0 or later (cannot skip from older versions) |
 
> Verify your Supervisor is HA-enabled before attempting VKS registration. A non-HA Supervisor will reject the installation.
 
---
 
## Part 11 — Download the VKS Standard Package Bundle
 
The VKS Standard Package bundle contains all Tanzu Packages (cert-manager, Contour, Prometheus, Grafana, Velero, etc.) that can be installed on VKS Clusters. It is pushed to the Enterprise registry from the Admin Host and added as a package repository on each VKS Cluster.
 
The `vsphere/supervisor/packages` path on `projects.packages.broadcom.com` is **publicly accessible** — no login required, the same way vCenter pulls it without any credentials configured.
 
```bash
# Start a tmux session — this bundle is several GB and takes 30+ minutes
tmux new -s airgap
 
# VKS Standard Packages v2025.6.17
# Compatible with VKS 3.5.x / 3.6.x and Kubernetes v1.30–v1.33
TANZU_PKG_VERSION="2025.6.17"
 
imgpkg copy \
  -b projects.packages.broadcom.com/vsphere/supervisor/packages/${TANZU_PKG_VERSION}/vks-standard-packages:v${TANZU_PKG_VERSION} \
  --to-tar /opt/airgap/tanzu-packages/tanzu-packages.tar \
  --cosign-signatures \
  --registry-response-header-timeout 600s
 
ls -lh /opt/airgap/tanzu-packages/
```
 
> If disconnected during the download, reattach with `tmux attach -t airgap`.
 
---
 
## Part 12 — Download Supervisor Service Assets
 
> **cert-manager and Contour:** For VKS 3.5+ these are included in the Standard Package bundle pulled in Part 12 and deployed as Tanzu Packages on VKS Clusters — they do not need a separate Supervisor Service bundle pull.
 
### Step 11.1 — Harbor Bootstrap Registry (OVA)
 
If no Enterprise OCI registry exists in the air-gapped environment yet, Harbor can be deployed from an OVA first. It then serves as the registry for the Standard Package bundle and any other images.
 
```
https://support.broadcom.com/
  → My Downloads → Search: "Harbor"
  → Download: photon-5-harbor-v2.x.x-vmware.x.ova
```
 
```bash
mv ~/photon-5-harbor-v2*.ova /opt/airgap/supervisor-services/
ls -lh /opt/airgap/supervisor-services/
```
 
### Step 11.2 — Local Consumption Interface (LCI)
 
The **Local Consumption Interface** is the self-service developer UI embedded in the vSphere Client. It allows developers and DevOps teams to create namespaces, request storage, and deploy workloads directly from the vSphere Client without needing vCenter admin access — the primary self-service layer for Supervisor tenants.
 
First, stage the service registration YAML (this is the file you downloaded from Broadcom — copy it to the staging directory):
 
```bash
cp ~/lci-svs-9.0.2.yaml /opt/airgap/supervisor-services/
```
 
Then pull the image bundle:
 
```bash
imgpkg copy \
  -b projects.packages.broadcom.com/vsphere/iaas/lci-service/9.0.2/lci-service:9.0.2-f943fb89 \
  --to-tar /opt/airgap/supervisor-services/lci-9.0.2.tar \
  --cosign-signatures
 
ls -lh /opt/airgap/supervisor-services/
```
 
> The LCI image is on the `vsphere/iaas/` path. If the pull returns UNAUTHORIZED, contact Broadcom support to confirm your account has access to this product on your Site ID.
 
### Step 11.3 — Additional Supervisor Services (Optional)
 
For any other Supervisor Services your environment requires, find the image reference in the Supervisor Services catalog and use the same pattern:
 
```
https://vsphere-tmm.github.io/Supervisor-Services/
```
 
```bash
# Example — replace with the actual image reference from the catalog above
imgpkg copy \
  -b projects.packages.broadcom.com/<service-path>:<version> \
  --to-tar /opt/airgap/supervisor-services/<service-name>.tar \
  --cosign-signatures
```
 
---
 
## Part 13 — Stage CLI Binaries
 
The Admin Host has no internet access, so it cannot download tools from GitHub or any public source. Every binary installed on the Bastion Host in Parts 3–7 needs to be physically carried into the air-gapped environment alongside the image bundles and OVAs. Copying them into `/opt/airgap` now means they transfer as part of the single `rsync` or tar operation in Part 16, rather than requiring a separate manual process. Without this step the Admin Host would have the image bundles but no tools to actually push them to the registry, authenticate to the Supervisor, or manage VKS Clusters.
 
```bash
cp /usr/local/bin/imgpkg  /opt/airgap/binaries/
cp /usr/local/bin/yq      /opt/airgap/binaries/
cp /usr/local/bin/kubectl /opt/airgap/binaries/
 
# VCF CLI (vSphere 9)
cp /usr/local/bin/vcf                       /opt/airgap/vcf-cli/  2>/dev/null || true
cp /opt/vcf/vcf-plugins-linux-amd64.tar.gz  /opt/airgap/vcf-cli/  2>/dev/null || true
 
# Tanzu CLI (both vSphere 8 and 9)
cp /usr/local/bin/tanzu                     /opt/airgap/tanzu-cli/
cp /opt/tanzu/tanzu-vsphere-plugins.tar.gz  /opt/airgap/tanzu-cli/ 2>/dev/null || true
 
ls -lh /opt/airgap/binaries/ /opt/airgap/vcf-cli/ /opt/airgap/tanzu-cli/
```
 
---
 
## Part 14 — Generate a Download Manifest
 
```bash
echo "Bastion Host Download Manifest" > /opt/airgap/MANIFEST.txt
echo "Generated: $(date)"             >> /opt/airgap/MANIFEST.txt
echo ""                               >> /opt/airgap/MANIFEST.txt
find /opt/airgap -type f | sort | xargs ls -lh >> /opt/airgap/MANIFEST.txt
cat /opt/airgap/MANIFEST.txt
```
 
---
 
## Part 15 — Record Harbor Registry Credentials
 
The Admin Host needs credentials to push images to Harbor. Record the robot account details now so they are ready when you run the `imgpkg copy` commands on the Admin Host.
 
Harbor uses **robot accounts** scoped to a project for programmatic access. The project `vc01.home.lab` already has a push-capable robot account configured:
 
| Field | Value |
|---|---|
| **Registry** | `harbor-01.home.lab` |
| **Project** | `vc01.home.lab` |
| **Robot account (push)** | `robot$vc01.home.lab+vcenter-push` |
| **Secret** | *(token generated in Harbor — store securely)* |
 
On the Admin Host, set these as environment variables before running any `imgpkg copy --to-repo` commands. This keeps the token out of shell history and command lines:
 
```bash
export IMGPKG_REGISTRY_HOSTNAME=harbor-01.home.lab
export IMGPKG_REGISTRY_USERNAME='robot$vc01.home.lab+vcenter-push'
export IMGPKG_REGISTRY_PASSWORD='<robot-account-secret>'
```
 
> To regenerate the secret if lost: Harbor UI → Project `vc01.home.lab` → Robot Accounts → `robot$vc01.home.lab+vcenter-push` → ⋯ → Refresh Secret.
 
Also save the Harbor CA cert to the bundle so it is available on the Admin Host without needing to re-fetch it:
 
```bash
openssl s_client -connect harbor-01.home.lab:443 -showcerts </dev/null 2>/dev/null \
  | openssl x509 -outform PEM > /opt/airgap/harbor-ca.crt
```
 
This cert is also needed when configuring the Supervisor to trust Harbor during the vCenter setup phase.
 
---
 
## Part 16 — Transfer Files to the Admin Host
 
### Option A — rsync (recommended for large transfers)
 
```bash
rsync -avz --progress /opt/airgap/ root@admin.home.lab:/opt/airgap/
```
 
### Option B — SCP
 
```bash
scp -r /opt/airgap root@admin.home.lab:/opt/
```
 
### Option C — Physical media
 
```bash
tar -czvf /tmp/airgap-bundle.tar.gz -C /opt airgap/
cp /tmp/airgap-bundle.tar.gz /mnt/usb/
 
# On the Admin Host to extract:
# tar -xzvf /mnt/usb/airgap-bundle.tar.gz -C /opt/
```
 
---
 
## Part 17 — Transfer Checklist
 
Confirm all items are present on the Admin Host before starting deployment.
 
### VKr OVA Files → vCenter Content Library
 
- [ ] `.ovf` + `.vmdk` for each required Kubernetes version and OS flavor
- [ ] `.mf` + `.cert` for each VKr (required if using OVF security policy)
- [ ] Distribution name strings recorded (e.g. `v1.33.1+vmware.2-photon-fips.1`) — needed when creating Content Library items
 
### VKS Service → Supervisor Service Registration
 
- [ ] `vks-3.6.0-package.yaml`
 
### CLI Binaries → Admin Host `/usr/local/bin`
 
- [ ] `imgpkg`
- [ ] `yq`
- [ ] `kubectl`
- [ ] `vcf` binary + `vcf-plugins-linux-amd64.tar.gz` *(vSphere 9)*
- [ ] `tanzu` binary + `tanzu-vsphere-plugins.tar.gz`
 
### Harbor Registry Credentials → Admin Host
 
- [ ] `harbor-ca.crt` transferred and path noted
- [ ] `IMGPKG_REGISTRY_USERNAME` and `IMGPKG_REGISTRY_PASSWORD` values recorded securely
 
### Container Image Bundles → Enterprise Registry
 
- [ ] `tanzu-packages.tar` (VKS Standard Packages `v2025.6.17`)
- [ ] `lci-9.0.2.tar` + `lci-svs-9.0.2.yaml` (Local Consumption Interface)
- [ ] `photon-5-harbor-v2.x.x.ova` *(if deploying Harbor as bootstrap registry)*
- [ ] Any additional Supervisor Service bundles
 
---
 
## What Happens Next (Admin Host Summary)
 
These steps are performed on the Admin Host.
 
1. **Install CLI tools** from the transferred binaries (`imgpkg`, `yq`, `kubectl`, `vcf`/`tanzu`)
2. **Deploy Harbor** *(if using as bootstrap registry)* — deploy OVA, create project, note registry URL and CA cert
3. **Obtain the Harbor CA certificate** — required for all `imgpkg` push operations and for configuring Supervisor trust later:
   ```bash
   openssl s_client -connect harbor-01.home.lab:443 -showcerts </dev/null 2>/dev/null \
     | openssl x509 -outform PEM > /opt/airgap/harbor-ca.crt
   cat /opt/airgap/harbor-ca.crt   # should start with -----BEGIN CERTIFICATE-----
   ```
   > **Note:** `--registry-insecure` does NOT skip TLS verification in imgpkg — it switches to plain HTTP. Since Harbor only listens on HTTPS this will still fail. The CA cert is the only working approach.
4. **Set Harbor push credentials** — see Part 15 for the robot account details. Set the environment variables before pushing:
   ```bash
   export IMGPKG_REGISTRY_HOSTNAME=harbor-01.home.lab
   export IMGPKG_REGISTRY_USERNAME='robot$vc01.home.lab+vcenter-push'
   export IMGPKG_REGISTRY_PASSWORD='<robot-account-secret>'
   ```
5. **Push Standard Package bundle** to Harbor:
   ```bash
   imgpkg copy --tar /opt/airgap/tanzu-packages/tanzu-packages.tar \
     --to-repo harbor-01.home.lab/vc01.home.lab/vks-standard-packages \
     --cosign-signatures \
     --registry-response-header-timeout 600s \
     --registry-ca-cert-path /opt/airgap/harbor-ca.crt \
     --registry-username 'robot$vc01.home.lab+vcenter-push' \
     --registry-password "${IMGPKG_REGISTRY_PASSWORD}"
   ```
6. **Push Supervisor Service image bundles** to Harbor, including LCI:
   ```bash
   imgpkg copy --tar /opt/airgap/supervisor-services/lci-9.0.2.tar \
     --to-repo harbor-01.home.lab/vc01.home.lab/lci \
     --cosign-signatures \
     --registry-ca-cert-path /opt/airgap/harbor-ca.crt \
     --registry-username 'robot$vc01.home.lab+vcenter-push' \
     --registry-password "${IMGPKG_REGISTRY_PASSWORD}"
   ```
7. **Update the service YAMLs to point to Harbor** — both the VKS and LCI YAMLs still reference the Broadcom public registry out of the box. Before uploading them to vCenter, repoint the image references to Harbor:
   ```bash
   # VKS
   sed -i 's|projects.packages.broadcom.com/vsphere/iaas/vsphere-kubernetes-service/3.6.0/vsphere-kubernetes-service:3.6.0|harbor-01.home.lab/vc01.home.lab/vsphere-kubernetes-service:3.6.0|g' \
     /opt/airgap/vks-service/vks-3.6.0-package.yaml
 
   # LCI
   sed -i 's|projects.packages.broadcom.com/vsphere/iaas/lci-service/9.0.2/lci-service:9.0.2-f943fb89|harbor-01.home.lab/vc01.home.lab/lci:9.0.2-f943fb89|g' \
     /opt/airgap/supervisor-services/lci-svs-9.0.2.yaml
 
   # Verify
   grep "image:" /opt/airgap/vks-service/vks-3.6.0-package.yaml
   grep "image:" /opt/airgap/supervisor-services/lci-svs-9.0.2.yaml
   ```
8. **Register Supervisor Services** in vCenter using the updated YAMLs:
   ```
   vCenter → Workload Management → Services → Add → Upload lci-svs-9.0.2.yaml
   ```
9. **Enable the Supervisor** in vCenter (networking, storage, config)
10. **Create a local Content Library** in vCenter — import each VKr OVA as a separate item; the item name must exactly match the VKr distribution name string
11. **Register VKS** with vCenter using the updated service YAML:
   ```
   vCenter → Workload Management → Services → Add → Upload vks-3.6.0-package.yaml
   ```
12. **Create vSphere Namespace(s)** — associate the Content Library so VKS can find VKr images
13. **Deploy VKS Cluster(s)** via `kubectl apply -f vksCluster.yaml -n <namespace>`
14. **Add Standard Package repository** on each VKS Cluster:
    ```bash
    tanzu package repository add vks-standard \
      --url harbor-01.home.lab/vc01.home.lab/vks-standard-packages:v2025.6.17 \
      --namespace vmware-system-vks-public
    ```
15. **Install Tanzu Packages** via `tanzu package install` (cert-manager, Contour, etc.)
 
---
 
## Photon OS 5 vs Ubuntu Command Reference
 
| Task | Ubuntu | Photon OS 5 |
|---|---|---|
| Install packages | `apt install -y <pkg>` | `tdnf install -y <pkg>` |
| Update system | `apt update && apt upgrade -y` | `tdnf update -y` |
| Search packages | `apt search <pkg>` | `tdnf search <pkg>` |
| List installed | `dpkg -l` | `tdnf list installed` |
| Service management | `systemctl` | `systemctl` *(identical)* |
| DNS lookup | `nslookup` / `dig` | `getent hosts <hostname>` |
| Network interfaces | `ip addr`, `netstat` | `ip addr`, `ss -tulnp` |
 
---
 
## Troubleshooting
 
**`tdnf update` fails with SSL errors:**
```bash
update-ca-trust
tdnf update -y
```
 
**imgpkg download times out behind a corporate proxy:**
 
Set proxy environment variables before running `imgpkg`:
```bash
export HTTP_PROXY=http://proxy.example.com:3128
export HTTPS_PROXY=http://proxy.example.com:3128
export NO_PROXY=localhost,127.0.0.1,harbor-01.home.lab
```
 
**imgpkg push to Harbor fails with `x509: certificate signed by unknown authority`:**
`--registry-insecure` does NOT skip TLS verification in imgpkg — it switches to plain HTTP, which Harbor doesn't serve. The CA cert is required. Retrieve it via the TLS handshake and pass it explicitly:
```bash
openssl s_client -connect harbor-01.home.lab:443 -showcerts </dev/null 2>/dev/null \
  | openssl x509 -outform PEM > /opt/airgap/harbor-ca.crt
 
imgpkg copy --tar <bundle.tar> \
  --to-repo harbor-01.home.lab/vc01.home.lab/<repo> \
  --registry-ca-cert-path /opt/airgap/harbor-ca.crt
```
Note: the Harbor API endpoint `/api/v2.0/systeminfo/getcert` only works if Harbor was deployed with its own internal CA. If Harbor uses an externally-issued cert that endpoint returns `cert not found` — use `openssl s_client` instead.
 
**imgpkg UNAUTHORIZED on `projects.packages.broadcom.com`:**
The `vsphere/supervisor/packages` path is publicly accessible and should never require credentials. If you get UNAUTHORIZED it means credentials are cached in `/root/.docker/config.json` from a previous login session — `imgpkg` picks them up automatically and the registry rejects them instead of falling back to anonymous access. Fix:
```bash
rm -f /root/.docker/config.json
```
 
**imgpkg copy fails on cosign verification:**
```bash
# Omit --cosign-signatures if pulling from a known-trusted internal mirror
imgpkg copy \
  -b projects.packages.broadcom.com/vsphere/supervisor/packages/2025.6.17/vks-standard-packages:v2025.6.17 \
  --to-tar /opt/airgap/tanzu-packages/tanzu-packages.tar \
  --registry-response-header-timeout 600s
```
 
**SSH root login not taking effect:**
```bash
grep -E "^PermitRootLogin|^PasswordAuthentication" /etc/ssh/sshd_config
systemctl restart sshd
```
 
**VKr Content Library item name mismatch — clusters fail to provision:**
The Content Library item name must exactly match the distribution name string, e.g. `v1.33.1+vmware.2-photon-fips.1`. Check the name used when importing the OVF and compare against what the VKS Cluster manifest or VKr custom resource expects.
 
---
 
## Appendix — Cloud-Init Bastion Bootstrap
 
Parts 1–9 of this guide can be fully automated using cloud-init. Photon OS 5 supports cloud-init natively and vCenter can inject user-data at OVA/OVF deployment time.
 
The file `bastion-cloud-init.yaml` automates:
 
| Part | What it does |
|---|---|
| 1.2 | Writes `/etc/ssh/sshd_config.d/99-bastion.conf` with `PermitRootLogin yes` |
| 1.3 | `package_upgrade: true` — full OS update on first boot |
| 1.4 | Sets hostname to `bastion` |
| 1.5 | Static IP block (commented out — uncomment and fill in values, or leave as DHCP) |
| 1.6 | Installs base utilities via `tdnf` |
| 2 | Downloads and installs `imgpkg v0.46.0` |
| 3B + 4 | Downloads and installs `tanzu-cli v1.5.4` |
| 3A | VCF CLI v9.0.0 block — **commented out**, uncomment for vSphere 9 |
| 5 | Downloads and installs `yq v4.52.4` |
| 6 | Downloads and installs `kubectl` (latest stable) |
| 8 | Creates `/opt/airgap/` directory structure |
| — | Copies all binaries to `/opt/airgap/binaries/` for transfer to Admin Host |
| — | Writes `/opt/airgap/bootstrap.log` with tool versions and layout summary |
 
### How to inject into vCenter OVA deployment
 
**Base64-encode the file** (required by vCenter's user-data field):
 
```bash
base64 -w0 bastion-cloud-init.yaml
```
 
In vCenter: **Deploy OVF Template → Customize template → vApp Options → user-data** — paste the base64 output.
 
> Alternatively, if your VM template supports `guestinfo.userdata`, set that property and `guestinfo.userdata.encoding` to `base64`.
 
### Monitoring bootstrap progress
 
Cloud-init runs at first boot. Connect to the VM console (or via SSH once sshd is running) and watch progress:
 
```bash
# Follow cloud-init output live
journalctl -u cloud-init -f
 
# Check final status
cloud-init status --long
 
# Review bootstrap log written by runcmd
cat /opt/airgap/bootstrap.log
```
 
Cloud-init completion typically takes 3–5 minutes depending on network speed (OS update + binary downloads).
 
### What is NOT automated
 
These steps remain manual after the VM boots:
 
- **Part 10** — VKr OVA downloads require the Broadcom support portal (browser login)
- **Part 11** — Copy `vks-3.6.0-package.yaml` to `/opt/airgap/vks-service/`
- **Part 12** — `imgpkg copy` of the Standard Package bundle (long-running, use `tmux`)
- **Part 13** — Harbor OVA and LCI bundle downloads (support portal)
- **Parts 15–18** — Manifest, transfer, and checklist
 
---
 
## Version Reference
 
| Component | Version (March 2026) | Source |
|---|---|---|
| VCF CLI | v9.0.0 | packages.broadcom.com/artifactory/vcf-distro/vcf-cli |
| Tanzu CLI | v1.5.4 | github.com/vmware-tanzu/tanzu-cli/releases |
| imgpkg (Carvel) | v0.46.0 | github.com/carvel-dev/imgpkg/releases |
| yq (mikefarah) | v4.52.4 | github.com/mikefarah/yq/releases |
| kubectl | v1.35.2 | dl.k8s.io |
| VKS Service | 3.6.0 | support.broadcom.com |
| VKS-supported VKr versions | v1.32, v1.33 | VKS 3.6 interoperability matrix |
| VKS Standard Packages repo | v2025.6.17 | projects.packages.broadcom.com |
 
---
 
*This guide targets Photon OS 5 Minimal with vSphere Supervisor / VKS using Content Library OVA mode for node images. For the original Ubuntu-based source, see [vsphere-tmm/vsphere-supervisor – air-gapped.md](https://github.com/vsphere-tmm/vsphere-supervisor/blob/main/airgapped/air-gapped.md).*
