---
title: Installation requirements
description: Link to the OKDP Sandbox repository for a quick installation, the OKDP deployment model, and the components an OKDP cluster needs.
sidebar:
  order: 1
---

[OKDP Sandbox](https://github.com/OKDP/okdp-sandbox) is a quick-start installation of an OKDP cluster. By following the README.md, you should have access to all current OKDP services.

The following installation guide explains how OKDP deploys its services and the mandatory dependencies it needs in order to have a working cluster.

## Deployment model

OKDP only relies on standard tooling: [Helm](https://helm.sh/), a GitOps engine ([Flux](https://fluxcd.io/) or [Argo CD](https://argo-cd.readthedocs.io/)), the [External Secrets Operator](https://external-secrets.io/) and Kubernetes built-in objects. There is no OKDP controller, no OKDP custom resource and no database.

### Helm charts

Every OKDP service (Hive Metastore, Trino, JupyterHub, ...) and every platform component is a Helm chart, published to `oci://quay.io/okdp/platform-charts`, `oci://quay.io/okdp/community-charts` or `oci://quay.io/okdp/sandbox-charts`. Helm is the only renderer: the console, Flux and Argo CD all render the same chart with the same values.

### Deployments Git repository

A Git repository holds the whole desired state of the platform: one directory per project, one directory per service instance containing an `instance.yaml` file (which chart, which version) and a `values.yaml` file (the parameters). Flux or Argo CD deploys what this repository describes. The OKDP console writes the very same files, so deploying from the console and writing the files by hand in Git give the same result.

The layout and the installation with Flux or Argo CD are described in [Deployments repository](/en/installation-requirements/deployments-repository).

### Platform values

The values shared by every release (ingress suffix, OIDC provider, HTTP/HTTPS proxy, certificate issuer, storage classes) live in one file of the deployments repository, `platform/platform-values.yaml`, under the key `global.okdp`. It is the first values layer of every release. The console service catalog is a separate file, `platform/catalog.yaml`.

```yaml
global:
  okdp:
    ingress:
      suffix: okdp.sandbox
      className: nginx
    oidc:
      enabled: true
      issuerUri: https://keycloak.okdp.sandbox/realms/master
      clientId: okdp-ui
      # authUrl, tokenUrl, jwksUri, userinfoUrl, displayName, scope, usePKCE, ...
      clientProvisioning: dcr
      dcr:
        registrationUrl: https://keycloak.okdp.sandbox/realms/master/clients-registrations/openid-connect
        authMethod: anonymous
    proxy:
      httpProxy: ""
      httpsProxy: ""
      noProxy: ""
    certificateIssuers:
      selfSigned:
        name: default-issuer
    storageClass:
      data: standard
      workspace: standard
```

### Contracts and connections

A service reaches another service or an external component (S3 storage, database server) through a connection. A connection follows a contract, which fixes its fields: `database-server`, `s3`, `hive`, `iceberg-catalog` and `trino`. Non-secret fields are written in Git; credentials stay in a Kubernetes Secret. See [Connections and external secrets management](/en/admin-guide/connections-and-external-secrets-management).

## Main mandatory dependencies

### GitOps engine and Git server

OKDP needs a Git server holding the deployments repository, with write access for the console, and one of the following engines:

- Flux (tested with v2.9.5), with helm-controller started with `--feature-gates=DisableChartDigestTracking=true`;
- Argo CD (tested with v3.4.2), with progressive syncs enabled on the ApplicationSet controller.

Both settings are explained in [Deployments repository](/en/installation-requirements/deployments-repository).

### External Secrets Operator

The [External Secrets Operator](https://external-secrets.io/) (ESO) is mandatory. OKDP charts never generate a password with Helm: generated passwords come from an ESO `Password` generator and an `ExternalSecret`, and credentials shared across namespaces are delivered by ESO. The platform runs ESO 2.11 (API `external-secrets.io/v1`; `v1beta1` is no longer served, generators stay `generators.external-secrets.io/v1alpha1`). The console also uses it to synchronize secrets from a secret store such as Vault.

### Kubernetes version

Kubernetes 1.30 or later is required: the deletion protection of critical platform objects (label `okdp.io/protected: "true"`) is enforced by ValidatingAdmissionPolicies, shipped by the `tools` platform component.

### Storage

OKDP does not provide storage, and an S3-compatible storage provider like [SeaweedFS](https://github.com/seaweedfs/seaweedfs), [RustFS](https://rustfs.com/) or [Ceph](https://ceph.io/en/) is required beforehand for the platform. OKDP communicates through the S3 API to the storage provider. Any S3 storage provider that also performs [STS](https://docs.aws.amazon.com/STS/latest/APIReference/Welcome.html), since it is being used by Polaris for securing its catalogs, should be compatible with OKDP; the sandbox uses SeaweedFS.

A database server controller like [CloudNativePG](https://cloudnative-pg.io/) must be provided too. A database server controller is a Kubernetes operator enabling you to manage a database `cluster` to handle the lifecycle of `database` objects in a Kubernetes native way.

Once the storage provider and the database server are present, OKDP services reach them through connections of contract `s3` and `database-server`, explained in [Connections and external secrets management](/en/admin-guide/connections-and-external-secrets-management). These two contracts are always external connections: a connection file in the project states the address of the store or of the database, and a Secret of the project holds the credentials. Services depending on them cannot be deployed without such a connection.

How to create a view of the storage provider's user interface is shown in the [user guide](/en/user-guide#project-panel).

### Ingress controller

An [Ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) must be installed to access the user interfaces of all services and the OKDP control plane itself. The sandbox uses [ingress-nginx](https://github.com/kubernetes/ingress-nginx), knowing that the project is retired: 4.15.1 is its final release and no fix, including for security vulnerabilities, will follow. The migration to another ingress controller is future work. The ingress class used by every chart is `global.okdp.ingress.className`.

### Identity provider

OKDP currently works and has been tested with [Keycloak](https://www.keycloak.org/) and [KubeAuth](https://www.kubeauth.io/) for OIDC authentication. Please refer to their documentation for any further information. The provider is described once in the platform values (`global.okdp.oidc`); `clientProvisioning` tells how the OIDC clients of the services are obtained (`existing` Secrets, `dcr` dynamic registration, or `kubauth`).

## Other dependencies

The OKDP sandbox currently depends on other components such as a DNS server (CoreDNS), cert-manager and trust-manager, and other tools, which you can see on the [stack](/en/stack/okdp-1-0) page. In the deployments repository, they are platform components installed in layers: `00` CRDs and operators, `10` infrastructure, `20` identity, storage and databases, `30` control plane.
