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
    subgraph "assisted-service Operator (Pod)"
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
```

A single operator typically implements multiple controllers, each responsible for reconciling a specific CRD type. See [Detailed Controller Reference](reference.md) for the complete list.

## Operator Ecosystem

```mermaid
graph TB
    subgraph "Cluster Lifecycle Layer"
        MCE[MCE Operator]
        HIVE[Hive Operator]
    end
    
    subgraph "Installation Layer"
        ASSISTED[Assisted Service]
        IBI_OP[IBI Operator]
        HYPERSHIFT[HyperShift Operator]
    end
    
    subgraph "Infrastructure Layer"
        BMO[Baremetal Operator]
        CAPI[Cluster API]
    end
    
    subgraph "Configuration Layer"
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
sequenceDiagram
    participant API as Kubernetes API
    participant Ctrl as Controller
    participant State as External State
    
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
    subgraph "Primary Watch (For)"
        A1[Controller A]
        CRD_A[CRD A]
    end
    
    subgraph "Secondary Watches"
        CRD_B[CRD B]
        CRD_C[CRD C]
        SECRET[Secret]
    end
    
    CRD_A --> A1
    CRD_B -.->|Watches| A1
    CRD_C -.->|Watches| A1
    SECRET -.->|Watches| A1
```

## Cross-Operator Communication

Operators communicate through:

1. **Shared CRDs** - One operator owns, others watch
2. **Status conditions** - Standard reporting mechanism
3. **Annotations** - Controller-specific metadata
4. **Finalizers** - Coordinated deletion

### Example: Assisted ↔ BMAC Flow

```mermaid
sequenceDiagram
    participant BMO as Baremetal Operator
    participant BMAC as BMAC Controller
    participant Assisted as Assisted Service
    
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
    OLM[OLM] --> MCE[MCE Operator]
    MCE --> HIVE[Hive]
    MCE --> ASSISTED[Assisted Service]
    MCE --> BMO[Baremetal Operator]
    MCE --> HYPERSHIFT[HyperShift]
    MCE --> SITECONFIG[SiteConfig]
    
    ASSISTED --> BMAC[BMAC Controller]
    BMO --> BMAC
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
    subgraph "Error Sources"
        VALIDATION[Validation Error]
        INFRA[Infrastructure Error]
        TIMEOUT[Timeout]
    end
    
    subgraph "Reporting"
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

