---
title: Installation requirements
description: Link to the OKDP Sanbox repository for a quick installation and the description of necessary components for an OKDP cluster installation.
---

[OKDP Sandbox](https://github.com/OKDP/okdp-sandbox) is a quick-start installation of an OKDP cluster. By following the README.md, you should have access the to all current OKDP services.

The following installation guide explains the mandatory dependencies that OKDP v1 needs in order to have a working cluster.

## KuboCD objects

[Kubocd](https://www.kubocd.io/) is an overlayer of [Flux](https://fluxcd.io/) for managing `helmRelease` object linked to [Helm charts](https://helm.sh/). Flux must be installed prior to the KuboCD CRD for the latter to work.

### KuboCD release

A KuboCD release contains one or several Helm charts with a dependency order between them if necessary and dynamically overwrites their `values.yaml` with variables set in a `context` and /or a `clusterContract` object.

### Context

A context is a Kubernetes object belonging to the KuboCD environment. It is basically a set of defined variables for KuboCD releases to have access to. The purpose is to centralize variables that are used through several releases and define the OKDP environment, like the ingress suffix, proxy HTTP/HTTPS address,.etc. The context also contains the service catalog.

### Cluster contract

A cluster contract (`clusterContract` with the kubectl command) is also a Kubernetes object belonging to the KuboCD environment, defining variables that are, in contrast to the context, specific for a service or a component.

## Main mandatory dependencies

### Storage

OKDP does not provide storage, and an S3-compatible storage provider like [SeaweedFS](https://github.com/seaweedfs/seaweedfs) or [Ceph](https://ceph.io/en/) is required beforehand for the platform. OKDP communicates through the S3 API to the storage provider. Any S3 storage provider that also performs [STS](https://docs.aws.amazon.com/STS/latest/APIReference/Welcome.html), since it is being used by Polaris for securing its catalogs, should be compatible with OKDP; here, for the demonstration purposes, SeaweedFS is being used.

A database server controller like [CloudNativePG](https://cloudnative-pg.io/) must be provided too. A database server controller is a Kubernetes operator enabling you to manage a database `cluster` to handle the lifecycle of `database` objects in a Kubernetes native way.

Once the storage provider and the database-server controller are present, OKDP communicates with them through connections explained in [Connections and external secrets management](/en/admin-guide/connections-and-external-secrets-management).

How to create a view of the storage provider's user interface is shown in the [user guide](/en/user-guide#project-panel).

The storage provider as well as the database server controller must be provided through a kubocd package with `storage` or `database-server` marked in their metadata. Otherwise, some services depending on them are undeployable.

### Ingress controller

An [Ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) must be installed to access the user interfaces of all services and the OKDP control plane itself. For the v1 of OKDP, Nginx is being used, knowing that it is deprecated; however, migration to other ingress controllers will be applied later on.

### Identity provider

OKDP currently works and has been tested with [Keycloak](https://www.keycloak.org/) and [KubeAuth](https://www.kubeauth.io/) for OIDC authentication. Please refer to their documentation for any further information.

## Other dependencies

The OKDP sandbox currently depends on other components such as a DNS server, a cert manager, and other tools, which you can see on the [stack](/en/stack/okdp-1-0) page.
