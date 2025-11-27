# Supporting CRDs Reference

This document covers CRDs that support the installation process but are not the primary cluster definition resources.

## AgentServiceConfig

**API Group:** `agent-install.openshift.io/v1beta1`  
**Owner:** Assisted Service

### Purpose

Configures the deployment of assisted-service on the hub cluster.

```yaml
apiVersion: agent-install.openshift.io/v1beta1
kind: AgentServiceConfig
metadata:
  name: agent
spec:
  # Storage for database
  databaseStorage:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 10Gi
  
  # Storage for files (ignition, logs)
  filesystemStorage:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 100Gi
  
  # Storage for images
  imageStorage:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 50Gi
  
  # RHCOS images to serve
  osImages:
    - cpuArchitecture: x86_64
      openshiftVersion: "4.14"
      url: "https://mirror.example.com/rhcos-live.iso"
      version: "414.92.202310210434-0"
  
  # Mirror registry (disconnected)
  mirrorRegistryRef:
    name: mirror-config
  
  # Unauthenticated registries
  unauthenticatedRegistries:
    - registry.example.com
```

---

## NMStateConfig

**API Group:** `agent-install.openshift.io/v1beta1`  
**Owner:** Assisted Service

### Purpose

Defines static network configuration for hosts in an InfraEnv.

```yaml
apiVersion: agent-install.openshift.io/v1beta1
kind: NMStateConfig
metadata:
  name: master-0-network
  namespace: my-cluster
  labels:
    # Must match InfraEnv's nmStateConfigLabelSelector
    cluster-name: my-cluster
spec:
  # NMState configuration
  config:
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
      - name: bond0
        type: bond
        state: up
        link-aggregation:
          mode: active-backup
          port:
            - eno2
            - eno3
    routes:
      config:
        - destination: 0.0.0.0/0
          next-hop-address: 192.168.1.1
          next-hop-interface: eno1
    dns-resolver:
      config:
        server:
          - 192.168.1.1
  
  # MAC address mapping
  interfaces:
    - name: eno1
      macAddress: "00:11:22:33:44:55"
    - name: eno2
      macAddress: "00:11:22:33:44:56"
    - name: eno3
      macAddress: "00:11:22:33:44:57"
```

---

## ClusterImageSet

**API Group:** `hive.openshift.io/v1`  
**Owner:** Hive Operator

### Purpose

Defines available OpenShift versions for installation.

```yaml
apiVersion: hive.openshift.io/v1
kind: ClusterImageSet
metadata:
  name: openshift-v4.14.0
spec:
  releaseImage: quay.io/openshift-release-dev/ocp-release:4.14.0-x86_64
```

For IBI seed images:
```yaml
apiVersion: hive.openshift.io/v1
kind: ClusterImageSet
metadata:
  name: openshift-4.14-seed
spec:
  releaseImage: registry.example.com/seeds/sno-seed:4.14.0
```

---

## AgentClassification

**API Group:** `agent-install.openshift.io/v1beta1`  
**Owner:** Assisted Service

### Purpose

Automatically labels Agents based on inventory criteria.

```yaml
apiVersion: agent-install.openshift.io/v1beta1
kind: AgentClassification
metadata:
  name: high-memory-hosts
  namespace: my-cluster
spec:
  # Label to apply
  labelKey: memory-class
  labelValue: high
  
  # JSONPath query on Agent inventory
  query: ".memory.physicalBytes > 68719476736"  # > 64 GB
---
apiVersion: agent-install.openshift.io/v1beta1
kind: AgentClassification
metadata:
  name: gpu-hosts
spec:
  labelKey: has-gpu
  labelValue: "true"
  query: '.gpus | length > 0'
```

---

## BareMetalHost

**API Group:** `metal3.io/v1alpha1`  
**Owner:** Baremetal Operator  
**Watched By:** BMACReconciler, ImageClusterInstallReconciler

### Purpose

Represents a physical or virtual bare metal host managed by the Baremetal Operator.

```yaml
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  name: master-0
  namespace: my-cluster
  labels:
    # Links to InfraEnv
    infraenvs.agent-install.openshift.io: my-cluster
  annotations:
    # Assisted agent configuration
    bmac.agent-install.openshift.io/role: master
    bmac.agent-install.openshift.io/hostname: master-0
    # Optional overrides
    bmac.agent-install.openshift.io/installer-args: '["--copy-network"]'
spec:
  # Power on the host
  online: true
  
  # Boot MAC address for matching
  bootMACAddress: "00:11:22:33:44:55"
  
  # BMC configuration
  bmc:
    address: redfish-virtualmedia://192.168.1.10/redfish/v1/Systems/1
    credentialsName: bmc-credentials
    disableCertificateVerification: true
  
  # Disable inspection (Assisted handles discovery)
  automatedCleaningMode: disabled
  
  # Installation disk
  rootDeviceHints:
    deviceName: /dev/sda

status:
  # Provisioning state
  provisioning:
    state: provisioned
  
  # Power state
  poweredOn: true
  
  # Hardware details (populated by agent/inspection)
  hardware:
    cpu:
      count: 32
      arch: x86_64
    ramMebibytes: 131072
```

### BMC Address Formats

| Protocol | Format |
|----------|--------|
| IPMI | `ipmi://192.168.1.10` |
| Redfish | `redfish://192.168.1.10/redfish/v1/Systems/1` |
| Redfish Virtual Media | `redfish-virtualmedia://...` |
| iDRAC | `idrac-virtualmedia://...` |
| iLO | `ilo5-virtualmedia://...` |

---

## PreprovisioningImage

**API Group:** `metal3.io/v1alpha1`  
**Owner:** Baremetal Operator

### Purpose

Represents a boot image for the converged BMO/Assisted flow.

```yaml
apiVersion: metal3.io/v1alpha1
kind: PreprovisioningImage
metadata:
  name: my-cluster
  namespace: my-cluster
spec:
  # Accept formats
  acceptFormats:
    - iso
    - initrd
  
  # Architecture
  architecture: x86_64

status:
  # Generated image URL
  imageUrl: "https://assisted-image-service/..."
  
  # Kernel/initrd for network boot
  kernelUrl: "https://..."
  extraKernelParams: "..."
  
  conditions:
    - type: Ready
      status: "True"
```

---

## ClusterPool

**API Group:** `hive.openshift.io/v1`  
**Owner:** Hive Operator

### Purpose

Maintains a pool of pre-provisioned clusters for on-demand claiming.

```yaml
apiVersion: hive.openshift.io/v1
kind: ClusterPool
metadata:
  name: development-pool
  namespace: cluster-pools
spec:
  # Pool sizing
  size: 3              # Maintain this many unclaimed clusters
  runningCount: 1      # Keep this many running (rest hibernated)
  maxSize: 10          # Maximum total clusters
  maxConcurrent: 2     # Max concurrent provisions
  
  # Cluster template
  baseDomain: dev.example.com
  imageSetRef:
    name: openshift-4.14
  platform:
    aws:
      region: us-east-1
      credentialsSecretRef:
        name: aws-creds
  
  pullSecretRef:
    name: pull-secret
  
  # Lifecycle
  hibernateAfter: 30m
  
  # Labels for claims
  clusterLabels:
    environment: development
```

---

## ClusterClaim

**API Group:** `hive.openshift.io/v1`  
**Owner:** Hive Operator

### Purpose

Claims a cluster from a ClusterPool.

```yaml
apiVersion: hive.openshift.io/v1
kind: ClusterClaim
metadata:
  name: my-dev-cluster
  namespace: cluster-pools
spec:
  # Pool to claim from
  clusterPoolName: development-pool
  
  # Optional: auto-release after duration
  lifetime: 8h
  
  # Optional: specific cluster (usually omitted)
  # namespace: specific-cluster-namespace

status:
  conditions:
    - type: ClusterRunning
      status: "True"
    - type: Pending
      status: "False"
  
  # Claimed cluster info
  clusterName: development-pool-abc123
  clusterNamespace: development-pool-abc123
```

---

## MachinePool

**API Group:** `hive.openshift.io/v1`  
**Owner:** Hive Operator

### Purpose

Manages a group of worker nodes for an installed cluster.

```yaml
apiVersion: hive.openshift.io/v1
kind: MachinePool
metadata:
  name: my-cluster-workers
  namespace: my-cluster
spec:
  # Reference to cluster
  clusterDeploymentRef:
    name: my-cluster
  
  # Pool name (appears in MachineSet)
  name: workers
  
  # Replicas
  replicas: 3
  
  # Platform-specific config
  platform:
    aws:
      type: m5.large
      rootVolume:
        size: 120
        type: gp3
        iops: 3000
      zones:
        - us-east-1a
        - us-east-1b
        - us-east-1c
  
  # Autoscaling
  autoscaling:
    minReplicas: 2
    maxReplicas: 10
  
  # Labels for nodes
  labels:
    node-role.kubernetes.io/app: ""
  
  # Taints
  taints:
    - key: dedicated
      value: app
      effect: NoSchedule

status:
  replicas: 3
  machineSets:
    - name: my-cluster-workers-us-east-1a
      replicas: 1
```

---

## SyncSet / SelectorSyncSet

**API Group:** `hive.openshift.io/v1`  
**Owner:** Hive Operator

### Purpose

Pushes resources and patches to managed clusters.

```yaml
# SyncSet - specific clusters
apiVersion: hive.openshift.io/v1
kind: SyncSet
metadata:
  name: common-config
  namespace: my-cluster
spec:
  clusterDeploymentRefs:
    - name: my-cluster
  
  # Resources to create/update
  resources:
    - apiVersion: v1
      kind: Namespace
      metadata:
        name: monitoring
  
  # Patches to apply
  patches:
    - apiVersion: v1
      kind: ConfigMap
      name: cluster-config
      namespace: kube-system
      patch: |
        data:
          custom-key: custom-value
      patchType: merge
  
  # Secrets to sync
  secretMappings:
    - sourceRef:
        name: image-pull-secret
        namespace: my-cluster
      targetRef:
        name: image-pull-secret
        namespace: default
---
# SelectorSyncSet - by label
apiVersion: hive.openshift.io/v1
kind: SelectorSyncSet
metadata:
  name: production-config
spec:
  clusterDeploymentSelector:
    matchLabels:
      environment: production
  
  resources:
    - apiVersion: v1
      kind: ConfigMap
      metadata:
        name: prod-config
        namespace: default
      data:
        environment: production
```

## Related Documentation

- [Installation CRDs](installation-crds.md)
- [Day 2 Machine Management](day2-machine-management.md)
- [Operators & Controllers Reference](../07-operators-controllers/reference.md)

