# OpenShift Installation Guide

A developer-focused guide to OpenShift cluster installation methods, with emphasis on the assisted-service and related components.

## What This Guide Covers

This guide explains:
- **How** OpenShift clusters are installed
- **Which method** to choose for your scenario
- **What components** are involved (operators, controllers, CRDs)
- **Where** to find source code and APIs

## How to Use This Guide

| If you want to... | Start here |
|-------------------|------------|
| Learn the basics first | [Getting Started](00-getting-started.md) |
| Choose an installation method | [Installation Methods Overview](01-installation-methods-overview.md) |
| Install on a cloud provider | [Traditional Installers](02-traditional-installers/index.md) |
| Use the Assisted Installer | [Assisted Installation](03-assisted-installation/overview.md) |
| Deploy in disconnected environments | [Agent-Based Installer](03-assisted-installation/abi.md) |
| Deploy SNO quickly | [Image-Based Installation](04-image-based-installation/index.md) |
| Run control planes as pods | [Hosted Control Planes](05-hosted-control-planes/index.md) |
| Deploy at scale with GitOps | [GitOps Provisioning](06-gitops-provisioning/index.md) |
| Understand the operator ecosystem | [Operators & Controllers](07-operators-controllers/overview.md) |
| Look up CRD specifications | [CRD Reference](08-crd-reference/index.md) |
| Find external resources | [Resources](10-resources.md) |
| Look up an abbreviation | [Glossary](11-glossary.md) |

## Relationship to Repository Documentation

This guide provides a **conceptual overview** and **cross-references** to help navigate the extensive documentation in the assisted-service and related repositories. It is not intended to duplicate detailed guides that already exist.

For in-depth information, refer to:
- [assisted-service/docs](https://github.com/openshift/assisted-service/tree/master/docs) - Comprehensive Assisted Installer documentation
- [assisted-service/docs/user-guide](https://github.com/openshift/assisted-service/tree/master/docs/user-guide) - Step-by-step tutorials
- [assisted-service/docs/hive-integration](https://github.com/openshift/assisted-service/tree/master/docs/hive-integration) - Kubernetes CRD documentation
- [assisted-service/docs/dev](https://github.com/openshift/assisted-service/tree/master/docs/dev) - Developer setup and testing

## Key Repositories

| Component | Repository |
|-----------|------------|
| OpenShift Installer | [openshift/installer](https://github.com/openshift/installer) |
| Assisted Service | [openshift/assisted-service](https://github.com/openshift/assisted-service) |
| Hive | [openshift/hive](https://github.com/openshift/hive) |
| HyperShift | [openshift/hypershift](https://github.com/openshift/hypershift) |
| Baremetal Operator | [metal3-io/baremetal-operator](https://github.com/metal3-io/baremetal-operator) |

See [Resources](10-resources.md) for a complete list.

