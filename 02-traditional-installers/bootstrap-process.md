# The Bootstrap Process

The bootstrap process is fundamental to all OpenShift installations. This document explains how a cluster forms from nothing to a fully operational state.

**Key source code:**
- [openshift/installer/pkg/asset/ignition/bootstrap](https://github.com/openshift/installer/tree/master/pkg/asset/ignition/bootstrap) - Bootstrap Ignition generation
- [openshift/machine-config-operator](https://github.com/openshift/machine-config-operator) - Machine Config Server and node configuration
- [openshift/cluster-bootstrap](https://github.com/openshift/cluster-bootstrap) - Bootstrap rendering logic

## The Bootstrap Problem

OpenShift has a unique challenge: every node needs configuration from the cluster it's joining, but the cluster doesn't exist yet. This creates a chicken-and-egg problem.

**Solution:** A temporary bootstrap node acts as a stand-in cluster until the real control plane is ready.

## Bootstrap Architecture

```mermaid
graph TB
    subgraph phase1["Phase 1: Bootstrap Active"]
        subgraph bootstrapnode["Bootstrap Node"]
            MCS[Machine Config Server<br/>:22623]
            ETCD_B[etcd bootstrap member]
            API_B[kube-apiserver<br/>bootstrap]
            BOOTKUBE[bootkube.service]
            RELEASE[release-image-pivot]
        end
        
        subgraph cpnodes["Control Plane Nodes"]
            KUBELET1[kubelet]
            KUBELET2[kubelet]
            KUBELET3[kubelet]
        end
        
        MCS -->|Ignition| KUBELET1
        MCS -->|Ignition| KUBELET2
        MCS -->|Ignition| KUBELET3
    end
    
    subgraph phase2["Phase 2: Handoff"]
        ETCD_B -->|Data migration| ETCD[etcd cluster]
        API_B -->|Handoff| API[Production API]
    end
    
    style MCS fill:#6d597a,stroke:#4a3f50,color:#fff
    style ETCD_B fill:#7d8597,stroke:#5c6378,color:#fff
    style API_B fill:#6d597a,stroke:#4a3f50,color:#fff
    style BOOTKUBE fill:#6d597a,stroke:#4a3f50,color:#fff
    style RELEASE fill:#6d597a,stroke:#4a3f50,color:#fff
    style KUBELET1 fill:#52796f,stroke:#354f52,color:#fff
    style KUBELET2 fill:#52796f,stroke:#354f52,color:#fff
    style KUBELET3 fill:#52796f,stroke:#354f52,color:#fff
    style ETCD fill:#7d8597,stroke:#5c6378,color:#fff
    style API fill:#6d597a,stroke:#4a3f50,color:#fff
    
    style phase1 fill:#beb8a8,stroke:#706858,stroke-width:2px,color:#2d2d2d
    style bootstrapnode fill:#cfc5b5,stroke:#8d7a5a,stroke-width:2px,color:#2d2d2d
    style cpnodes fill:#b8d4d0,stroke:#3d5a52,stroke-width:2px,color:#2d2d2d
    style phase2 fill:#c4bfaa,stroke:#7a6a1a,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
```

## Detailed Timeline

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'lineColor': '#2d3748', 'actorLineColor': '#2d3748' }}}%%
sequenceDiagram
    box rgb(207,197,181) Bootstrap
        participant B as Bootstrap
    end
    
    box rgb(184,212,208) Control Plane
        participant M1 as Master 1
        participant M2 as Master 2
        participant M3 as Master 3
    end
    
    Note over B: Boot with bootstrap.ign
    B->>B: Start temporary etcd
    B->>B: Start temporary API server
    B->>B: Start Machine Config Server
    
    Note over M1,M3: Boot with master.ign
    M1->>B: Fetch config from MCS :22623
    M2->>B: Fetch config from MCS :22623
    M3->>B: Fetch config from MCS :22623
    
    M1->>M1: Apply MachineConfig, start kubelet
    M2->>M2: Apply MachineConfig, start kubelet
    M3->>M3: Apply MachineConfig, start kubelet
    
    Note over B,M3: Form etcd cluster
    M1->>B: etcd joins bootstrap
    M2->>B: etcd joins cluster
    M3->>B: etcd joins cluster
    
    Note over B: Remove bootstrap etcd member
    B->>B: bootkube: wait for 2 control plane nodes
    
    Note over M1,M3: Production control plane starts
    M1->>M1: Start kube-apiserver
    M2->>M2: Start controller-manager
    M3->>M3: Start scheduler
    
    Note over B: Bootstrap complete
    B->>B: bootkube.service exits
    
    Note over B: Can be destroyed
```

## Bootstrap Node Components

### Core Services

| Service | Purpose | Port |
|---------|---------|------|
| `bootkube.service` | Orchestrates bootstrap | - |
| `machine-config-server` | Serves Ignition configs | 22623 |
| `etcd` (bootstrap) | Temporary etcd member | 2379, 2380 |
| `kube-apiserver` | Temporary API server | 6443 |
| `approve-csr.service` | Auto-approves initial CSRs | - |

### bootkube.service

The main orchestrator that:

1. Waits for etcd to be healthy
2. Starts the bootstrap control plane
3. Renders cluster manifests
4. Waits for the production control plane
5. Removes the bootstrap etcd member
6. Exits successfully (signaling bootstrap complete)

```bash
# View bootkube progress on bootstrap node
journalctl -b -f -u bootkube.service

# Key milestones to look for:
# - "Starting etcd-bootstrap..."
# - "Starting kube-apiserver-bootstrap..."
# - "Tearing down etcd-bootstrap..."
# - "bootkube.service complete"
```

## Machine Config Server (MCS)

The MCS runs on the bootstrap node and serves full Ignition configs to nodes.

### How It Works

1. `master.ign` contains only a pointer:
```json
{
  "ignition": {
    "config": {
      "merge": [{
        "source": "https://api-int.cluster.example.com:22623/config/master"
      }]
    }
  }
}
```

2. Node boots and fetches from MCS
3. MCS serves rendered MachineConfig as Ignition
4. Node applies full configuration

### MCS Endpoints

| Endpoint | Purpose |
|----------|---------|
| `/config/master` | Full control plane Ignition |
| `/config/worker` | Full worker Ignition |
| `/healthz` | Health check |

## etcd Bootstrap

### Initial State

```mermaid
graph LR
    subgraph step1["Step 1"]
        E_B1[etcd-bootstrap<br/>single member]
    end
    
    subgraph step2["Step 2"]
        E_B2[etcd-bootstrap]
        E_M1[etcd-0]
        E_B2 --- E_M1
    end
    
    subgraph step3["Step 3"]
        E_B3[etcd-bootstrap]
        E_M2[etcd-0]
        E_M3[etcd-1]
        E_B3 --- E_M2
        E_M2 --- E_M3
    end
    
    subgraph step4["Step 4 (Final)"]
        E_M4[etcd-0]
        E_M5[etcd-1]
        E_M6[etcd-2]
        E_M4 --- E_M5
        E_M5 --- E_M6
        E_M6 --- E_M4
    end
    
    style E_B1 fill:#adb5bd,stroke:#6c757d,color:#000
    style E_B2 fill:#adb5bd,stroke:#6c757d,color:#000
    style E_B3 fill:#adb5bd,stroke:#6c757d,color:#000
    style E_M1 fill:#52796f,stroke:#354f52,color:#fff
    style E_M2 fill:#52796f,stroke:#354f52,color:#fff
    style E_M3 fill:#52796f,stroke:#354f52,color:#fff
    style E_M4 fill:#52796f,stroke:#354f52,color:#fff
    style E_M5 fill:#52796f,stroke:#354f52,color:#fff
    style E_M6 fill:#52796f,stroke:#354f52,color:#fff
    
    style step1 fill:#cfc5b5,stroke:#8d7a5a,stroke-width:2px,color:#2d2d2d
    style step2 fill:#c4bfaa,stroke:#7a6a1a,stroke-width:2px,color:#2d2d2d
    style step3 fill:#b8d4d0,stroke:#3d5a52,stroke-width:2px,color:#2d2d2d
    style step4 fill:#a8b0b8,stroke:#2d4a42,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
```

### etcd Member Management

```bash
# On bootstrap, check etcd members
etcdctl member list

# Expected progression:
# 1. Only bootstrap member
# 2. bootstrap + master-0
# 3. bootstrap + master-0 + master-1
# 4. bootstrap + master-0 + master-1 + master-2
# 5. master-0 + master-1 + master-2 (bootstrap removed)
```

## Control Plane Components

During bootstrap, these components transition from bootstrap to production:

| Component | Bootstrap Phase | Production Phase |
|-----------|----------------|------------------|
| etcd | Static pod on bootstrap | StatefulSet on masters |
| kube-apiserver | Static pod on bootstrap | Static pod on masters (via MCO) |
| kube-controller-manager | Not running | Static pod on masters |
| kube-scheduler | Not running | Static pod on masters |
| Machine Config Server | Bootstrap binary | DaemonSet on masters (via MCO) |

## Cluster Operators During Bootstrap

The Cluster Version Operator (CVO) starts on the bootstrap node and begins deploying operators:

```mermaid
graph TD
    subgraph background[" "]
        CVO[Cluster Version Operator]
        
        CVO --> MCO[Machine Config Operator]
        CVO --> KAPI[kube-apiserver]
        CVO --> INGRESS[Ingress Operator]
        CVO --> DNS[DNS Operator]
        CVO --> NET[Network Operator]
        CVO --> AUTH[Authentication Operator]
        CVO --> CONSOLE[Console Operator]
        
        MCO -->|Manages| NODES[Node Configuration]
        INGRESS -->|Creates| ROUTER[Ingress Router]
        NET -->|Configures| CNI[CNI/OVN]
    end
    
    style CVO fill:#b56576,stroke:#8d4e5a,color:#fff
    style MCO fill:#6d597a,stroke:#4a3f50,color:#fff
    style KAPI fill:#6d597a,stroke:#4a3f50,color:#fff
    style INGRESS fill:#6d597a,stroke:#4a3f50,color:#fff
    style DNS fill:#6d597a,stroke:#4a3f50,color:#fff
    style NET fill:#6d597a,stroke:#4a3f50,color:#fff
    style AUTH fill:#6d597a,stroke:#4a3f50,color:#fff
    style CONSOLE fill:#6d597a,stroke:#4a3f50,color:#fff
    style NODES fill:#52796f,stroke:#354f52,color:#fff
    style ROUTER fill:#52796f,stroke:#354f52,color:#fff
    style CNI fill:#52796f,stroke:#354f52,color:#fff
    
    style background fill:#beb8a8,stroke:#706858,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
```

## Bootstrap Completion Criteria

The bootstrap is complete when:

1. etcd cluster has 3 healthy members (bootstrap removed)
2. kube-apiserver is running on control plane nodes
3. CVO has deployed critical operators
4. `bootkube.service` exits successfully

```bash
# Check bootstrap completion
openshift-install wait-for bootstrap-complete --dir=cluster --log-level=debug

# When complete, you'll see:
# INFO It is now safe to remove the bootstrap resources
# INFO Time elapsed: 15m30s
```

## Troubleshooting Bootstrap

### Common Failure Points

```mermaid
flowchart TD
    subgraph background[" "]
        START[Bootstrap Issues] --> Q1{etcd healthy?}
        Q1 -->|No| E1[Check etcd logs<br/>Verify networking]
        Q1 -->|Yes| Q2{API accessible?}
        
        Q2 -->|No| E2[Check MCS<br/>Verify DNS/LB]
        Q2 -->|Yes| Q3{Masters joining?}
        
        Q3 -->|No| E3[Check master Ignition<br/>Verify MCS accessibility]
        Q3 -->|Yes| Q4{bootkube completing?}
        
        Q4 -->|No| E4[Check bootkube.service logs<br/>Look for operator failures]
        Q4 -->|Yes| SUCCESS[Bootstrap Complete]
    end
    
    style START fill:#adb5bd,stroke:#6c757d,color:#000
    style Q1 fill:#457b9d,stroke:#1d3557,color:#fff
    style Q2 fill:#457b9d,stroke:#1d3557,color:#fff
    style Q3 fill:#457b9d,stroke:#1d3557,color:#fff
    style Q4 fill:#457b9d,stroke:#1d3557,color:#fff
    style E1 fill:#e9c46a,stroke:#c9a227,color:#000
    style E2 fill:#e9c46a,stroke:#c9a227,color:#000
    style E3 fill:#e9c46a,stroke:#c9a227,color:#000
    style E4 fill:#e9c46a,stroke:#c9a227,color:#000
    style SUCCESS fill:#52796f,stroke:#354f52,color:#fff
    
    style background fill:#beb8a8,stroke:#706858,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
    linkStyle 1,3,5,7 stroke:#9b2c2c,stroke-width:2px
```

### Gathering Bootstrap Logs

```bash
# SSH to bootstrap node
ssh core@<bootstrap_ip>

# View all journal logs
journalctl -b

# Specific services
journalctl -b -u bootkube.service
journalctl -b -u kubelet.service
journalctl -b -u crio.service

# Container logs (pods)
crictl logs <container_id>

# Or use installer command
openshift-install gather bootstrap --dir=cluster
```

### Key Log Locations

| Log | Location | Purpose |
|-----|----------|---------|
| bootkube | `journalctl -u bootkube.service` | Main orchestrator |
| kubelet | `journalctl -u kubelet.service` | Node agent |
| etcd | `crictl logs etcd-bootstrap` | etcd member |
| API server | `crictl logs kube-apiserver-bootstrap` | API |
| MCS | `crictl logs machine-config-server` | Config server |

### Network Connectivity Requirements

During bootstrap, these connections must work:

```
Bootstrap → Masters (all)
  - TCP 2379-2380 (etcd)
  - TCP 6443 (API)
  
Masters → Bootstrap
  - TCP 22623 (MCS)
  - TCP 2379-2380 (etcd)
  
Masters → Masters
  - TCP 2379-2380 (etcd)
  - TCP 6443 (API)
  
External → Load Balancer → Bootstrap/Masters
  - TCP 6443 (API)
  - TCP 22623 (MCS) - internal only
```

## Bootstrap in Different Installation Methods

### IPI
- Bootstrap VM is automatically provisioned
- Automatically destroyed after completion

### UPI
- User provisions bootstrap infrastructure
- User must destroy after `wait-for bootstrap-complete`

### Assisted Installer
- One node temporarily acts as bootstrap
- No separate bootstrap machine needed
- Uses "bootstrap-in-place" technique

### Agent-Based Installer
- Similar to Assisted Installer
- Bootstrap role determined automatically

## Related Documentation

- [Traditional Installers Overview](index.md) - Section overview
- [IPI Installation](ipi.md) - Automated infrastructure
- [UPI Installation](upi.md) - Manual infrastructure
- [Assisted Installation](../03-assisted-installation/overview.md) - Bootstrap-less approach

