# CRD Reference Index

This section documents the Custom Resource Definitions (CRDs) used in OpenShift installation.

## CRDs by Installation Method

| Installation Method | Primary CRDs |
|--------------------|--------------|
| **IPI/UPI** | (Uses install-config.yaml, not CRDs) |
| **Assisted (MCE)** | ClusterDeployment, AgentClusterInstall, InfraEnv, Agent |
| **Agent-Based Installer** | (Uses YAML files, embedded as ZTP manifests) |
| **Image-Based Install** | ClusterDeployment, ImageClusterInstall, BareMetalHost |
| **Hosted Control Planes** | HostedCluster, NodePool, HostedControlPlane |
| **ZTP** | ClusterInstance + above CRDs |

## CRD Categories

### [Installation CRDs](installation-crds.md)

Primary CRDs used to define and install clusters:

| CRD | API Group | Purpose |
|-----|-----------|---------|
| ClusterDeployment | hive.openshift.io | Cluster definition |
| AgentClusterInstall | extensions.hive.openshift.io | Assisted install config |
| InfraEnv | agent-install.openshift.io | Discovery environment |
| Agent | agent-install.openshift.io | Discovered host |
| ImageClusterInstall | extensions.hive.openshift.io | IBI cluster config |
| HostedCluster | hypershift.openshift.io | HCP cluster |
| NodePool | hypershift.openshift.io | HCP workers |
| ClusterInstance | siteconfig.open-cluster-management.io | Unified ZTP |

### [Supporting CRDs](supporting-crds.md)

CRDs that support the installation process:

| CRD | API Group | Purpose |
|-----|-----------|---------|
| AgentServiceConfig | agent-install.openshift.io | Assisted service deployment |
| NMStateConfig | agent-install.openshift.io | Static network config |
| ClusterImageSet | hive.openshift.io | OCP version reference |
| AgentClassification | agent-install.openshift.io | Host auto-labeling |
| BareMetalHost | metal3.io | Physical host management |
| PreprovisioningImage | metal3.io | BMO boot image |
| ClusterPool | hive.openshift.io | Cluster pooling |
| ClusterClaim | hive.openshift.io | Pool claiming |
| MachinePool | hive.openshift.io | Worker scaling |

### [Day 2 Machine Management](day2-machine-management.md)

CRDs for post-install machine management:

| CRD | API Group | Purpose |
|-----|-----------|---------|
| Machine | machine.openshift.io | Individual machine |
| MachineSet | machine.openshift.io | Machine group |
| MachineDeployment | machine.openshift.io | Rolling updates |
| MachineHealthCheck | machine.openshift.io | Auto-repair |
| MachineConfig | machineconfiguration.openshift.io | Node configuration |
| MachineConfigPool | machineconfiguration.openshift.io | Config grouping |

## CRD Relationship Diagram

```mermaid
graph TB
    subgraph clusterdef["Cluster Definition"]
        CI[ClusterInstance]
        CD[ClusterDeployment]
        ACI[AgentClusterInstall]
        ICI[ImageClusterInstall]
        HC[HostedCluster]
    end
    
    subgraph hostmgmt["Host Management"]
        IE[InfraEnv]
        AGENT[Agent]
        BMH[BareMetalHost]
        NMS[NMStateConfig]
    end
    
    subgraph supporting["Supporting"]
        CIS[ClusterImageSet]
        SECRET[Pull Secret]
        CP[ClusterPool]
    end
    
    subgraph hcpworkers["HCP Workers"]
        NP[NodePool]
        MD[MachineDeployment]
    end
    
    CI --> CD
    CI --> ACI
    CI --> IE
    CI --> BMH
    
    CD --> ACI
    CD --> ICI
    ACI --> CIS
    ACI --> IE
    
    IE --> NMS
    IE --> AGENT
    BMH --> AGENT
    
    CD --> SECRET
    
    CP --> CD
    
    HC --> NP
    NP --> MD
    
    style CI fill:#b56576,stroke:#8d4e5a,color:#fff
    style CD fill:#355070,stroke:#1d3557,color:#fff
    style ACI fill:#355070,stroke:#1d3557,color:#fff
    style ICI fill:#355070,stroke:#1d3557,color:#fff
    style HC fill:#355070,stroke:#1d3557,color:#fff
    style IE fill:#6d597a,stroke:#4a3f50,color:#fff
    style AGENT fill:#6d597a,stroke:#4a3f50,color:#fff
    style BMH fill:#6d597a,stroke:#4a3f50,color:#fff
    style NMS fill:#6d597a,stroke:#4a3f50,color:#fff
    style CIS fill:#7d8597,stroke:#5c6378,color:#fff
    style SECRET fill:#7d8597,stroke:#5c6378,color:#fff
    style CP fill:#7d8597,stroke:#5c6378,color:#fff
    style NP fill:#355070,stroke:#1d3557,color:#fff
    style MD fill:#355070,stroke:#1d3557,color:#fff
    
    style clusterdef fill:#c4bfaa,stroke:#7a6a1a,stroke-width:2px,color:#2d2d2d
    style hostmgmt fill:#cfc5b5,stroke:#8d7a5a,stroke-width:2px,color:#2d2d2d
    style supporting fill:#a8b0b8,stroke:#2d4a42,stroke-width:2px,color:#2d2d2d
    style hcpworkers fill:#b8d4d0,stroke:#3d5a52,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
```

## API Groups

| API Group | Owner | Description |
|-----------|-------|-------------|
| `hive.openshift.io` | Hive | Cluster lifecycle |
| `extensions.hive.openshift.io` | Assisted/IBI | Extension APIs |
| `agent-install.openshift.io` | Assisted | Discovery and agents |
| `hypershift.openshift.io` | HyperShift | Hosted control planes |
| `metal3.io` | Baremetal Operator | Bare metal hosts |
| `siteconfig.open-cluster-management.io` | SiteConfig | Template provisioning |
| `machine.openshift.io` | Machine API | Machine management |

## Common Patterns

### Reference Pattern

CRDs reference each other via typed references:

```yaml
spec:
  clusterDeploymentRef:
    name: my-cluster
  imageSetRef:
    name: openshift-4.14
  pullSecretRef:
    name: pull-secret
```

### Status Conditions

Standard condition reporting:

```yaml
status:
  conditions:
    - type: Ready
      status: "True"
      reason: AllChecksPassed
      message: "Resource is ready"
      lastTransitionTime: "2024-01-15T10:30:00Z"
```

### Labels for Selection

Resources use labels for matching:

```yaml
metadata:
  labels:
    cluster-name: my-cluster
    infraenvs.agent-install.openshift.io: my-infraenv
spec:
  agentSelector:
    matchLabels:
      cluster-name: my-cluster
```

## YAML Examples

See the [examples/](examples/) directory for complete YAML examples:

- [ClusterDeployment](examples/clusterdeployment.yaml)
- [AgentClusterInstall](examples/agentclusterinstall.yaml)
- [InfraEnv](examples/infraenv.yaml)
- [NMStateConfig](examples/nmstateconfig.yaml)
- [BareMetalHost](examples/baremetalhost.yaml)

> **Note:** For HostedCluster, NodePool, and ClusterInstance examples, see inline examples in:
> - [HCP Overview](../05-hosted-control-planes/hcp-overview.md)
> - [ZTP Documentation](../06-gitops-provisioning/ztp.md)

## Related Documentation

- [Operators & Controllers Reference](../07-operators-controllers/reference.md)
- [Installation Methods Overview](../01-installation-methods-overview.md)

