# Operators and Controllers Overview

OpenShift installation involves multiple operators working together. This document provides an architectural overview of how they interact.

## Terminology: Operators vs Controllers

Before diving in, let's clarify the relationship between operators and controllers:

| Term | Definition | Example |
|------|------------|---------|
| **Operator** | A deployment/pod that runs one or more controllers, often packaged and distributed via OLM (Operator Lifecycle Manager) | assisted-service, hive-operator, hypershift |
| **Controller** | A specific reconciliation loop that watches CRDs and drives actual state toward desired spec | InfraEnvReconciler, AgentReconciler |

An operator is the **container/deployment**, while controllers are the **logic inside it**:

```mermaid
graph TB
    subgraph operator["assisted-service Operator (Pod)"]
        CTRL1[InfraEnvReconciler]
        CTRL2[AgentReconciler]
        CTRL3[ClusterDeploymentsReconciler]
        CTRL4[BMACReconciler]
        API[REST API Handler]
    end
    
    IE[InfraEnv CR] --> CTRL1
    AGENT[Agent CR] --> CTRL2
    CD[ClusterDeployment CR] --> CTRL3
    BMH[BareMetalHost CR] --> CTRL4
    
    style CTRL1 fill:#6d597a,stroke:#4a3f50,color:#fff
    style CTRL2 fill:#6d597a,stroke:#4a3f50,color:#fff
    style CTRL3 fill:#6d597a,stroke:#4a3f50,color:#fff
    style CTRL4 fill:#6d597a,stroke:#4a3f50,color:#fff
    style API fill:#6d597a,stroke:#4a3f50,color:#fff
    style IE fill:#355070,stroke:#1d3557,color:#fff
    style AGENT fill:#355070,stroke:#1d3557,color:#fff
    style CD fill:#355070,stroke:#1d3557,color:#fff
    style BMH fill:#355070,stroke:#1d3557,color:#fff
    
    style operator fill:#c4bfaa,stroke:#7a6a1a,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
```

A single operator typically implements multiple controllers, each responsible for reconciling a specific CRD type. See [Detailed Controller Reference](reference.md) for the complete list.

## Operator Ecosystem

```mermaid
graph TB
    subgraph lifecycle["Cluster Lifecycle Layer"]
        MCE[MCE Operator]
        HIVE[Hive Operator]
    end
    
    subgraph installation["Installation Layer"]
        ASSISTED[Assisted Service]
        IBI_OP[IBI Operator]
        HYPERSHIFT[HyperShift Operator]
    end
    
    subgraph infra["Infrastructure Layer"]
        BMO[Baremetal Operator]
        CAPI[Cluster API]
    end
    
    subgraph config["Configuration Layer"]
        SITECONFIG[SiteConfig Operator]
        MCO[Machine Config Operator]
    end
    
    MCE --> HIVE
    MCE --> ASSISTED
    MCE --> BMO
    MCE --> HYPERSHIFT
    MCE --> SITECONFIG
    
    HIVE --> ASSISTED
    ASSISTED --> BMO
    IBI_OP --> BMO
    HYPERSHIFT --> CAPI
    
    SITECONFIG --> HIVE
    SITECONFIG --> ASSISTED
    SITECONFIG --> IBI_OP
    
    style MCE fill:#6d597a,stroke:#4a3f50,color:#fff
    style HIVE fill:#6d597a,stroke:#4a3f50,color:#fff
    style ASSISTED fill:#6d597a,stroke:#4a3f50,color:#fff
    style IBI_OP fill:#6d597a,stroke:#4a3f50,color:#fff
    style HYPERSHIFT fill:#6d597a,stroke:#4a3f50,color:#fff
    style BMO fill:#6d597a,stroke:#4a3f50,color:#fff
    style CAPI fill:#6d597a,stroke:#4a3f50,color:#fff
    style SITECONFIG fill:#6d597a,stroke:#4a3f50,color:#fff
    style MCO fill:#6d597a,stroke:#4a3f50,color:#fff
    
    style lifecycle fill:#c4bfaa,stroke:#7a6a1a,stroke-width:2px,color:#2d2d2d
    style installation fill:#cfc5b5,stroke:#8d7a5a,stroke-width:2px,color:#2d2d2d
    style infra fill:#b8d4d0,stroke:#3d5a52,stroke-width:2px,color:#2d2d2d
    style config fill:#a8b0b8,stroke:#2d4a42,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
```

## Operators by Function

### Cluster Lifecycle

| Operator | Purpose | Key CRDs |
|----------|---------|----------|
| **MCE Operator** | Central lifecycle management | MultiClusterEngine |
| **Hive Operator** | Cluster provisioning/deprovisioning | ClusterDeployment, ClusterPool |

### Installation Orchestration

| Operator | Purpose | Key CRDs |
|----------|---------|----------|
| **Assisted Service** | Guided installation | AgentClusterInstall, InfraEnv, Agent |
| **IBI Operator** | Image-based installation | ImageClusterInstall |
| **HyperShift Operator** | Hosted control planes | HostedCluster, NodePool |

### Infrastructure Management

| Operator | Purpose | Key CRDs |
|----------|---------|----------|
| **Baremetal Operator** | Bare metal host lifecycle | BareMetalHost, PreprovisioningImage |
| **Cluster API** | Machine management | Machine, MachineSet, MachineDeployment |

### Configuration Management

| Operator | Purpose | Key CRDs |
|----------|---------|----------|
| **SiteConfig Operator** | Template-based provisioning | ClusterInstance |
| **Machine Config Operator** | Node OS configuration | MachineConfig, MachineConfigPool |

## Controller Patterns

### Reconciliation Flow

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'lineColor': '#2d3748', 'actorLineColor': '#2d3748' }}}%%
sequenceDiagram
    box rgb(190,184,168) Kubernetes
        participant API as Kubernetes API
        participant Ctrl as Controller
    end
    
    box rgb(180,175,160) External
        participant State as External State
    end
    
    API->>Ctrl: Watch event (create/update/delete)
    Ctrl->>Ctrl: Read current state
    Ctrl->>State: Check external state
    State-->>Ctrl: Current state
    Ctrl->>Ctrl: Calculate diff
    Ctrl->>State: Apply changes
    Ctrl->>API: Update status
    
    Note over Ctrl: Requeue if needed
```

### Watch Relationships

Controllers watch primary resources and related resources:

```mermaid
graph LR
    subgraph primary["Primary Watch (For)"]
        A1[Controller A]
        CRD_A[CRD A]
    end
    
    subgraph secondary["Secondary Watches"]
        CRD_B[CRD B]
        CRD_C[CRD C]
        SECRET[Secret]
    end
    
    CRD_A --> A1
    CRD_B -.->|Watches| A1
    CRD_C -.->|Watches| A1
    SECRET -.->|Watches| A1
    
    style A1 fill:#6d597a,stroke:#4a3f50,color:#fff
    style CRD_A fill:#355070,stroke:#1d3557,color:#fff
    style CRD_B fill:#355070,stroke:#1d3557,color:#fff
    style CRD_C fill:#355070,stroke:#1d3557,color:#fff
    style SECRET fill:#7d8597,stroke:#5c6378,color:#fff
    
    style primary fill:#c4bfaa,stroke:#7a6a1a,stroke-width:2px,color:#2d2d2d
    style secondary fill:#b8d4d0,stroke:#3d5a52,stroke-width:2px,color:#2d2d2d
    
    linkStyle 0 stroke:#2d3748,stroke-width:2px
    linkStyle 1,2,3 stroke:#2d3748,stroke-width:2px,stroke-dasharray:5
```

## Cross-Operator Communication

Operators communicate through:

1. **Shared CRDs** - One operator owns, others watch
2. **Status conditions** - Standard reporting mechanism
3. **Annotations** - Controller-specific metadata
4. **Finalizers** - Coordinated deletion

### Example: Assisted ↔ BMAC Flow

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'lineColor': '#2d3748', 'actorLineColor': '#2d3748' }}}%%
sequenceDiagram
    box rgb(190,184,168) Operators
        participant BMO as Baremetal Operator
        participant BMAC as BMAC Controller
        participant Assisted as Assisted Service
    end
    
    Note over BMO: BareMetalHost created
    BMO->>BMO: Provision host
    BMAC->>Assisted: Watch for Agents
    
    Note over Assisted: Host boots, Agent registers
    Assisted-->>BMAC: Agent CR created
    
    BMAC->>BMAC: Match BMH ↔ Agent by MAC
    BMAC->>Assisted: Update Agent spec (from BMH annotations)
    BMAC->>BMO: Update BMH (from Agent inventory)
```

## Operator Dependencies

### Installation Order

```mermaid
graph TD
    subgraph background[" "]
        OLM[OLM] --> MCE[MCE Operator]
        MCE --> HIVE[Hive]
        MCE --> ASSISTED[Assisted Service]
        MCE --> BMO[Baremetal Operator]
        MCE --> HYPERSHIFT[HyperShift]
        MCE --> SITECONFIG[SiteConfig]
        
        ASSISTED --> BMAC[BMAC Controller]
        BMO --> BMAC
    end
    
    style OLM fill:#b56576,stroke:#8d4e5a,color:#fff
    style MCE fill:#6d597a,stroke:#4a3f50,color:#fff
    style HIVE fill:#6d597a,stroke:#4a3f50,color:#fff
    style ASSISTED fill:#6d597a,stroke:#4a3f50,color:#fff
    style BMO fill:#6d597a,stroke:#4a3f50,color:#fff
    style HYPERSHIFT fill:#6d597a,stroke:#4a3f50,color:#fff
    style SITECONFIG fill:#6d597a,stroke:#4a3f50,color:#fff
    style BMAC fill:#6d597a,stroke:#4a3f50,color:#fff
    
    style background fill:#beb8a8,stroke:#706858,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
```

### Namespace Layout

| Namespace | Components |
|-----------|------------|
| `multicluster-engine` | MCE, Assisted, Hive controllers |
| `openshift-machine-api` | Machine API, Baremetal Operator |
| `hypershift` | HyperShift Operator |
| `siteconfig-system` | SiteConfig Operator |
| `<cluster-name>` | Per-cluster resources |

## Failure Handling

### Retry Mechanisms

| Pattern | Use Case | Example |
|---------|----------|---------|
| **Requeue** | Temporary failure | Network timeout |
| **Exponential backoff** | Repeated failures | API rate limiting |
| **Finalizer blocking** | Wait for dependency | Clean up related resources |
| **Status condition** | Permanent error | Invalid configuration |

### Error Propagation

```mermaid
graph TB
    subgraph sources["Error Sources"]
        VALIDATION[Validation Error]
        INFRA[Infrastructure Error]
        TIMEOUT[Timeout]
    end
    
    subgraph reporting["Reporting"]
        CONDITION[Status Condition]
        EVENT[Kubernetes Event]
        LOG[Operator Logs]
    end
    
    VALIDATION --> CONDITION
    VALIDATION --> EVENT
    INFRA --> CONDITION
    INFRA --> LOG
    TIMEOUT --> CONDITION
    TIMEOUT --> LOG
    
    style VALIDATION fill:#c9184a,stroke:#a4133c,color:#fff
    style INFRA fill:#c9184a,stroke:#a4133c,color:#fff
    style TIMEOUT fill:#e9c46a,stroke:#c9a227,color:#000
    style CONDITION fill:#457b9d,stroke:#1d3557,color:#fff
    style EVENT fill:#457b9d,stroke:#1d3557,color:#fff
    style LOG fill:#457b9d,stroke:#1d3557,color:#fff
    
    style sources fill:#cfc5b5,stroke:#8d7a5a,stroke-width:2px,color:#2d2d2d
    style reporting fill:#b8d4d0,stroke:#3d5a52,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
```

## Monitoring Operators

### Health Checks

```bash
# Check operator pods
oc get pods -n multicluster-engine
oc get pods -n openshift-machine-api
oc get pods -n hypershift

# Check operator logs
oc logs -n multicluster-engine -l control-plane=assisted-service
oc logs -n multicluster-engine -l app=hive
```

### Common Metrics

| Metric | Description |
|--------|-------------|
| `controller_runtime_reconcile_total` | Total reconciliations |
| `controller_runtime_reconcile_errors_total` | Reconcile errors |
| `controller_runtime_reconcile_time_seconds` | Reconcile duration |
| `workqueue_depth` | Pending work items |

## Related Documentation

- [Detailed Controller Reference](reference.md)
- [CRD Reference](../08-crd-reference/index.md)
- [Component Diagrams](../09-diagrams/component-diagrams.md)

