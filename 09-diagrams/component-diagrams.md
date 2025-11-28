# Component Diagrams

This document contains comprehensive Mermaid diagrams for each OpenShift installation scenario.

## IPI Installation Flow

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'lineColor': '#2d3748', 'actorLineColor': '#2d3748' }}}%%
sequenceDiagram
    box rgb(190,184,168) User & Build Tools
        participant User
        participant Installer as openshift-install
        participant Terraform
    end
    
    box rgb(180,175,160) Infrastructure
        participant Cloud as Cloud Provider
        participant Bootstrap
        participant Masters
        participant Workers
    end

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
    subgraph hub["Hub Cluster"]
        MCE[MCE Operator]
        ASC[AgentServiceConfig]
        
        subgraph assisted["Assisted Components"]
            SERVICE[assisted-service]
            IMAGE_SVC[assisted-image-service]
            DB[(PostgreSQL)]
        end
        
        subgraph integration["Integration"]
            HIVE[Hive Controller]
            BMO[Baremetal Operator]
            BMAC[BMAC Controller]
        end
        
        subgraph crds["CRDs"]
            CD[ClusterDeployment]
            ACI[AgentClusterInstall]
            IE[InfraEnv]
            AGENT[Agent]
            BMH[BareMetalHost]
        end
    end
    
    subgraph hosts["Target Hosts"]
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
    
    style MCE fill:#6d597a,stroke:#4a3f50,color:#fff
    style ASC fill:#355070,stroke:#1d3557,color:#fff
    style SERVICE fill:#6d597a,stroke:#4a3f50,color:#fff
    style IMAGE_SVC fill:#6d597a,stroke:#4a3f50,color:#fff
    style DB fill:#7d8597,stroke:#5c6378,color:#fff
    style HIVE fill:#6d597a,stroke:#4a3f50,color:#fff
    style BMO fill:#6d597a,stroke:#4a3f50,color:#fff
    style BMAC fill:#6d597a,stroke:#4a3f50,color:#fff
    style CD fill:#355070,stroke:#1d3557,color:#fff
    style ACI fill:#355070,stroke:#1d3557,color:#fff
    style IE fill:#355070,stroke:#1d3557,color:#fff
    style AGENT fill:#355070,stroke:#1d3557,color:#fff
    style BMH fill:#355070,stroke:#1d3557,color:#fff
    style H1 fill:#52796f,stroke:#354f52,color:#fff
    style H2 fill:#52796f,stroke:#354f52,color:#fff
    style H3 fill:#52796f,stroke:#354f52,color:#fff
    
    style hub fill:#beb8a8,stroke:#706858,stroke-width:2px,color:#2d2d2d
    style assisted fill:#c4bfaa,stroke:#7a6a1a,stroke-width:2px,color:#2d2d2d
    style integration fill:#cfc5b5,stroke:#8d7a5a,stroke-width:2px,color:#2d2d2d
    style crds fill:#b8d4d0,stroke:#3d5a52,stroke-width:2px,color:#2d2d2d
    style hosts fill:#a8b0b8,stroke:#2d4a42,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
```

## Agent-Based Installer Flow

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'lineColor': '#2d3748', 'actorLineColor': '#2d3748' }}}%%
sequenceDiagram
    box rgb(190,184,168) User & Build
        participant User
        participant Installer as openshift-install
        participant ISO as Agent ISO
    end
    
    box rgb(180,175,160) Target Infrastructure
        participant Rendezvous as Rendezvous Host
        participant Other as Other Hosts
    end
    
    User->>Installer: agent create image
    Installer->>Installer: Parse install-config.yaml
    Installer->>Installer: Parse agent-config.yaml
    Installer->>ISO: Embed coordination service
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
    subgraph seed["Seed Cluster"]
        SEED_SNO[Running SNO]
        LCA[Lifecycle Agent]
        SEED[Seed Image<br/>~15-20 GB]
        
        SEED_SNO --> LCA
        LCA -->|Generate| SEED
    end
    
    subgraph hub["Hub Cluster"]
        IBI_OP[IBI Operator]
        
        subgraph crds["CRDs"]
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
    
    subgraph target["Target Host"]
        DISK[(Disk)]
        RECERT[Reconfiguration<br/>via recert]
        NEW_SNO[New SNO Cluster]
        
        SEED -->|Write| DISK
        CONFIG_ISO --> DISK
        DISK --> RECERT
        RECERT --> NEW_SNO
    end
    
    style SEED_SNO fill:#52796f,stroke:#354f52,color:#fff
    style LCA fill:#6d597a,stroke:#4a3f50,color:#fff
    style SEED fill:#7d8597,stroke:#5c6378,color:#fff
    style IBI_OP fill:#6d597a,stroke:#4a3f50,color:#fff
    style ICI fill:#355070,stroke:#1d3557,color:#fff
    style CD_IBI fill:#355070,stroke:#1d3557,color:#fff
    style BMH_IBI fill:#355070,stroke:#1d3557,color:#fff
    style CONFIG_ISO fill:#355070,stroke:#1d3557,color:#fff
    style DISK fill:#7d8597,stroke:#5c6378,color:#fff
    style RECERT fill:#6d597a,stroke:#4a3f50,color:#fff
    style NEW_SNO fill:#52796f,stroke:#354f52,color:#fff
    
    style seed fill:#cfc5b5,stroke:#8d7a5a,stroke-width:2px,color:#2d2d2d
    style hub fill:#c4bfaa,stroke:#7a6a1a,stroke-width:2px,color:#2d2d2d
    style crds fill:#b8d4d0,stroke:#3d5a52,stroke-width:2px,color:#2d2d2d
    style target fill:#a8b0b8,stroke:#2d4a42,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
```

## Hosted Control Planes Architecture

```mermaid
graph TB
    subgraph mgmt["Management Cluster"]
        subgraph hypershiftns["HyperShift Namespace"]
            HO[HyperShift Operator]
        end
        
        subgraph hcpns["HCP Namespace: clusters-my-hosted"]
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
    
    subgraph workers["Worker Infrastructure"]
        subgraph capi["CAPI Resources"]
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
    
    style HO fill:#6d597a,stroke:#4a3f50,color:#fff
    style CPO fill:#6d597a,stroke:#4a3f50,color:#fff
    style ETCD1 fill:#7d8597,stroke:#5c6378,color:#fff
    style ETCD2 fill:#7d8597,stroke:#5c6378,color:#fff
    style ETCD3 fill:#7d8597,stroke:#5c6378,color:#fff
    style KAPI fill:#6d597a,stroke:#4a3f50,color:#fff
    style KCM fill:#6d597a,stroke:#4a3f50,color:#fff
    style SCHED fill:#6d597a,stroke:#4a3f50,color:#fff
    style IGN fill:#6d597a,stroke:#4a3f50,color:#fff
    style HC fill:#355070,stroke:#1d3557,color:#fff
    style NP fill:#355070,stroke:#1d3557,color:#fff
    style MD fill:#355070,stroke:#1d3557,color:#fff
    style MS fill:#355070,stroke:#1d3557,color:#fff
    style M1 fill:#355070,stroke:#1d3557,color:#fff
    style M2 fill:#355070,stroke:#1d3557,color:#fff
    style M3 fill:#355070,stroke:#1d3557,color:#fff
    style W1 fill:#52796f,stroke:#354f52,color:#fff
    style W2 fill:#52796f,stroke:#354f52,color:#fff
    style W3 fill:#52796f,stroke:#354f52,color:#fff
    
    style mgmt fill:#beb8a8,stroke:#706858,stroke-width:2px,color:#2d2d2d
    style hypershiftns fill:#c4bfaa,stroke:#7a6a1a,stroke-width:2px,color:#2d2d2d
    style hcpns fill:#cfc5b5,stroke:#8d7a5a,stroke-width:2px,color:#2d2d2d
    style workers fill:#b8d4d0,stroke:#3d5a52,stroke-width:2px,color:#2d2d2d
    style capi fill:#a8b0b8,stroke:#2d4a42,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
```

## ZTP with SiteConfig Flow

```mermaid
graph TB
    subgraph git["Git Repository"]
        SITES[Site Configurations]
        TEMPLATES[Templates]
        POLICIES[Policies]
    end
    
    subgraph hub["Hub Cluster"]
        ARGOCD[ArgoCD]
        
        subgraph siteconfig["SiteConfig"]
            SC_OP[SiteConfig Operator]
            CI[ClusterInstance]
        end
        
        subgraph rendered["Rendered Resources"]
            CD[ClusterDeployment]
            ACI[AgentClusterInstall]
            IE[InfraEnv]
            BMH[BareMetalHost]
            NMS[NMStateConfig]
        end
        
        subgraph installation["Installation"]
            ASSISTED[Assisted Service]
        end
        
        subgraph postinstall["Post-Install"]
            TALM[TALM]
            POLICY[Policies]
        end
    end
    
    subgraph edge["Edge Sites"]
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
    
    style SITES fill:#355070,stroke:#1d3557,color:#fff
    style TEMPLATES fill:#355070,stroke:#1d3557,color:#fff
    style POLICIES fill:#355070,stroke:#1d3557,color:#fff
    style ARGOCD fill:#6d597a,stroke:#4a3f50,color:#fff
    style SC_OP fill:#6d597a,stroke:#4a3f50,color:#fff
    style CI fill:#b56576,stroke:#8d4e5a,color:#fff
    style CD fill:#355070,stroke:#1d3557,color:#fff
    style ACI fill:#355070,stroke:#1d3557,color:#fff
    style IE fill:#355070,stroke:#1d3557,color:#fff
    style BMH fill:#355070,stroke:#1d3557,color:#fff
    style NMS fill:#355070,stroke:#1d3557,color:#fff
    style ASSISTED fill:#6d597a,stroke:#4a3f50,color:#fff
    style TALM fill:#6d597a,stroke:#4a3f50,color:#fff
    style POLICY fill:#355070,stroke:#1d3557,color:#fff
    style S1 fill:#52796f,stroke:#354f52,color:#fff
    style S2 fill:#52796f,stroke:#354f52,color:#fff
    style S3 fill:#52796f,stroke:#354f52,color:#fff
    
    style git fill:#a8b0b8,stroke:#2d4a42,stroke-width:2px,color:#2d2d2d
    style hub fill:#beb8a8,stroke:#706858,stroke-width:2px,color:#2d2d2d
    style siteconfig fill:#c4bfaa,stroke:#7a6a1a,stroke-width:2px,color:#2d2d2d
    style rendered fill:#cfc5b5,stroke:#8d7a5a,stroke-width:2px,color:#2d2d2d
    style installation fill:#b8d4d0,stroke:#3d5a52,stroke-width:2px,color:#2d2d2d
    style postinstall fill:#a8b0b8,stroke:#2d4a42,stroke-width:2px,color:#2d2d2d
    style edge fill:#c4bfaa,stroke:#7a6a1a,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
```

## Controller Watch Relationships

```mermaid
graph TB
    subgraph crds["CRDs"]
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
    
    subgraph controllers["Controllers"]
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
    
    style ASC fill:#355070,stroke:#1d3557,color:#fff
    style CD fill:#355070,stroke:#1d3557,color:#fff
    style ACI fill:#355070,stroke:#1d3557,color:#fff
    style IE fill:#355070,stroke:#1d3557,color:#fff
    style AGENT fill:#355070,stroke:#1d3557,color:#fff
    style BMH fill:#355070,stroke:#1d3557,color:#fff
    style NMS fill:#355070,stroke:#1d3557,color:#fff
    style PPI fill:#355070,stroke:#1d3557,color:#fff
    style AC fill:#355070,stroke:#1d3557,color:#fff
    style ASCC fill:#6d597a,stroke:#4a3f50,color:#fff
    style CDC fill:#6d597a,stroke:#4a3f50,color:#fff
    style IEC fill:#6d597a,stroke:#4a3f50,color:#fff
    style AGENTC fill:#6d597a,stroke:#4a3f50,color:#fff
    style BMAC fill:#6d597a,stroke:#4a3f50,color:#fff
    style PPIC fill:#6d597a,stroke:#4a3f50,color:#fff
    style ACC fill:#6d597a,stroke:#4a3f50,color:#fff
    
    style crds fill:#b8d4d0,stroke:#3d5a52,stroke-width:2px,color:#2d2d2d
    style controllers fill:#c4bfaa,stroke:#7a6a1a,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
```

## CRD Ownership and References

```mermaid
graph LR
    subgraph clusterdef["Cluster Definition"]
        CI[ClusterInstance]
        CD[ClusterDeployment]
        ACI[AgentClusterInstall]
        ICI[ImageClusterInstall]
    end
    
    subgraph infra["Infrastructure"]
        IE[InfraEnv]
        BMH[BareMetalHost]
        NMS[NMStateConfig]
        AGENT[Agent]
    end
    
    subgraph supporting["Supporting"]
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
    
    style CI fill:#b56576,stroke:#8d4e5a,color:#fff
    style CD fill:#355070,stroke:#1d3557,color:#fff
    style ACI fill:#355070,stroke:#1d3557,color:#fff
    style ICI fill:#355070,stroke:#1d3557,color:#fff
    style IE fill:#6d597a,stroke:#4a3f50,color:#fff
    style BMH fill:#6d597a,stroke:#4a3f50,color:#fff
    style NMS fill:#6d597a,stroke:#4a3f50,color:#fff
    style AGENT fill:#6d597a,stroke:#4a3f50,color:#fff
    style CIS fill:#7d8597,stroke:#5c6378,color:#fff
    style SECRET fill:#7d8597,stroke:#5c6378,color:#fff
    
    style clusterdef fill:#c4bfaa,stroke:#7a6a1a,stroke-width:2px,color:#2d2d2d
    style infra fill:#cfc5b5,stroke:#8d7a5a,stroke-width:2px,color:#2d2d2d
    style supporting fill:#b8d4d0,stroke:#3d5a52,stroke-width:2px,color:#2d2d2d
    
    linkStyle 0,1,2,3,4 stroke:#8a2a3a,stroke-width:3px
    linkStyle 5,6,7,8,9,10,11,12,13,14 stroke:#2d3748,stroke-width:2px
```

## State Machine: Cluster Installation

```mermaid
flowchart LR
    subgraph background[" "]
        direction LR
        
        START(( )) -->|Cluster Created| PendingForInput
        
        PendingForInput -->|Partial Config| Insufficient
        PendingForInput -->|Full Config| Ready
        Insufficient -->|Validations Pass| Ready
        Ready -->|Validation Fails| Insufficient
        
        Ready -->|User Triggers| Preparing[PreparingFor<br/>Installation]
        Preparing -->|Ignition Ready| Installing
        Installing -->|Hosts Installed| Finalizing
        Finalizing -->|Operators Ready| Installed
        
        Installing -->|Fatal Error| Error
        Finalizing -->|Timeout| Error
        
        Ready -->|User Cancels| Cancelled
        Installing -->|User Cancels| Cancelled
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
    style Cancelled fill:#e9c46a,stroke:#c9a227,color:#000
    
    style background fill:#beb8a8,stroke:#706858,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
    linkStyle 9,10 stroke:#9b2c2c,stroke-width:2px
```

## State Machine: Host Discovery

```mermaid
flowchart LR
    subgraph background[" "]
        direction LR
        
        START(( )) -->|Host Boots| Discovering
        
        Discovering -->|Validations Pass| Known
        Discovering -->|Validations Fail| Insufficient
        Discovering -->|Needs Config| PendingForInput
        
        PendingForInput -->|Config Provided| Known
        Insufficient -->|Issues Fixed| Known
        Known -->|New Issues| Insufficient
        
        Known -->|Heartbeat Timeout| Disconnected
        Disconnected -->|Reconnects| Known
        
        Known -->|Install Triggered| Preparing[PreparingFor<br/>Installation]
        Preparing -->|Disk Ready| Installing
        Installing -->|Writing| InProgress[Installing<br/>InProgress]
        InProgress -->|Success| Installed
        InProgress -->|Failure| Error
        InProgress -->|Boot Order| PendingAction[Pending<br/>UserAction]
        
        Known -->|User Disables| Disabled
        Disabled -->|Re-enabled| Known
    end
    
    style START fill:#2d3748,stroke:#2d3748
    style Discovering fill:#adb5bd,stroke:#6c757d,color:#000
    style Known fill:#adb5bd,stroke:#6c757d,color:#000
    style Insufficient fill:#adb5bd,stroke:#6c757d,color:#000
    style PendingForInput fill:#adb5bd,stroke:#6c757d,color:#000
    style Disconnected fill:#e9c46a,stroke:#c9a227,color:#000
    style Preparing fill:#457b9d,stroke:#1d3557,color:#fff
    style Installing fill:#457b9d,stroke:#1d3557,color:#fff
    style InProgress fill:#457b9d,stroke:#1d3557,color:#fff
    style PendingAction fill:#e9c46a,stroke:#c9a227,color:#000
    style Installed fill:#52796f,stroke:#354f52,color:#fff
    style Error fill:#c9184a,stroke:#a4133c,color:#fff
    style Disabled fill:#adb5bd,stroke:#6c757d,color:#000
    
    style background fill:#beb8a8,stroke:#706858,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
```

## Comparison: Installation Methods

```mermaid
graph TD
    subgraph factors["Decision Factors"]
        D1[Connectivity?]
        D2[Hub Cluster?]
        D3[Cluster Type?]
        D4[Scale?]
    end
    
    subgraph methods["Methods"]
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
    
    style D1 fill:#457b9d,stroke:#1d3557,color:#fff
    style D2 fill:#457b9d,stroke:#1d3557,color:#fff
    style D3 fill:#457b9d,stroke:#1d3557,color:#fff
    style D4 fill:#457b9d,stroke:#1d3557,color:#fff
    style IPI fill:#52796f,stroke:#354f52,color:#fff
    style UPI fill:#52796f,stroke:#354f52,color:#fff
    style SAAS fill:#52796f,stroke:#354f52,color:#fff
    style MCE fill:#52796f,stroke:#354f52,color:#fff
    style ABI fill:#52796f,stroke:#354f52,color:#fff
    style IBI fill:#52796f,stroke:#354f52,color:#fff
    style HCP fill:#52796f,stroke:#354f52,color:#fff
    style ZTP fill:#52796f,stroke:#354f52,color:#fff
    
    style factors fill:#c4bfaa,stroke:#7a6a1a,stroke-width:2px,color:#2d2d2d
    style methods fill:#b8d4d0,stroke:#3d5a52,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
```

## Related Documentation

- [Installation Methods Overview](../01-installation-methods-overview.md)
- [Operators & Controllers Reference](../07-operators-controllers/reference.md)
- [CRD Reference](../08-crd-reference/index.md)

