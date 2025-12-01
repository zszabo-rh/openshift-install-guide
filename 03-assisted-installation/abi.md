# Agent-Based Installer (ABI)

The Agent-Based Installer is a standalone installation method that embeds the Assisted Installer into a bootable ISO, enabling fully disconnected OpenShift deployments without requiring an external service.

**Repositories:**
- [openshift/installer](https://github.com/openshift/installer) - The `openshift-install agent` command
- [openshift/assisted-service](https://github.com/openshift/assisted-service) - Embedded coordination service
- [openshift/assisted-installer-agent](https://github.com/openshift/assisted-installer-agent) - Discovery agent

## Overview

```mermaid
graph TB
    subgraph build["Build Phase"]
        IC[install-config.yaml]
        AC[agent-config.yaml]
        CMD[openshift-install agent create image]
        ISO[Agent ISO]
        
        IC --> CMD
        AC --> CMD
        CMD --> ISO
    end
    
    subgraph boot["Boot Phase"]
        ISO -->|Boot| HOST[Target Host]
        HOST --> AGENT[Discovery Agent]
        HOST --> SERVICE[Embedded Coordination Service]
        HOST --> REGISTER[Agent Registers]
    end
    
    subgraph install["Install Phase"]
        REGISTER --> VALIDATE[Validations]
        VALIDATE --> INSTALL[Installation]
        INSTALL --> CLUSTER[Running Cluster]
    end
    
    style IC fill:#355070,stroke:#1d3557,color:#fff
    style AC fill:#355070,stroke:#1d3557,color:#fff
    style CMD fill:#6d597a,stroke:#4a3f50,color:#fff
    style ISO fill:#52796f,stroke:#354f52,color:#fff
    style HOST fill:#52796f,stroke:#354f52,color:#fff
    style AGENT fill:#6d597a,stroke:#4a3f50,color:#fff
    style SERVICE fill:#6d597a,stroke:#4a3f50,color:#fff
    style REGISTER fill:#457b9d,stroke:#1d3557,color:#fff
    style VALIDATE fill:#457b9d,stroke:#1d3557,color:#fff
    style INSTALL fill:#457b9d,stroke:#1d3557,color:#fff
    style CLUSTER fill:#52796f,stroke:#354f52,color:#fff
    
    style build fill:#c4bfaa,stroke:#7a6a1a,stroke-width:2px,color:#2d2d2d
    style boot fill:#b8d4d0,stroke:#3d5a52,stroke-width:2px,color:#2d2d2d
    style install fill:#cfc5b5,stroke:#8d7a5a,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
```

## When to Use ABI

| Scenario | Use ABI |
|----------|---------|
| Fully disconnected/air-gapped environment | Yes |
| No hub cluster available | Yes |
| Single cluster deployment | Yes |
| Automation pipelines | Yes |
| Pre-defined static configuration | Yes |
| Interactive installation | No (use Assisted SaaS) |
| Multi-cluster management | No (use MCE) |
| GitOps-driven deployment | Partial (consider ZTP) |

## Key Differences from Other Methods

| Aspect | IPI/UPI | Assisted SaaS | Assisted MCE | ABI |
|--------|---------|---------------|--------------|-----|
| External service | No | Yes (Red Hat) | Yes (Hub) | No (Embedded) |
| Disconnected | Partial | No | Yes | Yes |
| Pre-validation | No | Yes | Yes | Yes |
| Bootstrap node | Separate | None | None | None |
| Configuration | install-config | Web UI/API | CRDs | YAML files |

## Configuration Files

### install-config.yaml

Standard OpenShift install configuration:

```yaml
apiVersion: v1
metadata:
  name: my-cluster
baseDomain: example.com
compute:
  - name: worker
    replicas: 2
controlPlane:
  name: master
  replicas: 3
networking:
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  serviceNetwork:
    - 172.30.0.0/16
  networkType: OVNKubernetes
platform:
  baremetal:
    apiVIPs:
      - 192.168.1.100
    ingressVIPs:
      - 192.168.1.101
pullSecret: '<pull_secret>'
sshKey: '<ssh_public_key>'
```

### agent-config.yaml

Agent-specific configuration:

```yaml
apiVersion: v1alpha1
kind: AgentConfig
metadata:
  name: my-cluster
rendezvousIP: 192.168.1.10
additionalNTPSources:
  - ntp.example.com
hosts:
  - hostname: master-0
    role: master
    interfaces:
      - name: eno1
        macAddress: 00:11:22:33:44:01
    rootDeviceHints:
      deviceName: /dev/sda
    networkConfig:
      interfaces:
        - name: eno1
          type: ethernet
          state: up
          ipv4:
            address:
              - ip: 192.168.1.10
                prefix-length: 24
            enabled: true
            dhcp: false
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.1.1
            next-hop-interface: eno1
      dns-resolver:
        config:
          server:
            - 192.168.1.1
  - hostname: master-1
    role: master
    interfaces:
      - name: eno1
        macAddress: 00:11:22:33:44:02
    networkConfig:
      # Similar static config...
  - hostname: master-2
    role: master
    interfaces:
      - name: eno1
        macAddress: 00:11:22:33:44:03
    networkConfig:
      # Similar static config...
```

## Installation Process

### Step 1: Create Configuration

```bash
mkdir cluster-config
cd cluster-config

# Create install-config.yaml and agent-config.yaml
# (as shown above)
```

### Step 2: Generate ISO

```bash
openshift-install agent create image --dir=.

# Output:
# agent.x86_64.iso    - Bootable ISO
# auth/               - kubeconfig and kubeadmin password
```

### Step 3: Boot Nodes

Boot all nodes from the generated ISO. The rendezvousIP host becomes the coordination point:

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'lineColor': '#2d3748', 'actorLineColor': '#2d3748' }}}%%
sequenceDiagram
    box rgb(207,197,181) Rendezvous Host
        participant M0 as Master 0 (rendezvous)
    end
    
    box rgb(184,212,208) Other Nodes
        participant M1 as Master 1
        participant M2 as Master 2
        participant W as Workers
    end
    
    Note over M0: Boot from ISO
    M0->>M0: Start embedded coordination service
    M0->>M0: Start agent
    
    Note over M1,W: Boot from ISO
    M1->>M0: Register with service
    M2->>M0: Register with service
    W->>M0: Register with service
    
    M0->>M0: Validate all hosts
    M0->>M0: Start installation
    
    Note over M0,W: Installation proceeds
```

### Step 4: Monitor Progress

```bash
# Watch installation progress
openshift-install agent wait-for bootstrap-complete --dir=.

# Wait for full installation
openshift-install agent wait-for install-complete --dir=.
```

## Architecture

### Embedded Components

The agent ISO contains embedded minimal services for coordination (internally using the same assisted-service codebase):

```mermaid
graph TB
    subgraph contents["Agent ISO Contents"]
        RHCOS[RHCOS Live Image]
        AGENT[assisted-installer-agent]
        SERVICE[Embedded Coordination Service]
        STATE[Cluster State Storage]
        IMAGES[Container Images]
        CONFIG[ZTP Manifests]
    end
    
    RHCOS --> BOOT[Boot System]
    BOOT --> AGENT
    BOOT --> SERVICE
    SERVICE --> STATE
    AGENT --> SERVICE
    CONFIG --> SERVICE
    
    style RHCOS fill:#52796f,stroke:#354f52,color:#fff
    style AGENT fill:#6d597a,stroke:#4a3f50,color:#fff
    style SERVICE fill:#6d597a,stroke:#4a3f50,color:#fff
    style STATE fill:#7d8597,stroke:#5c6378,color:#fff
    style IMAGES fill:#355070,stroke:#1d3557,color:#fff
    style CONFIG fill:#355070,stroke:#1d3557,color:#fff
    style BOOT fill:#52796f,stroke:#354f52,color:#fff
    
    style contents fill:#c4bfaa,stroke:#7a6a1a,stroke-width:2px,color:#2d2d2d
    
    linkStyle default stroke:#2d3748,stroke-width:2px
```

> **Implementation Note:** The embedded service is based on assisted-service running in a lightweight mode with local state storage. This is an implementation detail; the user experience is through the generated ISO and `openshift-install agent` commands.

### Rendezvous Host

One host (specified by `rendezvousIP`) acts as the coordination point:

| Responsibility | Description |
|----------------|-------------|
| Run coordination service | Embedded REST API for inter-node communication |
| Store cluster state | Local state storage |
| Coordinate installation | Track all hosts and progress |
| Serve Ignition | Machine Config Server role |

### Agent Registration

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'lineColor': '#2d3748', 'actorLineColor': '#2d3748' }}}%%
sequenceDiagram
    box rgb(184,212,208) Agent Process
        participant Agent
    end
    
    box rgb(207,197,181) Coordination
        participant Service as Embedded Service
        participant Cluster as Cluster State
    end
    
    Agent->>Agent: Read MAC address
    Agent->>Agent: Match to host config
    Agent->>Service: POST /v2/infra-envs/{id}/hosts
    Service->>Cluster: Register host
    
    rect rgb(180,175,160)
        loop Every 60 seconds
            Agent->>Service: Hardware inventory
            Agent->>Service: Connectivity results
            Service->>Cluster: Update host status
        end
    end
```

## ZTP Manifests

ABI uses the same manifest format as ZTP for cluster configuration:

```bash
# Directory structure after image creation
cluster-config/
├── agent.x86_64.iso
├── auth/
│   ├── kubeconfig
│   └── kubeadmin-password
└── cluster-manifests/          # Auto-generated
    ├── agent-cluster-install.yaml
    ├── cluster-deployment.yaml
    ├── cluster-image-set.yaml
    ├── infraenv.yaml
    ├── nmstate-config.yaml
    └── pull-secret.yaml
```

These manifests are embedded in the ISO and loaded by the embedded assisted-service.

## Static Network Configuration

### Using agent-config.yaml

```yaml
hosts:
  - hostname: master-0
    networkConfig:
      interfaces:
        - name: bond0
          type: bond
          state: up
          link-aggregation:
            mode: active-backup
            port:
              - eno1
              - eno2
          ipv4:
            address:
              - ip: 192.168.1.10
                prefix-length: 24
            enabled: true
        - name: eno1
          type: ethernet
          state: up
        - name: eno2
          type: ethernet
          state: up
```

### VLAN Configuration

```yaml
hosts:
  - hostname: master-0
    networkConfig:
      interfaces:
        - name: eno1.100
          type: vlan
          state: up
          vlan:
            base-iface: eno1
            id: 100
          ipv4:
            address:
              - ip: 192.168.100.10
                prefix-length: 24
            enabled: true
```

## Disconnected Installation

### Prerequisites

1. Mirror registry with OpenShift images
2. RHCOS ISO accessible locally
3. Network connectivity between nodes

### Configuration

```yaml
# install-config.yaml additions for disconnected
imageDigestSources:
  - source: quay.io/openshift-release-dev/ocp-release
    mirrors:
      - registry.example.com/ocp4/openshift4
  - source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
    mirrors:
      - registry.example.com/ocp4/openshift4
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  <mirror registry CA cert>
  -----END CERTIFICATE-----
```

```bash
# Use local RHCOS image
export OPENSHIFT_INSTALL_OS_IMAGE_OVERRIDE="file:///path/to/rhcos-live.iso"
openshift-install agent create image --dir=.
```

## Single Node OpenShift (SNO)

ABI fully supports SNO deployments:

```yaml
# install-config.yaml for SNO
compute:
  - name: worker
    replicas: 0
controlPlane:
  name: master
  replicas: 1
platform:
  none: {}  # No VIPs needed for SNO
```

```yaml
# agent-config.yaml for SNO
apiVersion: v1alpha1
kind: AgentConfig
metadata:
  name: sno-cluster
rendezvousIP: 192.168.1.10
hosts:
  - hostname: sno-node
    role: master
    interfaces:
      - name: eno1
        macAddress: 00:11:22:33:44:55
```

## Troubleshooting

### Accessing the Rendezvous Host

```bash
# SSH to rendezvous host
ssh core@<rendezvous_ip>

# View assisted-service logs
journalctl -u agent.service -f

# Check installation progress
curl -s http://localhost:8090/api/assisted-install/v2/clusters | jq
```

### Common Issues

| Issue | Diagnosis | Solution |
|-------|-----------|----------|
| Hosts not registering | Check network connectivity | Verify `rendezvousIP` is reachable |
| Validation failures | Check agent logs | Review hardware requirements |
| Installation stuck | Check bootkube logs | Review network/DNS configuration |
| Wrong host roles | MAC address mismatch | Verify MAC addresses in agent-config |

### Gathering Logs

```bash
# Before installation completes
openshift-install agent gather --dir=.

# Creates support bundle with:
# - Agent logs from all hosts
# - assisted-service logs
# - Cluster state
```

## Comparison with Appliance

| Aspect | ABI | Appliance |
|--------|-----|-----------|
| Image type | ISO (boot + discover) | Disk image (complete) |
| Container images | Downloaded during install | Pre-embedded |
| Registry needed | Yes (mirror for disconnected) | No (embedded) |
| Customization | At ISO generation | At appliance build |
| Image size | ~1 GB | ~20+ GB |
| Use case | Normal disconnected | Fully air-gapped |

## Related Documentation

### Detailed Documentation

- [Agent-Based Installer](https://github.com/openshift/assisted-service/blob/master/docs/agent-based-installer.md) - Technical details
- [Static Network Configuration](https://github.com/openshift/assisted-service/blob/master/docs/user-guide/network-configuration/static-configuration.md)
- [OpenShift Installer ABI Docs](https://github.com/openshift/installer/tree/master/docs/user/agent) - Official installer documentation

### This Guide

- [Assisted Installation Overview](overview.md)
- [Image-Based Installation](../04-image-based-installation/ibi.md)
- [Appliance](../04-image-based-installation/appliance.md)
- [ZTP with SiteConfig](../06-gitops-provisioning/ztp.md)

