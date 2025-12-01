# Getting Started

This chapter introduces the fundamental concepts you need to understand before diving into OpenShift installation methods.

## What is OpenShift?

OpenShift is Red Hat's enterprise Kubernetes platform. It extends Kubernetes with:

- **Integrated container registry** for storing images
- **Built-in CI/CD** with OpenShift Pipelines and GitOps
- **Developer tools** like Source-to-Image and web console
- **Enterprise security** with SELinux, RBAC, and network policies
- **Operator-managed components** - all cluster services run as operators, managed by the Cluster Version Operator

An OpenShift cluster consists of:

| Component | Purpose |
|-----------|---------|
| **Control plane nodes** | Run the API server, etcd, and controllers that manage the cluster |
| **Worker nodes** | Run your application workloads |
| **etcd** | Distributed key-value store holding all cluster state |
| **Operators** | Automated software managing cluster components |

---

## What is OpenShift Installation?

Installing OpenShift means:

1. **Provisioning infrastructure** - Creating VMs, bare metal servers, or cloud resources
2. **Bootstrapping** - Starting a temporary control plane to initialize the cluster
3. **Forming the cluster** - Control plane nodes join and take over from bootstrap
4. **Adding workers** - Worker nodes join to run workloads
5. **Day-2 configuration** - Applying post-install settings and operators

Different installation methods handle these steps differently, but all result in a running OpenShift cluster.

---

## Red Hat CoreOS (RHCOS)

OpenShift nodes run **Red Hat CoreOS (RHCOS)**, a specialized operating system:

| Feature | Description |
|---------|-------------|
| **Immutable** | Base OS is read-only; changes applied via layered config |
| **Container-optimized** | Minimal footprint, designed to run containers |
| **Ignition-based** | Configured at first boot via Ignition files |
| **Managed** | OS updates handled by the Machine Config Operator |

You don't install RHCOS manually; the installation process handles it automatically using **Ignition configs** (JSON files defining system configuration).

---

## Release and OS Images

Two types of images are central to installation:

| Image Type | Purpose | Example |
|------------|---------|---------|
| **Release image** | Contains all OpenShift component versions | `quay.io/openshift-release-dev/ocp-release:4.14.10-x86_64` |
| **OS image** | Bootable RHCOS for nodes | `rhcos-4.14.10-x86_64-live.x86_64.iso` |

The **release image** is a container that includes:
- Metadata about all operator images
- The `openshift-install` binary
- Version and upgrade information

Each OpenShift version has a matching RHCOS version, and the installer handles this automatically.

---

## The openshift-install Binary

The core installation tool is `openshift-install`:

```bash
# Check version
openshift-install version

# Interactive installation
openshift-install create cluster --dir=my-cluster

# Generate only configuration files
openshift-install create install-config --dir=my-cluster
```

This binary is built from the [openshift/installer](https://github.com/openshift/installer) repository and is embedded in each release image.

---

## Operators and Controllers

OpenShift uses the **Operator pattern** extensively. Understanding this is essential.

### What is a Controller?

A **controller** is a loop that:
1. Watches for changes to specific resources
2. Compares actual state to desired state
3. Takes action to reconcile differences

```
Desired State (YAML) → Controller → Actual State (Running System)
```

### What is an Operator?

An **operator** is a Kubernetes application that:
- Packages one or more controllers
- Manages complex applications or infrastructure
- Often installed via OLM (Operator Lifecycle Manager)

**Example:** The Machine Config Operator runs controllers that watch `MachineConfig` resources and apply OS changes to nodes.

### Why This Matters

OpenShift installation relies heavily on operators:
- **Assisted Service** is an operator with controllers for cluster provisioning
- **Hive** is an operator managing cluster lifecycle
- Post-install, dozens of operators manage cluster components

---

## Custom Resources and CRDs

In OpenShift, almost everything is represented as a **resource** in the Kubernetes API. Built-in resources like Pods, Deployments, and Services are well-known, but Kubernetes also allows defining new resource types. These are called **Custom Resources (CRs)**, and they are central to how OpenShift installation works.

When you install a cluster using the Assisted Installer, you create CRs that describe what you want: which cluster to create, which hosts to use, what network configuration to apply. Controllers watch these resources and do the actual work.

| Term | Definition | Representation |
|------|------------|----------------|
| **CRD** (Custom Resource Definition) | Schema defining a new resource type | Go structs in `*_types.go` files |
| **CR** (Custom Resource) | Instance of a CRD | YAML files applied to the cluster |
| **Controller** | Code watching CRs and acting on them | Go code with reconciliation loops |

**How CRDs are defined:**

CRDs are typically defined as Go structs. For example, the [InfraEnv CRD](https://github.com/openshift/assisted-service/blob/master/api/v1beta1/infraenv_types.go) is defined in `infraenv_types.go`:

```go
// InfraEnv is the Schema for the infraenvs API
type InfraEnv struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec   InfraEnvSpec   `json:"spec,omitempty"`
    Status InfraEnvStatus `json:"status,omitempty"`
}
```

**How CRs are used:**

Users create Custom Resources as YAML files and apply them to the cluster:

```yaml
# This is a Custom Resource (instance of InfraEnv CRD)
apiVersion: agent-install.openshift.io/v1beta1
kind: InfraEnv
metadata:
  name: my-infraenv
spec:
  pullSecretRef:
    name: pull-secret
```

The Assisted Service watches `InfraEnv` resources and generates discovery ISOs in response.

**How to explore CRD schemas:**

| Method | Command / Location |
|--------|-------------------|
| **oc/kubectl explain** | `oc explain infraenv.spec` - interactive schema explorer |
| **Get CRD definition** | `oc get crd infraenvs.agent-install.openshift.io -o yaml` |
| **OpenShift Console** | Administration → CustomResourceDefinitions → search for CRD |

```bash
# Example: explore InfraEnv spec fields
oc explain infraenv.spec
oc explain infraenv.spec.pullSecretRef
```

---

## Installation Methods Overview

OpenShift offers multiple installation paths for different scenarios:

### Traditional Methods

| Method | Description | When to Use |
|--------|-------------|-------------|
| **IPI** | Installer provisions cloud infrastructure automatically | Cloud deployments (AWS, Azure, GCP) |
| **UPI** | You provision infrastructure, installer configures it | Custom requirements, restricted environments |

### Assisted Methods

| Method | Description | When to Use |
|--------|-------------|-------------|
| **Assisted SaaS** | Hosted service at console.redhat.com | Quick start, no hub cluster needed |
| **Assisted MCE** | On-premise service on hub cluster | Enterprise, multi-cluster, disconnected |
| **ABI** | Self-contained ISO with embedded service | Fully disconnected, no external dependencies |

### Specialized Methods

| Method | Description | When to Use |
|--------|-------------|-------------|
| **IBI** | Deploy SNO from pre-built seed images | Fast edge deployment, disaster recovery |
| **HCP** | Control plane runs as pods on management cluster | Multi-tenancy, cost optimization |
| **ZTP** | GitOps-driven deployment at scale | Large fleets, edge/telco |

See [Installation Methods Overview](01-installation-methods-overview.md) for detailed comparison.

---

## What is the Assisted Installer?

The **Assisted Installer** ([assisted-service](https://github.com/openshift/assisted-service)) simplifies OpenShift installation by:

1. **Generating discovery ISOs** - Bootable images with an agent
2. **Discovering hardware** - Agent reports CPU, RAM, disks, network
3. **Validating requirements** - Checks hardware meets OpenShift minimums
4. **Orchestrating installation** - Coordinates the entire process

```
User → Creates Cluster Definition
       ↓
Assisted Service → Generates Discovery ISO
       ↓
Hosts boot ISO → Agent discovers hardware
       ↓
Validations pass → Installation proceeds
       ↓
Running OpenShift Cluster
```

### Assisted Installer Components

| Component | Repository | Purpose |
|-----------|------------|---------|
| **assisted-service** | [openshift/assisted-service](https://github.com/openshift/assisted-service) | Core API and business logic |
| **assisted-image-service** | [openshift/assisted-image-service](https://github.com/openshift/assisted-image-service) | Generates customized ISOs |
| **assisted-installer-agent** | [openshift/assisted-installer-agent](https://github.com/openshift/assisted-installer-agent) | Runs on hosts during discovery |

### Which Methods Use the Assisted Installer?

| Method | Assisted Installer Role |
|--------|------------------------|
| **Assisted SaaS** | Directly - hosted service at console.redhat.com |
| **Assisted MCE** | Directly - on-premise deployment on hub cluster |
| **ABI** | Embedded in the bootable ISO |
| **IBI** | Used for orchestration via IBI Operator |
| **ZTP** | Via rendered CRDs (ClusterDeployment, AgentClusterInstall) |
| **HCP** | Optional - Agent CAPI provider for bare metal workers |
| **IPI/UPI** | Not used - these use `openshift-install` directly |

### Assisted Installer API Styles

The Assisted Installer supports two API styles:

| Style | How It Works | Used By |
|-------|--------------|---------|
| **REST API** | Direct HTTP calls to the service | Assisted SaaS (console.redhat.com) |
| **Kubernetes API** | Create CRs, controllers reconcile | Assisted MCE (on-premise) |

**REST API (Imperative):**
- You tell the system *what to do* step by step
- `curl -X POST /api/v2/clusters`, `curl -X PATCH /api/v2/hosts/{id}`

**Kubernetes API (Declarative):**
- You define *what you want*, controllers figure out how
- `oc apply -f clusterdeployment.yaml`
- Better for GitOps and automation

Both APIs interact with the same assisted-service backend. This guide focuses primarily on the Kubernetes/CRD approach, as it's more common for enterprise and automated deployments.

For detailed tutorials, see:
- [REST API Getting Started](https://github.com/openshift/assisted-service/blob/master/docs/user-guide/rest-api-getting-started.md)
- [Kube-API Getting Started](https://github.com/openshift/assisted-service/blob/master/docs/hive-integration/kube-api-getting-started.md)

---

## Pull Secrets

A **pull secret** is JSON credentials for accessing container registries:

```json
{
  "auths": {
    "quay.io": {"auth": "base64-encoded-credentials"},
    "registry.redhat.io": {"auth": "base64-encoded-credentials"}
  }
}
```

You need a pull secret to:
- Download OpenShift container images
- Access the Assisted Installer service

Get yours from [console.redhat.com/openshift/downloads](https://console.redhat.com/openshift/downloads).

---

## Next Steps

Now that you understand the fundamentals:

1. **Choose your method** → [Installation Methods Overview](01-installation-methods-overview.md)
2. **Traditional cloud install** → [IPI Guide](02-traditional-installers/ipi.md)
3. **Assisted installation** → [Assisted Overview](03-assisted-installation/overview.md)
4. **Disconnected install** → [ABI Guide](03-assisted-installation/abi.md)

---

## Quick Reference

| I want to... | Go to... |
|--------------|----------|
| Understand all installation options | [Installation Methods Overview](01-installation-methods-overview.md) |
| Install on AWS/Azure/GCP | [IPI Guide](02-traditional-installers/ipi.md) |
| Use the web-based installer | [Assisted SaaS](03-assisted-installation/saas-vs-onprem.md) |
| Install without internet | [ABI Guide](03-assisted-installation/abi.md) |
| Manage multiple clusters | [Assisted MCE](03-assisted-installation/saas-vs-onprem.md) |
| Deploy edge/telco at scale | [ZTP Guide](06-gitops-provisioning/index.md) |
| Look up an abbreviation | [Glossary](11-glossary.md) |


