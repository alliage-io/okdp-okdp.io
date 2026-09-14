---
title: Upgrade from KuboCD
description: Moving an OKDP platform deployed with KuboCD to Helm charts in a deployments Git repository, reconciled by Flux or Argo CD.
---

Earlier OKDP versions deployed their services with [KuboCD](https://www.kubocd.io/): a KuboCD `Release` per instance, a `Context` for the platform values and the service catalog, `ClusterContract` and `Connection` objects for connections. OKDP now deploys Helm charts described by files in a [deployments Git repository](/en/installation-requirements/deployments-repository), reconciled by Flux or Argo CD. This page lists what changes and the order of operations.

There is no in-place upgrade: every Helm release and most resource names change, so each instance is installed anew and starts with new volumes. Data kept outside of the instance (buckets of the S3 store, databases reached through a connection) stays where it is; data kept in the volumes of an instance must be moved by hand.

This page covers the services of the projects. Platform components (cert-manager, External Secrets Operator, CloudNativePG, Keycloak, ...) are also renamed, and several of them move to new major versions; the simplest path is to install them on a new cluster from the reference layout of [OKDP Sandbox](https://github.com/OKDP/okdp-sandbox). To upgrade them in place, see [Platform components](#platform-components).

## From KuboCD objects to files

| KuboCD                                                              | Now                                                                                                                |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `Context` (platform values)                                         | `platform/platform-values.yaml`, under `global.okdp`, same keys: `.Context.X` becomes `.Values.global.okdp.X`      |
| `serviceCatalog` of the `Context`                                   | `platform/catalog.yaml`, `defaultRepository: oci://quay.io/okdp/platform-charts`                                   |
| `Release` `<project>-<instance>`                                    | `projects/<project>/services/<instance>/instance.yaml` and `values.yaml` (the former `spec.parameters`)            |
| package `quay.io/okdp/platform-packages/<service>`, tag `4.0.1-p02` | chart `oci://quay.io/okdp/platform-charts/<service>`, version `4.0.1-1.0.1`                                        |
| `Connection` object                                                 | `projects/<project>/connections/<name>.yaml` and a Secret; `secretRef: <name>` becomes `secretRef: {name: <name>}` |
| connection published by a release, `kcd-<release>-<output>`         | the release name `<project>-<instance>`                                                                            |
| `ClusterContract`                                                   | contract schemas of the `okdp-lib` chart                                                                           |
| `Release` status                                                    | Flux `HelmRelease` or Argo CD `Application`, and the descriptor ConfigMap `<release>-okdp`                         |
| `protected: true`, `kubocd-webhooks`                                | label `okdp.io/protected: "true"`, enforced by ValidatingAdmissionPolicies of the `tools` component                |
| dependencies between releases                                       | none between services; layers `00`, `10`, `20`, `30` between platform components                                   |

## Renamed resources

The Helm release of an instance is `<project>-<instance>` (for example `demo-trino`), and there is one release per instance instead of one per KuboCD module: the `-main` suffix disappears from every name.

- Services: Trino `<release>-trino-*` (was `<release>-main-trino-*`), Hive Metastore `<release>-hive-metastore` (was `<release>`), Polaris `<release>-polaris` (was `<release>-main`), Spark History Server `<release>-spark-history-server` (was `<release>-main`), Ollama `<release>` (was `<release>-main`), Open WebUI `<release>-open-webui` (was `<release>-main-open-webui`), Superset cache and Celery broker, now Valkey, Service `<release>-redis` (was the Redis `<release>-main-redis-headless`).
- Generated Secrets: `creds-<release>-internal` becomes `<release>-internal` (Trino, Airflow, Superset), and `creds-<release>-root` becomes `<release>-root` (Polaris). The OAuth client Secrets keep their name, `creds-<release>-oauth2`.
- TLS Secrets: `airflow-tls` becomes `<release>-airflow-tls`, `polaris-console-tls` becomes `<release>-polaris-console-tls`, and the console's `okdp-ui-tls` becomes `<release>-tls`.
- Persistent volume claims follow the release name: the new instances start with empty volumes.
- CloudNativePG databases of the `cnpg-postgresql` component: the `Database` objects are `<release>-<database>` (were `<database>`), and their connections `<release>-<database>` (were `kcd-<release>-<database>`).

## Data to move by hand

- Ollama: the models are pulled again.
- Open WebUI (`ollama-ui`): conversations and accounts start empty; copy the former data volume to keep them.
- RustFS: its Service, volumes and Secrets change name; copy the data of the store.
- Superset: the cache and the Celery broker are now [Valkey](https://valkey.io/) (was Redis with a volume), part of the chart and not persisted: the cache and the Celery queue start empty. The Superset Deployments also select their pods on new labels (Apache Superset chart 0.22.8): they are recreated, not updated in place, and their pods restart.

## Users and OIDC clients

- JupyterHub: the cookie secret and the auth-state keys are generated anew by ESO (Secret `<release>-hub-generated`): users sign in again.
- Polaris: the root credentials are generated into `<release>-root` and kept when the instance is deleted. A realm already bootstrapped in the database knows the former credentials of `creds-<release>-root`: keep a copy of that Secret.

## kubauth is removed

OKDP no longer ships kubauth: identity is Keycloak only. `clientProvisioning` accepts `existing` (clients created in Keycloak beforehand) or `dcr` (dynamic client registration); `kubauth` is no longer a valid value.

A platform that uses kubauth must move its users, groups and OIDC clients to Keycloak, and set `clientProvisioning` to `existing` or `dcr`, **before** upgrading. No migration tool is provided: recreate them in Keycloak by hand or with your own scripts. Users sign in again afterwards, and the services get new client secrets. Once the users and groups are in Keycloak, the console Identity page manages them again, provided the control-plane server has Keycloak admin credentials (see [Identity provider](/en/installation-requirements#identity-provider)).

With `dcr`, the OIDC client of a service is registered in Keycloak by a Job of its chart. Deleting the service does not remove that client from Keycloak, with Flux as with Argo CD: delete it in the Keycloak console.

## Parameter changes

- References to other instances use the release name: `kcd-demo-hive-metastore` becomes `demo-hive`, `kcd-demo-polaris-catalog` becomes `demo-polaris`, `kcd-demo-trino-endpoint` becomes `demo-trino`. Trino needs `warehouse` on a catalog of a Polaris instance of the project.
- JupyterHub: the parameter `connections` (PySpark connections) is renamed `sparkConnections`, since `connections` now holds the external connections of the release.
- Parameters are no longer templates. The KuboCD escape ``{{ `{{ ... }}` }}`` is not needed any more: write the value as it must reach the service. JupyterHub still replaces `{{ .Context.ingress.suffix }}` (or `{{ .Values.global.okdp.ingress.suffix }}`), `{{ .Release.Name }}` and `{{ .Release.Namespace }}` in `welcomeNotebook`, `sparkDefaults` and the `properties` of `sparkConnections`.
- Spark History Server: OIDC is enabled unless `global.okdp.oidc.enabled` is `false` (the package defaulted to `false` when the key was missing).
- Control plane server: `kubocdNamespace`, `releaseInterval` and `releaseTimeout` are gone; the Git settings are under `gitops` (repository URL, branch, path, engine, credentials Secret).

## Platform components

The components of the reference layout run newer upstream versions than the KuboCD packages. If you keep the cluster and upgrade them in place instead of installing a new one, follow the upstream upgrade paths below; do not jump straight to the new versions.

### External Secrets Operator

ESO moves from 0.15 to 2.11, and the API from `external-secrets.io/v1beta1` to `external-secrets.io/v1`: ESO 2.x no longer serves `v1beta1`, and every OKDP chart now renders `v1` (the `Password` generators stay `generators.external-secrets.io/v1alpha1`).

1. Upgrade ESO from 0.15 to 0.16, the release that serves both `v1` and `v1beta1`.
2. Move every ESO manifest to `v1`: the OKDP charts do it themselves, your own `SecretStore`, `ClusterSecretStore` and `ExternalSecret` manifests (and anything that creates them) must be changed. Objects already stored as `v1beta1` must be rewritten as `v1` (for example an update of each object) before `v1beta1` is removed from the `status.storedVersions` of the ESO CRDs.
3. Upgrade ESO to 2.x (the reference layout installs the upstream chart 2.11.0). ESO 2.0 removed the Alibaba and Device42 providers.

### cert-manager

cert-manager moves from 1.17 to 1.21 (and trust-manager from 0.16 to 0.25). Upgrade one minor version at a time, 1.17 → 1.18 → 1.19 → 1.20 → 1.21, as the cert-manager project recommends, and read each release's notes: since 1.18, `Certificate.spec.privateKey.rotationPolicy` defaults to `Always` (a new private key on every renewal).

### Keycloak

The Keycloak component moves from the Bitnami chart and image to the codecentric `keycloakx` chart with the official image (`quay.io/keycloak/keycloak` 26.7), run in production mode. It keeps its database: the same `database-server` connection, Keycloak migrates the schema when it starts. The StatefulSet selector differs, so the release cannot be upgraded in place: delete the Bitnami StatefulSet of the Keycloak release first (the data is in the database), then let the engine install the new chart. The realm is applied by keycloak-config-cli, which removes the role mappings and user profile settings the realm file does not declare: declare in the component values what was granted by hand (for example the roles of the console's service account and the `comment` and `uid` user attributes).

### Ingress NGINX

ingress-nginx is retired upstream: 4.15.1, which the reference layout installs, is its final release, and no fix, including for security vulnerabilities, will follow. The migration to another ingress controller is future work; until then, do not expose the ingress to untrusted users.

## Order of operations

1. **Prerequisites.** Make sure the External Secrets Operator 2.x runs (API `external-secrets.io/v1`, see [External Secrets Operator](#external-secrets-operator) for the path from 0.15), and prepare the GitOps engine: Flux, which KuboCD already runs on, with helm-controller started with `--feature-gates=DisableChartDigestTracking=true`, or Argo CD with progressive syncs enabled. Kubernetes 1.30 or later. See [Deployments repository](/en/installation-requirements/deployments-repository).
2. **Platform values.** Create the deployments repository from the reference layout. Move the values of the `Context` into `platform/platform-values.yaml` under `global.okdp` (same keys), and its `serviceCatalog` into `platform/catalog.yaml`, with chart versions.
3. **Instance files.** Convert each `Connection` into `projects/<project>/connections/<name>.yaml` (the contract and the fields of `spec.values`, with `secretRef: {name: ...}`), and each `Release` into `projects/<project>/services/<instance>/`: `instance.yaml` (the instance name from the label `okdp.io/instance-name`, the project from the namespace, the service from the label `okdp.io/service`, the chart and its version, the connection files it uses) and `values.yaml` (`spec.parameters`, with the changes above). With Flux, run `scripts/render-flux.sh`; then `scripts/check.sh`.
4. **Back up** the data listed above.
5. **Remove KuboCD.** Delete the KuboCD `Release` objects, which uninstalls their Helm releases, then the `Connection`, `Context` and `ClusterContract` objects, the KuboCD controller and its CRDs, and the `kubocd-webhooks` component. The former and the new instances cannot run side by side: they serve the same ingress hosts.
6. **Start the engine** on the deployments repository: `kubectl apply -f gitops/flux/sync.yaml`, or `kubectl apply -n argocd -f gitops/argocd/`.
7. **Restore** the data, and tell the users to sign in again.
