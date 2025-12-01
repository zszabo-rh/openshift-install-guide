# Glossary

A complete reference of abbreviations and terms used in OpenShift installation.

## Abbreviations

| Term | Full Name | Description | Learn More |
|------|-----------|-------------|------------|
| **ABI** | Agent-Based Installer | Standalone disconnected installation method | [ABI Guide](03-assisted-installation/abi.md) |
| **ACM** | Advanced Cluster Management | Full multi-cluster management platform built on MCE | [ACM Integration](06-gitops-provisioning/acm-integration.md) |
| **BMO** | Baremetal Operator | Operator managing physical host lifecycle | [Operators Reference](07-operators-controllers/overview.md) |
| **BMAC** | Baremetal Agent Controller | Controller bridging BMO and Assisted Service | [Operators Reference](07-operators-controllers/overview.md) |
| **CAPI** | Cluster API | Kubernetes-native cluster lifecycle management | [CAPI Integration](05-hosted-control-planes/capi-integration.md) |
| **CIM** | Central Infrastructure Management | Host inventory management in MCE | [Assisted MCE](03-assisted-installation/saas-vs-onprem.md) |
| **CRD** | Custom Resource Definition | Kubernetes API extension mechanism | [Getting Started](00-getting-started.md#custom-resources-and-crds) |
| **CSR** | Certificate Signing Request | Kubernetes certificate request mechanism | [Bootstrap Process](02-traditional-installers/bootstrap-process.md) |
| **CVO** | Cluster Version Operator | Operator managing OpenShift version and upgrades | [Operators Reference](07-operators-controllers/overview.md) |
| **HCP** | Hosted Control Planes | Deployment model with control plane as pods | [HCP Overview](05-hosted-control-planes/index.md) |
| **IBI** | Image-Based Install | Fast SNO deployment from seed images | [IBI Guide](04-image-based-installation/ibi.md) |
| **IPI** | Installer-Provisioned Infrastructure | Fully automated cloud installation | [IPI Guide](02-traditional-installers/ipi.md) |
| **LCA** | Lifecycle Agent | Operator managing seed images and image-based upgrades | [IBI Guide](04-image-based-installation/ibi.md) |
| **MCE** | Multicluster Engine | Core cluster lifecycle operator | [Assisted MCE](03-assisted-installation/saas-vs-onprem.md) |
| **MCO** | Machine Config Operator | Operator managing node OS configuration | [Bootstrap Process](02-traditional-installers/bootstrap-process.md) |
| **MCS** | Machine Config Server | Service serving Ignition configs to nodes | [Bootstrap Process](02-traditional-installers/bootstrap-process.md) |
| **NMState** | Network Manager State | Declarative network configuration | [CRD Reference](08-crd-reference/supporting-crds.md) |
| **OLM** | Operator Lifecycle Manager | System for installing and updating operators | [Getting Started](00-getting-started.md#operators-and-controllers) |
| **RHCOS** | Red Hat CoreOS | Immutable OS for OpenShift nodes | [Getting Started](00-getting-started.md#red-hat-coreos-rhcos) |
| **SNO** | Single Node OpenShift | OpenShift running on a single machine | [Installation Methods](01-installation-methods-overview.md) |
| **TALM** | Topology Aware Lifecycle Manager | Operator for upgrade orchestration at scale | [ZTP Guide](06-gitops-provisioning/ztp.md) |
| **UPI** | User-Provisioned Infrastructure | Installation on pre-existing infrastructure | [UPI Guide](02-traditional-installers/upi.md) |
| **VIP** | Virtual IP | Floating IP address shared across nodes | [UPI Guide](02-traditional-installers/upi.md) |
| **ZTP** | Zero Touch Provisioning | GitOps-driven cluster deployment at scale | [ZTP Guide](06-gitops-provisioning/index.md) |

## Common Terms

| Term | Definition | Learn More |
|------|------------|------------|
| **Bootstrap node** | Temporary machine hosting initial control plane during installation | [Bootstrap Process](02-traditional-installers/bootstrap-process.md) |
| **Control plane** | The set of components managing the cluster (API server, etcd, controllers) | [Getting Started](00-getting-started.md) |
| **Discovery ISO** | Bootable image containing the discovery agent for Assisted Installer | [Assisted Overview](03-assisted-installation/overview.md) |
| **Hub cluster** | Central management cluster running MCE/ACM | [Hub and Spoke](03-assisted-installation/saas-vs-onprem.md#hub-and-spoke-architecture) |
| **Ignition** | First-boot provisioning system for CoreOS | [Bootstrap Process](02-traditional-installers/bootstrap-process.md) |
| **Pull secret** | JSON credentials for accessing container registries | [Getting Started](00-getting-started.md) |
| **Release image** | Container image containing complete OpenShift version metadata | [Getting Started](00-getting-started.md#release-and-os-images) |
| **Rendezvous host** | Host coordinating ABI installation; becomes cluster member | [ABI Guide](03-assisted-installation/abi.md) |
| **Seed image** | Pre-captured SNO state used for IBI deployments | [IBI Guide](04-image-based-installation/ibi.md) |
| **Spoke cluster** | Cluster provisioned and/or managed by a hub cluster | [Hub and Spoke](03-assisted-installation/saas-vs-onprem.md#hub-and-spoke-architecture) |
| **Worker node** | Cluster node running application workloads | [Getting Started](00-getting-started.md) |


