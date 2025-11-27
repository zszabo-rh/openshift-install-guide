# Component Diagrams

This document contains comprehensive Mermaid diagrams for each OpenShift installation scenario.

## IPI Installation Flow

```mermaid
sequenceDiagram
    participant User
    participant Installer as openshift-install
    participant Terraform
    participant Cloud as Cloud Provider
    participant Bootstrap
    participant Masters
    participant Workers

    User->>Installer: create cluster
    Installer->>Installer: Generate install-config
    Installer->>Installer: Generate manifests
    Installer->>Installer: Generate Ignition configs
    
    Installer->>Terraform: Create infrastructure
    Terraform->>Cloud: VPC, Subnets, SGs
    Terraform->>Cloud: Load Balancers
    Terraform->>Cloud: DNS Records
    Terraform->>Cloud: Bootstrap VM
    Terraform->>Cloud: Master VMs
    
    Cloud-->>Bootstrap: Boot with bootstrap.ign
    Bootstrap->>Bootstrap: Start temp control plane
    Bootstrap->>Bootstrap: Start MCS on :22623
    
    Cloud-->>Masters: Boot with master.ign
    Masters->>Bootstrap: Fetch config from MCS
    Masters->>Masters: Join etcd cluster
    Masters->>Masters: Start permanent control plane
    
    Bootstrap-->>Installer: Bootstrap complete
    Installer->>Cloud: Destroy bootstrap
    
    Cloud->>Workers: Provision worker VMs
    Workers->>Masters: Join cluster
    
    Installer-->>User: Installation complete
```

## Assisted Installation Flow (On-Premise)

```mermaid
graph TB
    subgraph "Hub Cluster"
        MCE[MCE Operator]
        ASC[AgentServiceConfig]
        
        subgraph "Assisted Components"
            SERVICE[assisted-service]
            IMAGE_SVC[assisted-image-service]
            DB[(PostgreSQL)]
        end
        
        subgraph "Integration"
            HIVE[Hive Controller]
            BMO[Baremetal Operator]
            BMAC[BMAC Controller]
        end
        
        subgraph "CRDs"
            CD[ClusterDeployment]
            ACI[AgentClusterInstall]
            IE[InfraEnv]
            AGENT[Agent]
            BMH[BareMetalHost]
        end
    end
    
    subgraph "Target Hosts"
        H1[Host 1]
        H2[Host 2]
        H3[Host 3]
    end
    
    MCE --> ASC
    ASC --> SERVICE
    ASC --> IMAGE_SVC
    SERVICE --> DB
    
    HIVE --> CD
    CD --> ACI
    ACI --> IE
    
    BMO --> BMH
    BMAC --> BMH
    BMAC --> AGENT
    
    IMAGE_SVC --> H1
    IMAGE_SVC --> H2
    IMAGE_SVC --> H3
    
    H1 --> AGENT
    H2 --> AGENT
    H3 --> AGENT
```

## Agent-Based Installer Flow

```mermaid
sequenceDiagram
    participant User
    participant Installer as openshift-install
    participant ISO as Agent ISO
    participant Rendezvous as Rendezvous Host
    participant Other as Other Hosts
    
    User->>Installer: agent create image
    Installer->>Installer: Parse install-config.yaml
    Installer->>Installer: Parse agent-config.yaml
    Installer->>ISO: Embed assisted-service
    Installer->>ISO: Embed agent
    Installer->>ISO: Embed ZTP manifests
    Installer-->>User: agent.iso
    
    User->>Rendezvous: Boot from ISO
    User->>Other: Boot from ISO
    
    Rendezvous->>Rendezvous: Start embedded service
    Rendezvous->>Rendezvous: Register self
    
    Other->>Rendezvous: Register
    
    Rendezvous->>Rendezvous: Validate all hosts
    Rendezvous->>Rendezvous: Generate Ignition
    
    Rendezvous->>Rendezvous: Write to disk
    Other->>Other: Write to disk
    
    Note over Rendezvous,Other: Reboot
    
    Rendezvous->>Rendezvous: Form cluster
    Other->>Rendezvous: Join cluster
    
    Rendezvous-->>User: Cluster ready
```

## Image-Based Installation Flow

```mermaid
graph TB
    subgraph "Seed Cluster"
        SEED_SNO[Running SNO]
        LCA[Lifecycle Agent]
        SEED[Seed Image<br/>~15-20 GB]
        
        SEED_SNO --> LCA
        LCA -->|Generate| SEED
    end
    
    subgraph "Hub Cluster"
        IBI_OP[IBI Operator]
        
        subgraph "CRDs"
            ICI[ImageClusterInstall]
            CD_IBI[ClusterDeployment]
            BMH_IBI[BareMetalHost]
        end
        
        CONFIG_ISO[Config ISO]
        
        IBI_OP --> ICI
        ICI --> CD_IBI
        ICI --> BMH_IBI
        IBI_OP --> CONFIG_ISO
    end
    
    subgraph "Target Host"
        DISK[(Disk)]
        RECERT[Reconfiguration<br/>via recert]
        NEW_SNO[New SNO Cluster]
        
        SEED -->|Write| DISK
        CONFIG_ISO --> DISK
        DISK --> RECERT
        RECERT --> NEW_SNO
    end
```

## Hosted Control Planes Architecture

```mermaid
graph TB
    subgraph "Management Cluster"
        subgraph "HyperShift Namespace"
            HO[HyperShift Operator]
        end
        
        subgraph "HCP Namespace: clusters-my-hosted"
            CPO[Control Plane Operator]
            ETCD1[etcd-0]
            ETCD2[etcd-1]
            ETCD3[etcd-2]
            KAPI[kube-apiserver]
            KCM[controller-manager]
            SCHED[scheduler]
            IGN[ignition-server]
        end
        
        HC[HostedCluster]
        NP[NodePool]
    end
    
    subgraph "Worker Infrastructure"
        subgraph "CAPI Resources"
            MD[MachineDeployment]
            MS[MachineSet]
            M1[Machine 1]
            M2[Machine 2]
            M3[Machine 3]
        end
        
        W1[Worker Node 1]
        W2[Worker Node 2]
        W3[Worker Node 3]
    end
    
    HO --> HC
    HO --> NP
    HC --> CPO
    CPO --> ETCD1
    CPO --> ETCD2
    CPO --> ETCD3
    CPO --> KAPI
    CPO --> KCM
    CPO --> SCHED
    CPO --> IGN
    
    NP --> MD
    MD --> MS
    MS --> M1
    MS --> M2
    MS --> M3
    
    M1 --> W1
    M2 --> W2
    M3 --> W3
    
    W1 -->|API| KAPI
    W2 -->|API| KAPI
    W3 -->|API| KAPI
    
    IGN -->|Ignition| W1
    IGN -->|Ignition| W2
    IGN -->|Ignition| W3
```

## ZTP with SiteConfig Flow

```mermaid
graph TB
    subgraph "Git Repository"
        SITES[Site Configurations]
        TEMPLATES[Templates]
        POLICIES[Policies]
    end
    
    subgraph "Hub Cluster"
        ARGOCD[ArgoCD]
        
        subgraph "SiteConfig"
            SC_OP[SiteConfig Operator]
            CI[ClusterInstance]
        end
        
        subgraph "Rendered Resources"
            CD[ClusterDeployment]
            ACI[AgentClusterInstall]
            IE[InfraEnv]
            BMH[BareMetalHost]
            NMS[NMStateConfig]
        end
        
        subgraph "Installation"
            ASSISTED[Assisted Service]
        end
        
        subgraph "Post-Install"
            TALM[TALM]
            POLICY[Policies]
        end
    end
    
    subgraph "Edge Sites"
        S1[Site 1 SNO]
        S2[Site 2 SNO]
        S3[Site N SNO]
    end
    
    SITES --> ARGOCD
    TEMPLATES --> ARGOCD
    POLICIES --> ARGOCD
    
    ARGOCD --> SC_OP
    SC_OP --> CI
    CI --> CD
    CI --> ACI
    CI --> IE
    CI --> BMH
    CI --> NMS
    
    CD --> ASSISTED
    ACI --> ASSISTED
    IE --> ASSISTED
    BMH --> ASSISTED
    
    ASSISTED --> S1
    ASSISTED --> S2
    ASSISTED --> S3
    
    ARGOCD --> TALM
    TALM --> POLICY
    POLICY --> S1
    POLICY --> S2
    POLICY --> S3
```

## Controller Watch Relationships

```mermaid
graph TB
    subgraph "CRDs"
        ASC[AgentServiceConfig]
        CD[ClusterDeployment]
        ACI[AgentClusterInstall]
        IE[InfraEnv]
        AGENT[Agent]
        BMH[BareMetalHost]
        NMS[NMStateConfig]
        PPI[PreprovisioningImage]
        AC[AgentClassification]
    end
    
    subgraph "Controllers"
        ASCC[AgentServiceConfig<br/>Controller]
        CDC[ClusterDeployments<br/>Controller]
        IEC[InfraEnv<br/>Controller]
        AGENTC[Agent<br/>Controller]
        BMAC[BMAC<br/>Controller]
        PPIC[PPI<br/>Controller]
        ACC[AgentClassification<br/>Controller]
    end
    
    ASC --> ASCC
    
    CD --> CDC
    ACI -.->|watches| CDC
    AGENT -.->|watches| CDC
    
    IE --> IEC
    NMS -.->|watches| IEC
    CD -.->|watches| IEC
    
    AGENT --> AGENTC
    
    BMH --> BMAC
    AGENT -.->|watches| BMAC
    IE -.->|watches| BMAC
    CD -.->|watches| BMAC
    
    PPI --> PPIC
    IE -.->|watches| PPIC
    BMH -.->|watches| PPIC
    
    AC --> ACC
    AGENT -.->|watches| ACC
    
    style ASCC fill:#f9f,stroke:#333
    style CDC fill:#f9f,stroke:#333
    style IEC fill:#f9f,stroke:#333
    style AGENTC fill:#f9f,stroke:#333
    style BMAC fill:#f9f,stroke:#333
    style PPIC fill:#f9f,stroke:#333
    style ACC fill:#f9f,stroke:#333
```

## CRD Ownership and References

```mermaid
graph LR
    subgraph "Cluster Definition"
        CI[ClusterInstance]
        CD[ClusterDeployment]
        ACI[AgentClusterInstall]
        ICI[ImageClusterInstall]
    end
    
    subgraph "Infrastructure"
        IE[InfraEnv]
        BMH[BareMetalHost]
        NMS[NMStateConfig]
        AGENT[Agent]
    end
    
    subgraph "Supporting"
        CIS[ClusterImageSet]
        SECRET[Pull Secret]
    end
    
    CI -->|renders| CD
    CI -->|renders| ACI
    CI -->|renders| IE
    CI -->|renders| BMH
    CI -->|renders| NMS
    
    CD -->|refs| ACI
    CD -->|refs| ICI
    CD -->|refs| SECRET
    
    ACI -->|refs| CIS
    ACI -->|refs| IE
    
    ICI -->|refs| CIS
    ICI -->|refs| BMH
    
    IE -->|selects| NMS
    IE -->|creates| AGENT
    
    BMH -->|matched by| AGENT
    
    linkStyle 0,1,2,3,4 stroke:#00f,stroke-width:2px
    linkStyle 5,6,7,8,9,10,11,12,13,14 stroke:#090,stroke-width:1px
```

## State Machine: Cluster Installation

```mermaid
stateDiagram-v2
    [*] --> PendingForInput: Cluster Created
    
    PendingForInput --> Insufficient: Partial Config
    PendingForInput --> Ready: Full Config
    
    Insufficient --> Ready: All Validations Pass
    Ready --> Insufficient: Validation Fails
    
    Ready --> PreparingForInstallation: User Triggers Install
    PreparingForInstallation --> Installing: Ignition Ready
    Installing --> Finalizing: Hosts Installed
    Finalizing --> Installed: Operators Ready
    
    Installing --> Error: Fatal Error
    Finalizing --> Error: Timeout
    
    Ready --> Cancelled: User Cancels
    Installing --> Cancelled: User Cancels
```

## State Machine: Host Discovery

```mermaid
stateDiagram-v2
    [*] --> Discovering: Host Boots
    
    Discovering --> Known: Validations Pass
    Discovering --> Insufficient: Validations Fail
    Discovering --> PendingForInput: Needs Config
    
    PendingForInput --> Known: Config Provided
    Insufficient --> Known: Issues Fixed
    Known --> Insufficient: New Issues
    
    Known --> Disconnected: Heartbeat Timeout
    Disconnected --> Known: Reconnects
    
    Known --> PreparingForInstallation: Install Triggered
    PreparingForInstallation --> Installing: Disk Ready
    Installing --> InstallingInProgress: Writing
    InstallingInProgress --> Installed: Success
    InstallingInProgress --> Error: Failure
    InstallingInProgress --> InstallingPendingUserAction: Boot Order Issue
    
    Known --> Disabled: User Disables
    Disabled --> Known: Re-enabled
```

## Comparison: Installation Methods

```mermaid
graph TD
    subgraph "Decision Factors"
        D1[Connectivity?]
        D2[Hub Cluster?]
        D3[Cluster Type?]
        D4[Scale?]
    end
    
    subgraph "Methods"
        IPI[IPI]
        UPI[UPI]
        SAAS[Assisted SaaS]
        MCE[Assisted MCE]
        ABI[ABI]
        IBI[IBI]
        HCP[HCP]
        ZTP[ZTP]
    end
    
    D1 -->|Connected + Cloud| IPI
    D1 -->|Connected + Custom| UPI
    D1 -->|Connected + Simple| SAAS
    D1 -->|Any + Hub| MCE
    D1 -->|Disconnected| ABI
    D1 -->|Disconnected + Fast SNO| IBI
    D2 -->|Yes + Multi-tenant| HCP
    D4 -->|Large Scale| ZTP
```

## Related Documentation

- [Installation Methods Overview](../01-installation-methods-overview.md)
- [Operators & Controllers Reference](../07-operators-controllers/reference.md)
- [CRD Reference](../08-crd-reference/index.md)

