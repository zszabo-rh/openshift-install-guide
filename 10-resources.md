# Resources for Further Study

A curated collection of documentation, repositories, and learning resources for OpenShift installation.

## Official Documentation

### Red Hat Documentation

| Resource | Description |
|----------|-------------|
| [OpenShift Installation Overview](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/installation_overview/index) | Official installation documentation |
| [Agent-Based Installer](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/installing_an_on-premise_cluster_with_the_agent-based_installer/index) | ABI documentation |
| [Assisted Installer](https://docs.redhat.com/en/documentation/assisted_installer_for_openshift_container_platform/) | Standalone Assisted Installer docs |
| [Hosted Control Planes](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/hosted_control_planes/index) | HCP documentation |
| [ACM Clusters](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.13/html/clusters/index) | ACM cluster management |
| [MCE SiteConfig](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.13/html/multicluster_engine_operator_with_red_hat_advanced_cluster_management/siteconfig-intro) | SiteConfig operator docs |
| [ZTP for Edge](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/edge_computing/ztp-deploying-far-edge-sites) | GitOps ZTP documentation |

### Kubernetes Documentation

| Resource | Description |
|----------|-------------|
| [Cluster API Book](https://cluster-api.sigs.k8s.io/) | CAPI concepts and usage |
| [Ignition Specification](https://coreos.github.io/ignition/) | CoreOS Ignition docs |
| [NMState Documentation](https://nmstate.io/) | Network configuration |

## Source Code Repositories

### Core Installation Components

| Repository | Description |
|------------|-------------|
| [openshift/installer](https://github.com/openshift/installer) | The `openshift-install` binary |
| [openshift/assisted-service](https://github.com/openshift/assisted-service) | Assisted Installer service |
| [openshift/assisted-installer-agent](https://github.com/openshift/assisted-installer-agent) | Discovery agent |
| [openshift/assisted-image-service](https://github.com/openshift/assisted-image-service) | ISO generation service |

### Cluster Lifecycle

| Repository | Description |
|------------|-------------|
| [openshift/hive](https://github.com/openshift/hive) | Cluster provisioning operator |
| [openshift/hypershift](https://github.com/openshift/hypershift) | Hosted control planes |
| [openshift/cluster-api-provider-agent](https://github.com/openshift/cluster-api-provider-agent) | CAPI provider for bare metal |

### Infrastructure Operators

| Repository | Description |
|------------|-------------|
| [metal3-io/baremetal-operator](https://github.com/metal3-io/baremetal-operator) | Bare metal host management |
| [openshift/machine-config-operator](https://github.com/openshift/machine-config-operator) | Node configuration |
| [openshift/machine-api-operator](https://github.com/openshift/machine-api-operator) | Machine management |

### Image-Based Installation

| Repository | Description |
|------------|-------------|
| [openshift/image-based-install-operator](https://github.com/openshift/image-based-install-operator) | IBI operator |
| [openshift/lifecycle-agent](https://github.com/openshift/lifecycle-agent) | Seed image and IBU |
| [openshift/appliance](https://github.com/openshift/appliance) | Appliance builder |

### GitOps/ZTP

| Repository | Description |
|------------|-------------|
| [openshift-kni/cnf-features-deploy](https://github.com/openshift-kni/cnf-features-deploy) | ZTP reference implementation |
| [stolostron/siteconfig](https://github.com/stolostron/siteconfig) | SiteConfig operator |

## API References

### Swagger/OpenAPI

| API | URL |
|-----|-----|
| Assisted Service REST API | [swagger.yaml](https://raw.githubusercontent.com/openshift/assisted-service/master/swagger.yaml) |
| Assisted API Viewer | [Swagger UI](https://generator.swagger.io/?url=https://raw.githubusercontent.com/openshift/assisted-service/master/swagger.yaml) |

### CRD Definitions

| CRD Group | Source |
|-----------|--------|
| Hive CRDs | [hive/apis](https://github.com/openshift/hive/tree/master/apis/hive/v1) |
| Assisted CRDs | [assisted-service/api](https://github.com/openshift/assisted-service/tree/master/api) |
| HyperShift CRDs | [hypershift/api](https://github.com/openshift/hypershift/tree/main/api) |
| Metal3 CRDs | [baremetal-operator/apis](https://github.com/metal3-io/baremetal-operator/tree/main/apis) |

## Learning Resources

### Blogs and Articles

| Resource | Description |
|----------|-------------|
| [Red Hat Developer Blog](https://developers.redhat.com/blog) | Technical articles |
| [OpenShift Blog](https://www.redhat.com/en/blog/channel/red-hat-openshift) | Product announcements and guides |
| [cloud.redhat.com/blog](https://cloud.redhat.com/blog) | Assisted Installer articles |

### Videos and Tutorials

| Resource | Description |
|----------|-------------|
| [OpenShift YouTube](https://www.youtube.com/c/OpenShift) | Official channel |
| [Red Hat Summit Sessions](https://www.redhat.com/en/summit) | Conference recordings |

### Community

| Resource | Description |
|----------|-------------|
| [OpenShift Commons](https://commons.openshift.org/) | Community meetings and resources |
| [Kubernetes Slack](https://slack.k8s.io/) | #openshift channel |
| [Red Hat Customer Portal](https://access.redhat.com/) | Support and knowledge base |

## Tools

### CLI Tools

| Tool | Description | Installation |
|------|-------------|--------------|
| `oc` | OpenShift CLI | [Download](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/) |
| `openshift-install` | Installer CLI | [Download](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/) |
| `hypershift` | HyperShift CLI | Built from [hypershift repo](https://github.com/openshift/hypershift) |
| `hiveutil` | Hive utilities | Built from [hive repo](https://github.com/openshift/hive) |

### Development Tools

| Tool | Description |
|------|-------------|
| [kind](https://kind.sigs.k8s.io/) | Kubernetes in Docker for testing |
| [skipper](https://github.com/Stratoscale/skipper) | Build container environment |
| [podman](https://podman.io/) | Container runtime |

## Quick Reference

### Common Commands

```bash
# IPI Installation
openshift-install create cluster --dir=cluster

# Agent-Based Installation
openshift-install agent create image --dir=cluster

# Check cluster status
oc get clusterversion
oc get clusteroperators

# Assisted Service status (on-premise)
oc get agentserviceconfig
oc get clusterdeployment,agentclusterinstall,infraenv,agent -A

# HyperShift status
oc get hostedcluster,nodepool -A

# ZTP status
oc get clusterinstance -A
```

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `KUBECONFIG` | Path to kubeconfig file |
| `OPENSHIFT_INSTALL_RELEASE_IMAGE_OVERRIDE` | Custom release image |
| `OPENSHIFT_INSTALL_OS_IMAGE_OVERRIDE` | Custom RHCOS image |
| `OPENSHIFT_INSTALL_INVOKER` | Installation source tracking |

## Troubleshooting Resources

### Log Collection

```bash
# Installer logs
openshift-install gather bootstrap --dir=cluster

# Assisted Service logs
oc logs -n multicluster-engine -l app=assisted-service

# Hive logs
oc logs -n multicluster-engine -l app=hive
```

### Diagnostic Tools

| Tool | Purpose |
|------|---------|
| `must-gather` | Collect diagnostic data |
| `oc adm inspect` | Inspect resources |
| `oc debug node/` | Debug node issues |

---

## Contributing to This Documentation

This documentation is maintained alongside the OpenShift installation repositories. To contribute:

1. Fork the repository
2. Make your changes
3. Submit a pull request

For repository-specific contribution guidelines, see the CONTRIBUTING.md in each repository.

