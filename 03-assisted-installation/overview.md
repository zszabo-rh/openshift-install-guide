# Assisted Installation Overview

The Assisted Installer is an installation method that simplifies OpenShift deployment through hardware discovery, validation, and guided installation. Unlike traditional installers, it provides real-time feedback about hardware compatibility and installation progress.

## Core Concepts

```mermaid
graph TB
    subgraph components["Assisted Installer Components"]
        SERVICE[assisted-service<br/>API & Business Logic]
        IMAGE_SVC[assisted-image-service<br/>ISO Generation]
        DB[(PostgreSQL)]
        S3[(Object Storage)]
    end
    
    subgraph hosts["On Target Hosts"]
        AGENT[assisted-installer-agent<br/>Discovery & Installation]
        CONTROLLER[assisted-controller<br/>Post-boot coordination]
    end
    
    subgraph ui["User Interfaces"]
        UI[Web Console]
        CLI[CLI / API]
        KUBE[Kubernetes CRDs]
    end
    
    UI --> SERVICE
    CLI --> SERVICE
    KUBE --> SERVICE
    
    SERVICE --> DB
    SERVICE --> S3
    SERVICE --> IMAGE_SVC
    
    IMAGE_SVC -->|ISO| AGENT
    AGENT -->|Hardware Info| SERVICE
    AGENT -->|Installs| CONTROLLER
    
    style SERVICE fill:#6d597a,stroke:#4a3f50,color:#fff
    style IMAGE_SVC fill:#6d597a,stroke:#4a3f50,color:#fff
    style DB fill:#7d8597,stroke:#5c6378,color:#fff
    style S3 fill:#7d8597,stroke:#5c6378,color:#fff
    style AGENT fill:#52796f,stroke:#354f52,color:#fff
    style CONTROLLER fill:#52796f,stroke:#354f52,color:#fff
    style UI fill:#355070,stroke:#1d3557,color:#fff
    style CLI fill:#355070,stroke:#1d3557,color:#fff
    style KUBE fill:#355070,stroke:#1d3557,color:#fff
    
    style components fill:#c4bfaa,stroke:#7a6a1a,stroke-width:2px,color:#2d2d2d
    style hosts fill:#b8d4d0,stroke:#3d5a52,stroke-width:2px,color:#2d2d2d
    style ui fill:#a8b0b8,stroke:#2d4a42,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
```

## How It Works

### 1. Discovery Phase

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'lineColor': '#2d3748', 'actorLineColor': '#2d3748' }}}%%
sequenceDiagram
    box rgb(190,184,168) Users & Services
        participant User
        participant Service as Assisted Service
        participant ISO as Image Service
    end
    
    box rgb(180,175,160) Target Infrastructure
        participant Host as Target Host
        participant Agent
    end
    
    User->>Service: Create Cluster & InfraEnv
    Service->>ISO: Request Discovery ISO
    ISO-->>User: Discovery ISO URL
    User->>Host: Boot from ISO
    Host->>Agent: Agent starts automatically
    Agent->>Agent: Collect hardware inventory
    Agent->>Service: Register host
    Agent->>Agent: Run connectivity checks
    Agent->>Service: Report validations
    loop Every 60 seconds
        Agent->>Service: Heartbeat + updates
    end
```

### 2. Validation Phase

The service validates hosts against requirements:

| Validation Category | Checks |
|--------------------|--------|
| **Hardware** | CPU cores, RAM, disk size, disk speed |
| **Network** | Connectivity between hosts, DNS resolution, NTP sync |
| **Platform** | Virtualization support, required capabilities |
| **Cluster** | Sufficient control plane nodes, network CIDR validity |

### 3. Installation Phase

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'lineColor': '#2d3748', 'actorLineColor': '#2d3748' }}}%%
sequenceDiagram
    box rgb(190,184,168) User & Service
        participant User
        participant Service as Assisted Service
    end
    
    box rgb(180,175,160) Target Host
        participant Agent
        participant Host
    end
    
    User->>Service: Trigger installation
    Service->>Service: Generate Ignition configs
    Service->>Agent: Instruction: Install
    Agent->>Host: Write OS to disk
    Agent->>Host: Apply Ignition
    Agent->>Host: Reboot
    Host->>Host: Boot from disk
    Host->>Host: Form cluster
    Note over Host: assisted-controller takes over
    Host->>Service: Report progress
    Service-->>User: Installation complete
```

## Components Deep Dive

### assisted-service

**Repository:** [openshift/assisted-service](https://github.com/openshift/assisted-service)

The brain of the Assisted Installer. It provides:

- **REST API** for cluster and host management ([swagger.yaml](https://github.com/openshift/assisted-service/blob/master/swagger.yaml))
- **State machines** for cluster and host lifecycle
- **Validation engine** for pre-flight checks
- **Ignition generation** using openshift-install
- **Kubernetes controllers** (in on-premise mode)

**Key packages:**
- [`internal/bminventory/`](https://github.com/openshift/assisted-service/tree/master/internal/bminventory) - Main API handlers
- [`internal/cluster/`](https://github.com/openshift/assisted-service/tree/master/internal/cluster) - Cluster state machine
- [`internal/host/`](https://github.com/openshift/assisted-service/tree/master/internal/host) - Host state machine
- [`internal/controller/`](https://github.com/openshift/assisted-service/tree/master/internal/controller) - Kubernetes controllers

### assisted-image-service

**Repository:** [openshift/assisted-image-service](https://github.com/openshift/assisted-image-service)

Generates customized discovery ISOs:

- Embeds cluster-specific configuration
- Injects pull secrets and certificates
- Supports minimal and full ISO types
- Streams images directly (no storage needed)

### assisted-installer-agent

**Repository:** [openshift/assisted-installer-agent](https://github.com/openshift/assisted-installer-agent)

Runs on discovered hosts:

- Collects hardware inventory (CPU, RAM, disks, NICs)
- Performs network connectivity tests
- Executes installation steps
- Reports status to service

**Discovery data collected:**
- CPU architecture and features
- Memory size and configuration
- Block devices and their properties
- Network interfaces and addresses
- System vendor and product info

### assisted-controller

**Repository:** [openshift/assisted-installer](https://github.com/openshift/assisted-installer) (contains assisted-installer-controller)

A Kubernetes Job that runs post-installation:

- Monitors cluster formation
- Approves Certificate Signing Requests (CSRs)
- Reports installation progress
- Collects and uploads logs

## Deployment Modes

### SaaS (console.redhat.com)

```mermaid
graph LR
    subgraph cloud["Red Hat Cloud"]
        CONSOLE[console.redhat.com]
        SERVICE[Assisted Service]
        DB[(Database)]
    end
    
    subgraph customer["Customer Site"]
        HOSTS[Target Hosts]
        AGENT[Agents]
    end
    
    CONSOLE --> SERVICE
    SERVICE --> DB
    AGENT -->|HTTPS| SERVICE
    HOSTS --> AGENT
    
    style CONSOLE fill:#b56576,stroke:#8d4e5a,color:#fff
    style SERVICE fill:#6d597a,stroke:#4a3f50,color:#fff
    style DB fill:#7d8597,stroke:#5c6378,color:#fff
    style HOSTS fill:#52796f,stroke:#354f52,color:#fff
    style AGENT fill:#52796f,stroke:#354f52,color:#fff
    
    style cloud fill:#cfc5b5,stroke:#8d7a5a,stroke-width:2px,color:#2d2d2d
    style customer fill:#b8d4d0,stroke:#3d5a52,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
```

**Characteristics:**
- No hub cluster required
- Red Hat manages the service
- Internet connectivity required
- REST API only

### On-Premise (via MCE)

```mermaid
graph LR
    subgraph hub["Hub Cluster"]
        MCE[MCE Operator]
        HIVE[Hive]
        ASSISTED[Assisted Service]
        BMO[Baremetal Operator]
    end
    
    subgraph target["Target Cluster"]
        HOSTS[Target Hosts]
        AGENT[Agents]
    end
    
    MCE --> ASSISTED
    MCE --> HIVE
    HIVE --> ASSISTED
    ASSISTED --> BMO
    AGENT --> ASSISTED
    
    style MCE fill:#6d597a,stroke:#4a3f50,color:#fff
    style HIVE fill:#6d597a,stroke:#4a3f50,color:#fff
    style ASSISTED fill:#6d597a,stroke:#4a3f50,color:#fff
    style BMO fill:#6d597a,stroke:#4a3f50,color:#fff
    style HOSTS fill:#52796f,stroke:#354f52,color:#fff
    style AGENT fill:#52796f,stroke:#354f52,color:#fff
    
    style hub fill:#c4bfaa,stroke:#7a6a1a,stroke-width:2px,color:#2d2d2d
    style target fill:#b8d4d0,stroke:#3d5a52,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
```

**Characteristics:**
- Runs on your OpenShift hub
- Works in disconnected environments
- Uses Kubernetes CRDs
- Integrates with Hive for lifecycle

[Detailed comparison →](saas-vs-onprem.md)

## State Machines

### Host States

```mermaid
flowchart LR
    subgraph background[" "]
        direction LR
        
        START(( )) -->|Host boots| Discovering
        
        Discovering -->|Missing config| PendingForInput
        Discovering -->|Validations pass| Known
        Discovering -->|Validations fail| Insufficient
        
        PendingForInput -->|Config provided| Known
        Insufficient -->|Issues fixed| Known
        Known -->|New issues| Insufficient
        
        Known -->|Heartbeat timeout| Disconnected
        Disconnected -->|Reconnects| Known
        
        Known -->|Install triggered| PreparingFor[PreparingFor<br/>Installation]
        PreparingFor -->|Disk ready| Installing
        Installing -->|Writing| InProgress[Installing<br/>InProgress]
        InProgress -->|Boot order issue| PendingAction[Pending<br/>UserAction]
        InProgress -->|Success| Installed
        InProgress -->|Failure| Error
        
        PendingAction -->|Fixed| InProgress
        
        Known -->|User disabled| Disabled
        Disabled -->|Re-enabled| Known
    end
    
    style START fill:#2d3748,stroke:#2d3748
    style Discovering fill:#adb5bd,stroke:#6c757d,color:#000
    style PendingForInput fill:#adb5bd,stroke:#6c757d,color:#000
    style Known fill:#adb5bd,stroke:#6c757d,color:#000
    style Insufficient fill:#adb5bd,stroke:#6c757d,color:#000
    style Disconnected fill:#e9c46a,stroke:#c9a227,color:#000
    style PreparingFor fill:#457b9d,stroke:#1d3557,color:#fff
    style Installing fill:#457b9d,stroke:#1d3557,color:#fff
    style InProgress fill:#457b9d,stroke:#1d3557,color:#fff
    style PendingAction fill:#e9c46a,stroke:#c9a227,color:#000
    style Installed fill:#52796f,stroke:#354f52,color:#fff
    style Error fill:#c9184a,stroke:#a4133c,color:#fff
    style Disabled fill:#adb5bd,stroke:#6c757d,color:#000
    
    style background fill:#beb8a8,stroke:#706858,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
```

### Cluster States

```mermaid
flowchart LR
    subgraph background[" "]
        direction LR
        
        START(( )) -->|Cluster created| PendingForInput
        
        PendingForInput -->|Partial config| Insufficient
        PendingForInput -->|Full config| Ready
        Insufficient -->|Validations pass| Ready
        Ready -->|Validation fails| Insufficient
        
        Ready -->|Install triggered| Preparing[PreparingFor<br/>Installation]
        Preparing -->|Ignition ready| Installing
        Installing -->|Hosts installed| Finalizing
        Finalizing -->|Operators ready| Installed
        
        Installing -->|Fatal error| Error
        Finalizing -->|Timeout| Error
    end
    
    style START fill:#2d3748,stroke:#2d3748
    style PendingForInput fill:#adb5bd,stroke:#6c757d,color:#000
    style Insufficient fill:#adb5bd,stroke:#6c757d,color:#000
    style Ready fill:#adb5bd,stroke:#6c757d,color:#000
    style Preparing fill:#457b9d,stroke:#1d3557,color:#fff
    style Installing fill:#457b9d,stroke:#1d3557,color:#fff
    style Finalizing fill:#457b9d,stroke:#1d3557,color:#fff
    style Installed fill:#52796f,stroke:#354f52,color:#fff
    style Error fill:#c9184a,stroke:#a4133c,color:#fff
    
    style background fill:#beb8a8,stroke:#706858,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
    linkStyle 9,10 stroke:#9b2c2c,stroke-width:2px
```

## Key Features

### Hardware Discovery

The agent collects comprehensive hardware information:

```json
{
  "cpu": {
    "architecture": "x86_64",
    "model_name": "Intel Xeon E5-2680",
    "count": 32,
    "flags": ["vmx", "sse4_2", "avx2"]
  },
  "memory": {
    "physical_bytes": 137438953472,
    "usable_bytes": 134217728000
  },
  "disks": [{
    "name": "sda",
    "size_bytes": 480103981056,
    "drive_type": "SSD",
    "bootable": true
  }],
  "interfaces": [{
    "name": "eno1",
    "mac_address": "00:11:22:33:44:55",
    "ipv4_addresses": ["192.168.1.10/24"]
  }]
}
```

### Network Validations

Agents test connectivity to each other:

```mermaid
graph TB
    subgraph matrix["Connectivity Matrix"]
        H1[Host 1] <-->|L3 + L2| H2[Host 2]
        H2 <-->|L3 + L2| H3[Host 3]
        H3 <-->|L3 + L2| H1
    end
    
    subgraph external["External Checks"]
        H1 -->|NTP| NTP[NTP Server]
        H1 -->|DNS| DNS[DNS Server]
        H1 -->|API| API[api.cluster.example.com]
    end
    
    style H1 fill:#52796f,stroke:#354f52,color:#fff
    style H2 fill:#52796f,stroke:#354f52,color:#fff
    style H3 fill:#52796f,stroke:#354f52,color:#fff
    style NTP fill:#b56576,stroke:#8d4e5a,color:#fff
    style DNS fill:#b56576,stroke:#8d4e5a,color:#fff
    style API fill:#b56576,stroke:#8d4e5a,color:#fff
    
    style matrix fill:#c4bfaa,stroke:#7a6a1a,stroke-width:2px,color:#2d2d2d
    style external fill:#cfc5b5,stroke:#8d7a5a,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
```

### Static Network Configuration

For environments requiring static IPs, use NMState:

```yaml
apiVersion: agent-install.openshift.io/v1beta1
kind: NMStateConfig
metadata:
  name: host-static-config
  labels:
    infraenvs.agent-install.openshift.io: my-infraenv
spec:
  config:
    interfaces:
      - name: eno1
        type: ethernet
        state: up
        ipv4:
          address:
            - ip: 192.168.1.10
              prefix-length: 24
          dhcp: false
          enabled: true
    routes:
      config:
        - destination: 0.0.0.0/0
          next-hop-address: 192.168.1.1
          next-hop-interface: eno1
    dns-resolver:
      config:
        server:
          - 192.168.1.1
  interfaces:
    - name: eno1
      macAddress: "00:11:22:33:44:55"
```

## Cluster Types Supported

| Type | Control Plane | Workers | Notes |
|------|--------------|---------|-------|
| **Standard HA** | 3 | 2+ | Production recommended |
| **Compact** | 3 | 0 | Control plane runs workloads |
| **SNO** | 1 | 0 | Single Node OpenShift |
| **Day 2 Workers** | N/A | 1+ | Add to existing cluster |

## Integration Points

### With Hive (On-Premise)

The Assisted Service integrates with [Hive](https://github.com/openshift/hive) for:
- ClusterDeployment as cluster definition
- Automatic cluster import after installation
- ClusterPool support

See [Hive integration docs](https://github.com/openshift/assisted-service/tree/master/docs/hive-integration).

### With Baremetal Operator

When [BMO](https://github.com/metal3-io/baremetal-operator) is present, the [Baremetal Agent Controller (BMAC)](https://github.com/openshift/assisted-service/blob/master/internal/controller/controllers/bmh_agent_controller.go):
- Syncs BareMetalHost with Agent CRs
- Automates ISO attachment
- Manages host power state

### With SiteConfig/ZTP

[SiteConfig](https://github.com/stolostron/siteconfig) renders Assisted CRDs for GitOps workflows:
- ClusterInstance → ClusterDeployment + AgentClusterInstall + InfraEnv
- Enables fleet-scale deployments

## Further Reading

### Detailed Documentation

The [assisted-service repository](https://github.com/openshift/assisted-service/tree/master/docs) contains comprehensive documentation:

| Topic | Link |
|-------|------|
| **Architecture overview** | [docs/architecture.md](https://github.com/openshift/assisted-service/blob/master/docs/architecture.md) |
| **User guide** | [docs/user-guide/](https://github.com/openshift/assisted-service/tree/master/docs/user-guide) |
| **Hive/Kube-API integration** | [docs/hive-integration/](https://github.com/openshift/assisted-service/tree/master/docs/hive-integration) |
| **Network configuration** | [docs/user-guide/network-configuration/](https://github.com/openshift/assisted-service/tree/master/docs/user-guide/network-configuration) |
| **Developer documentation** | [docs/dev/](https://github.com/openshift/assisted-service/tree/master/docs/dev) |
| **Enhancement proposals** | [docs/enhancements/](https://github.com/openshift/assisted-service/tree/master/docs/enhancements) |

### Next Steps in This Guide

- [SaaS vs On-Premise comparison](saas-vs-onprem.md)
- [REST API vs Kubernetes API](rest-api-vs-kube-api.md)
- [Agent-Based Installer](abi.md)
- [Component architecture](components.md)

