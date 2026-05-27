# mce-ocp-cluster-install

---

# OpenShift ACM Cluster Deployment (Bare Metal)

This repository contains GitOps-friendly **Helm charts** and **Kubernetes manifests** used to automate the provisioning and management of bare-metal Red Hat OpenShift Container Platform (OCP) clusters.

By utilizing **Red Hat Advanced Cluster Management (ACM)**,  or **Multicluster Engine (MCE)**,  **Hive (`ClusterDeployment`)**, and **Metal3 (`BareMetalHost`)**, this repository allows you to declare a cluster's complete infrastructure setup—including network configurations (bonding, static IPs), disk encryption, and hardware BMC details—entirely as code.

---

## 🚀 What This Repository Does

* **Bare-Metal Automation:** Leverages the OpenShift Assisted Installer (`AgentClusterInstall`) and Metal3 to provision bare-metal hardware via Redfish/iLO/iDRAC virtual media.
* **Declarative Networking:** Uses `NMStateConfig` to automate LACP bonding (`802.3ad`), custom MTU settings (9000/Jumbo Frames), static IP pool assignments, and DNS configurations.
* **Enterprise Security:** Integrates **External Secrets Operator (ESO)** with HashiCorp Vault to securely inject infrastructure credentials (BMC iLO/iDRAC passwords, pull secrets) without exposing them in Git.
* **Day-0/Day-1 Ready:** Configures FIPS compliance, Disk Encryption via TPM v2, NTP services, and automatically registers the tenant cluster back to the ACM hub manager (`ManagedCluster`).

---

## 🛠 Prerequisites

Before deploying a cluster with this repository, ensure your environment has the following:

1. A working **Red Hat Advanced Cluster Management (ACM)** Hub Cluster. or a clsuter with **Multicluster Engine (MCE)** enabled 
2. **External Secrets Operator (ESO)** installed on the hub cluster and connected to your HashiCorp Vault server.
3. Bare-metal servers with working **iLO/iDRAC Redfish API** access and mapped network interfaces.
4. Reserved IP addresses in your target network VLAN for node IPs, API (`apiVIP`), and Ingress (`ingressVIP`).

---

## ⚙️ Configuration (`values.yaml`)

To deploy a new cluster, you must create or modify a Helm `values.yaml` file targeting your hardware environment.

### Core Variable Schema

| Variable | Expected Value | Dynamic/Static | Comments |
| --- | --- | --- | --- |
| **cluster** | `cluster-name` | Dynamic | Changes from cluster to cluster |
| **ingress** | `*.apps.<cluster-name>.<base-domain>` | Dynamic | Need a DNS record via Hostmaster |
| **ingressip** | IP from native vlan (Vlan3701) | Dynamic | For multi-node cluster, distinct from node IPs |
| **api** | `api.<cluster-name>.<base-domain>` | Dynamic | Need a DNS record via Hostmaster |
| **apiip** | IP from native vlan (Vlan3701) | Dynamic | For multi-node cluster, distinct from node IPs |
| **subnet** | native vlan subnet (Vlan3701) | Dynamic | e.g., `10.85.208.64/26` |
| **gateway** | native vlan gateway (Vlan3701) | Dynamic |  |
| **resolvers** | DNS Resolvers, an array if multiple | Static for Environment | e.g., `169.88.8.8` |
| **network.bondalgorithm** | `802.3ad` | Static | Standardized to LACP bonding |
| **network.miimon** | `150` | Static | Link monitoring frequency |
| **ntpservers** | An array of servers list | Static for Environment | Target NTP endpoints |
| **sshkey** | An ssh public key for the cluster nodes | Dynamic | Placed in configuration |
| **workercount** | Number of worker nodes | Dynamic |  |
| **mastercount** | Number of master/control-plane nodes | Dynamic | Standard is `3` for HA |
| **bootMode** | `UEFI` | Static | Required for PowerFlex driver compliance |
| **diskEncryption** | `enable` / `disable` | Dynamic | Toggles TPM v2 disk encryption |
| **fips** | `enable` / `disable` | Dynamic | Toggles FIPS 140-2 compliance mode |
| **vault.server** | Vault server endpoint url | Dynamic | Used by ESO to fetch BMC credentials |
| **imagesetref** | clusterimageset from the ACM cluster | Dynamic | Target OpenShift version (e.g., `openshift-4.18`) |
| **nodes** | Array of hardware blocks | Dynamic | Define `hostname`, `role`, `ip`, `mac1`, `mac2`, `ilo` endpoints, hardware `vendor` (Dell/HPE), and native interface names (`if1`, `if2`, etc.) |

---

## 📖 How to Use

### Step 1: Prepare Vault Secrets

Before starting the GitOps pipeline, add your hardware BMC credentials to your Vault server at the paths specified in your template configurations:

* **Global Pull Secret:** Path `provisioning/pull-secret-pulp` containing the `dockerconfigjson`.
* **Host BMC Credentials:** Path `provisioning/bmc` containing user logins matching your host short-names (e.g., `wh-0000053243-creds`).

### Step 2: Validate the Rendered Templates locally

You can use the Helm CLI to render the manifests locally and ensure your `values.yaml` parses correctly without syntax blocks:

```bash
helm template mce-ocp-cluster-install ./path-to-chart -f values.yaml

```

### Step 3: Deploy to the ACM Hub

You can apply these manifests directly via `kubectl` or hook this repository up to **ArgoCD / ACM Application Lifecycle** for GitOps synchronization.

To manually install via Helm:

```bash
helm install deploy-clusters-workload ./path-to-chart -f values.yaml --namespace my-cluster-namespace --create-namespace

```

### Step 4: Track Deployment Status

Once applied, Hive and ACM will begin the execution loop. You can monitor progress on the hub cluster via:

```bash
# Check the status of your bare-metal hosts
kubectl get bmh -n <cluster-namespace>

# Follow the installation progress
kubectl get agentclusterinstall -n <cluster-namespace>

# View the status of your managed cluster enrollment
kubectl get managedcluster <cluster-name>

```