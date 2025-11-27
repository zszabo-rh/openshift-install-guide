# Installation CRDs Reference

This document details the primary CRDs used to define and install OpenShift clusters.

## ClusterDeployment

**API Group:** `hive.openshift.io/v1`  
**Owner:** Hive Operator  
**Watched By:** ClusterDeploymentsReconciler (assisted-service), Hive controllers

### Purpose

The central resource defining a cluster to be installed. Acts as the anchor for all cluster-related resources.

### Key Fields

```yaml
apiVersion: hive.openshift.io/v1
kind: ClusterDeployment
metadata:
  name: my-cluster
  namespace: my-cluster
spec:
  # Cluster identity
  baseDomain: example.com
  clusterName: my-cluster
  
  # Platform configuration
  platform:
    # For cloud (IPI)
    aws:
      region: us-east-1
      credentialsSecretRef:
        name: aws-creds
    # OR for bare metal (Assisted)
    agentBareMetal:
      agentSelector:
        matchLabels:
          cluster-name: my-cluster
  
  # Installation method reference
  clusterInstallRef:
    group: extensions.hive.openshift.io
    kind: AgentClusterInstall  # or ImageClusterInstall
    name: my-cluster
    version: v1beta1
  
  # For IPI: provisioning config
  provisioning:
    imageSetRef:
      name: openshift-4.14
    installConfigSecretRef:
      name: install-config
  
  # Secrets
  pullSecretRef:
    name: pull-secret
  
  # Power management
  powerState: Running  # Running, Hibernating

status:
  # Installation state
  installed: true
  installedTimestamp: "2024-01-15T10:30:00Z"
  
  # Access info
  webConsoleURL: "https://console-..."
  apiURL: "https://api.my-cluster.example.com:6443"
  
  # Credentials
  clusterMetadata:
    adminKubeconfigSecretRef:
      name: my-cluster-admin-kubeconfig
    adminPasswordSecretRef:
      name: my-cluster-admin-password
  
  conditions:
    - type: ClusterInstallCompleted
      status: "True"
```

---

## AgentClusterInstall

**API Group:** `extensions.hive.openshift.io/v1beta1`  
**Owner:** Assisted Service  
**Watched By:** ClusterDeploymentsReconciler, AgentClusterInstallReconciler

### Purpose

Defines the installation parameters for Assisted Installer clusters. Referenced by ClusterDeployment's `clusterInstallRef`.

### Key Fields

```yaml
apiVersion: extensions.hive.openshift.io/v1beta1
kind: AgentClusterInstall
metadata:
  name: my-cluster
  namespace: my-cluster
spec:
  # Reference back to ClusterDeployment
  clusterDeploymentRef:
    name: my-cluster
  
  # OCP version
  imageSetRef:
    name: openshift-4.14
  
  # Networking
  networking:
    clusterNetwork:
      - cidr: 10.128.0.0/14
        hostPrefix: 23
    serviceNetwork:
      - 172.30.0.0/16
    machineNetwork:
      - cidr: 192.168.1.0/24
    networkType: OVNKubernetes
  
  # VIPs (for multi-node)
  apiVIPs:
    - 192.168.1.100
  ingressVIPs:
    - 192.168.1.101
  
  # Node requirements
  provisionRequirements:
    controlPlaneAgents: 3
    workerAgents: 2
  
  # Platform type
  platformType: BareMetal  # BareMetal, VSphere, None, External
  
  # Additional manifests
  manifestsConfigMapRefs:
    - name: extra-manifests
  
  # SSH key
  sshPublicKey: "ssh-rsa AAAA..."

status:
  conditions:
    - type: SpecSynced
      status: "True"
    - type: Validated
      status: "True"
    - type: RequirementsMet
      status: "True"
    - type: Completed
      status: "True"
    - type: Failed
      status: "False"
  
  debugInfo:
    eventsURL: "https://assisted-service/events/..."
```

---

## InfraEnv

**API Group:** `agent-install.openshift.io/v1beta1`  
**Owner:** Assisted Service  
**Watched By:** InfraEnvReconciler, BMACReconciler, PreprovisioningImageReconciler

### Purpose

Represents a discovery environment and its associated discovery ISO. Hosts boot from this ISO to register as Agents.

### Key Fields

```yaml
apiVersion: agent-install.openshift.io/v1beta1
kind: InfraEnv
metadata:
  name: my-infraenv
  namespace: my-cluster
spec:
  # Cluster reference (optional for late binding)
  clusterRef:
    name: my-cluster
    namespace: my-cluster
  
  # Pull secret
  pullSecretRef:
    name: pull-secret
  
  # SSH key for discovery hosts
  sshAuthorizedKey: "ssh-rsa AAAA..."
  
  # CPU architecture
  cpuArchitecture: x86_64  # x86_64, aarch64
  
  # Proxy settings
  proxy:
    httpProxy: "http://proxy.example.com:8080"
    httpsProxy: "http://proxy.example.com:8080"
    noProxy: ".example.com,10.0.0.0/8"
  
  # Additional certificates
  additionalTrustBundle: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
  
  # Static network config selector
  nmStateConfigLabelSelector:
    matchLabels:
      cluster-name: my-cluster
  
  # Kernel arguments
  kernelArguments:
    - operation: append
      value: "console=ttyS0"
  
  # Ignition overrides
  ignitionConfigOverride: |
    {"ignition":{"version":"3.2.0"},...}

status:
  # ISO download URL
  isoDownloadURL: "https://assisted-image-service/images/..."
  
  # For iPXE boot
  bootArtifacts:
    initrd: "https://..."
    kernel: "https://..."
    rootfs: "https://..."
  
  conditions:
    - type: ImageCreated
      status: "True"
```

---

## Agent

**API Group:** `agent-install.openshift.io/v1beta1`  
**Owner:** Assisted Service  
**Watched By:** AgentReconciler, BMACReconciler, AgentLabelReconciler

### Purpose

Represents a discovered host that has booted from a discovery ISO. Created automatically when a host registers.

### Key Fields

```yaml
apiVersion: agent-install.openshift.io/v1beta1
kind: Agent
metadata:
  name: aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee  # UUID from agent
  namespace: my-cluster
  labels:
    infraenvs.agent-install.openshift.io: my-infraenv
spec:
  # Approval for installation
  approved: true
  
  # Cluster binding (can be late-bound)
  clusterDeploymentName:
    name: my-cluster
    namespace: my-cluster
  
  # Host configuration
  hostname: master-0
  role: master  # master, worker, auto-assign
  
  # Installation disk
  installation_disk_id: /dev/sda
  
  # Ignition endpoint (for custom config)
  ignitionEndpointTokenReference:
    name: ignition-token
    namespace: my-cluster
  
  # Machine config pool (for workers)
  machineConfigPool: worker

status:
  # Hardware inventory
  inventory:
    hostname: localhost.localdomain
    systemVendor:
      manufacturer: Dell Inc.
      productName: PowerEdge R750
    cpu:
      count: 32
      architecture: x86_64
      modelName: "Intel Xeon Gold 6330"
    memory:
      physicalBytes: 137438953472
    disks:
      - name: sda
        sizeBytes: 480103981056
        driveType: SSD
        bootable: true
    interfaces:
      - name: eno1
        macAddress: "00:11:22:33:44:55"
        ipV4Addresses:
          - 192.168.1.10/24
  
  # Validation results
  validationsInfo: |
    {"hardware":[{"id":"has-min-cpu-cores","status":"success"},...]}
  
  conditions:
    - type: SpecSynced
      status: "True"
    - type: Connected
      status: "True"
    - type: RequirementsMet
      status: "True"
    - type: Validated
      status: "True"
    - type: Installed
      status: "False"
      reason: InstallationInProgress
```

---

## ImageClusterInstall

**API Group:** `extensions.hive.openshift.io/v1alpha1`  
**Owner:** IBI Operator  
**Watched By:** ImageClusterInstallReconciler

### Purpose

Defines installation parameters for Image-Based Install (IBI) clusters. Used for rapid SNO deployment from seed images.

### Key Fields

```yaml
apiVersion: extensions.hive.openshift.io/v1alpha1
kind: ImageClusterInstall
metadata:
  name: my-sno
  namespace: my-sno
spec:
  # Reference to ClusterDeployment
  clusterDeploymentRef:
    name: my-sno
  
  # Seed image reference
  imageSetRef:
    name: openshift-4.14-seed
  
  # Target host
  hostname: sno-node.example.com
  
  # BareMetalHost to use
  bareMetalHostRef:
    name: sno-host
    namespace: my-sno
  
  # Network configuration
  machineNetwork: 192.168.1.0/24
  networkConfig:
    interfaces:
      - name: eno1
        type: ethernet
        state: up
        ipv4:
          address:
            - ip: 192.168.1.100
              prefix-length: 24
          enabled: true
  
  # Extra manifests
  extraManifestsRefs:
    - name: day1-config

status:
  conditions:
    - type: ImageCreated
      status: "True"
    - type: HostConfigured
      status: "True"
    - type: Completed
      status: "True"
  
  bootTime: "2024-01-15T10:30:00Z"
  installedTimestamp: "2024-01-15T10:45:00Z"
```

---

## HostedCluster

**API Group:** `hypershift.openshift.io/v1beta1`  
**Owner:** HyperShift Operator

### Purpose

Defines a hosted control plane cluster where the control plane runs as pods on the management cluster.

### Key Fields

```yaml
apiVersion: hypershift.openshift.io/v1beta1
kind: HostedCluster
metadata:
  name: my-hosted
  namespace: clusters
spec:
  # Release version
  release:
    image: quay.io/openshift-release-dev/ocp-release:4.14.0-x86_64
  
  # Pull secret
  pullSecret:
    name: pull-secret
  
  # SSH key
  sshKey:
    name: ssh-key
  
  # Networking
  networking:
    clusterNetwork:
      - cidr: 10.132.0.0/14
    serviceNetwork:
      - cidr: 172.31.0.0/16
    networkType: OVNKubernetes
  
  # Platform
  platform:
    type: AWS  # AWS, Azure, Agent, KubeVirt
    aws:
      region: us-east-1
      endpointAccess: Public
  
  # Control plane configuration
  controllerAvailabilityPolicy: HighlyAvailable
  
  # etcd configuration
  etcd:
    managementType: Managed
    managed:
      storage:
        persistentVolume:
          size: 8Gi
  
  # Service publishing
  services:
    - service: APIServer
      servicePublishingStrategy:
        type: LoadBalancer
    - service: Ignition
      servicePublishingStrategy:
        type: Route

status:
  conditions:
    - type: Available
      status: "True"
    - type: Progressing
      status: "False"
    - type: Degraded
      status: "False"
  
  kubeconfig:
    name: my-hosted-admin-kubeconfig
  
  version:
    desired:
      image: "..."
    history:
      - state: Completed
        version: "4.14.0"
```

---

## NodePool

**API Group:** `hypershift.openshift.io/v1beta1`  
**Owner:** HyperShift Operator

### Purpose

Manages worker nodes for a HostedCluster.

### Key Fields

```yaml
apiVersion: hypershift.openshift.io/v1beta1
kind: NodePool
metadata:
  name: my-hosted-workers
  namespace: clusters
spec:
  # Reference to HostedCluster
  clusterName: my-hosted
  
  # Replicas
  replicas: 3
  
  # Optional: specific release (defaults to HostedCluster)
  release:
    image: quay.io/openshift-release-dev/ocp-release:4.14.0-x86_64
  
  # Platform configuration
  platform:
    type: AWS
    aws:
      instanceType: m5.large
      rootVolume:
        size: 120
        type: gp3
  
  # Autoscaling
  autoScaling:
    min: 2
    max: 10
  
  # Management
  management:
    autoRepair: true
    upgradeType: Replace  # Replace, InPlace

status:
  conditions:
    - type: Ready
      status: "True"
  
  replicas: 3
  version: "4.14.0"
```

---

## ClusterInstance

**API Group:** `siteconfig.open-cluster-management.io/v1alpha1`  
**Owner:** SiteConfig Operator

### Purpose

Unified API for template-driven cluster provisioning. Decouples cluster definition from installation method.

### Key Fields

```yaml
apiVersion: siteconfig.open-cluster-management.io/v1alpha1
kind: ClusterInstance
metadata:
  name: edge-site-1
  namespace: edge-site-1
spec:
  # Cluster identity
  clusterName: edge-site-1
  baseDomain: example.com
  clusterType: SNO  # SNO, Compact, Standard
  
  # Template reference
  templateRefs:
    - name: ai-cluster-templates-v1
      namespace: siteconfig-system
  
  # Image reference
  clusterImageSetNameRef: openshift-v4.14.0
  
  # Secrets
  pullSecretRef:
    name: pull-secret
  sshPublicKey: "ssh-rsa AAAA..."
  
  # Networking
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  serviceNetwork:
    - 172.30.0.0/16
  machineNetwork:
    - cidr: 192.168.1.0/24
  
  # Labels for policies
  clusterLabels:
    cluster-type: sno
    region: us-east
  
  # Node definitions
  nodes:
    - hostname: sno-node
      role: master
      bmcAddress: redfish://192.168.1.10/redfish/v1/Systems/1
      bmcCredentialsName:
        name: bmc-secret
      bootMACAddress: "00:11:22:33:44:55"
      rootDeviceHints:
        deviceName: /dev/sda
      nodeNetwork:
        interfaces:
          - name: eno1
            macAddress: "00:11:22:33:44:55"
        config:
          # NMState config
          interfaces:
            - name: eno1
              type: ethernet
              state: up
              ipv4:
                address:
                  - ip: 192.168.1.100
                    prefix-length: 24
                enabled: true

status:
  conditions:
    - type: ClusterInstanceValidated
      status: "True"
    - type: RenderedTemplates
      status: "True"
    - type: Provisioned
      status: "True"
  
  clusterDeploymentRef:
    name: edge-site-1
  
  manifestsRendered:
    - kind: ClusterDeployment
      name: edge-site-1
    - kind: AgentClusterInstall
      name: edge-site-1
```

## Related Documentation

- [Supporting CRDs](supporting-crds.md)
- [Day 2 Machine Management](day2-machine-management.md)
- [YAML Examples](examples/)

