---
title: Deployments repository
description: Layout of the OKDP deployments Git repository, its values layers, and the installation with Flux or Argo CD.
sidebar:
  order: 2
---

The deployments repository is the only desired state of an OKDP platform. The console (through the control plane server) and GitOps users write the same files; Flux **or** Argo CD deploys them. Helm renders the same chart with the same values layers under both engines.

The reference layout, its scripts and the byte-exact specification of every file are in the [`gitops/` directory of OKDP Sandbox](https://github.com/OKDP/okdp-sandbox/tree/main/gitops).

## Layout

```
platform/
  platform-values.yaml            # {global: {okdp: {...}}}: first values layer of every release
  catalog.yaml                    # console service catalog
  connections/<name>.yaml         # {connections: {<name>: {...}}}, external connections of platform components
  kustomization.yaml              # generated: platform values + platform connection ConfigMaps (both engines)
  components/<NN>-<name>/         # platform components; NN = layer 00, 10, 20 or 30
    instance.yaml  values.yaml    #   written by hand
    helmrelease.yaml  kustomization.yaml   # generated (Flux)
projects/<project>/
  project.yaml                    # {name, description, ...}; namespace = <project>
  connections/<name>.yaml         # {connections: {<name>: {...}}}  (external connection)
  services/<instance>/
    instance.yaml                 # engine-neutral source of both engines
    values.yaml                   # user parameters only
    helmrelease.yaml              # generated (Flux)
    kustomization.yaml            # generated (Flux)
  kustomization.yaml              # generated (Flux): services + connection ConfigMaps
flux/                             # Flux entry point
argocd/                           # Argo CD entry point
scripts/render-flux.sh            # instance.yaml -> generated Flux files
scripts/check.sh                  # CI: generated files up to date, shapes and contracts valid
```

## A service instance

A service instance is a directory `projects/<project>/services/<instance>/` with two files written by people or by the console.

`instance.yaml` says which chart to deploy:

```yaml
name: hive # the instance, equals the directory name
project: demo # the project, equals the project directory and the target namespace
service: hive-metastore # the chart name
chart: oci://quay.io/okdp/platform-charts/hive-metastore
version: 4.0.1-1.0.1 # exact chart version
connections: # connection files to layer in, in this order
  - demo-db-hive
  - demo-storage
```

The rules, checked by the console and by `scripts/check.sh`:

- exactly these six keys; `connections: []` when there is none;
- `name`, `project` and `service` are DNS labels, `chart` is an `oci://` reference ending with `/<service>`;
- `version` is an exact version (`4.0.1-1.0.1`), not a range; quote it only when YAML would read it as something else than a string (`"6.10"`);
- the Helm release name is `<project>-<instance>` (here `demo-hive`): at most 53 characters, and unique across projects and platform components (project `a-b` with instance `c` collides with project `a` with instance `b-c`);
- each name in `connections` is a file `projects/<project>/connections/<name>.yaml`.

`values.yaml` holds only the parameters the user set (`{}` when there is none). The chart defaults apply to everything else. It never contains `global.okdp` nor `connections`.

```yaml
db: demo-db-hive
storage: demo-storage
s3SecretRef: creds-hive-metastore-s3
warehouseBucket: hive
```

`project.yaml` describes the project: `{name: demo, description: ...}`, where `name` equals the directory name.

Platform components (`platform/components/<NN>-<name>/`) have the same two files; their `project` is the target namespace, the release is still `<project>-<name>`, and their `connections` name files of `platform/connections/` (for example `keycloak-db`, the database of Keycloak), whose `secretRef` names a Secret of the component's namespace. A layer starts when every component of the previous non-empty layer is ready.

## Values layers

Every release receives three layers, in this exact order under both engines (later layers win, maps merge, lists are replaced, as with `helm -f a -f b`):

1. `platform/platform-values.yaml`
2. each connection file listed in `connections`, in that order: `projects/<project>/connections/<name>.yaml` for a service, `platform/connections/<name>.yaml` for a platform component
3. the `values.yaml` of the instance

## Generated files (Flux)

Flux needs a `HelmRelease` and an `OCIRepository` per instance, and ConfigMaps carrying the values layers. They are generated from the `instance.yaml` files by `scripts/render-flux.sh` (bash 4+ and yq v4), never edited by hand: `helmrelease.yaml` and `kustomization.yaml` of each instance, `projects/<project>/kustomization.yaml`, and, for platform administrators, `platform/kustomization.yaml` and `flux/components.yaml`. The console writes byte-identical files, and `scripts/check.sh` fails when a generated file is not up to date. Argo CD ignores these files.

The HelmReleases, OCIRepositories and values ConfigMaps (`okdp-platform-values`, `conn-<project>-<name>`, `okdp-platform-conn-<name>`, `values-<project>-<instance>`) all live in namespace `okdp-releases`; the Helm release is stored in the target namespace.

## Install with Flux

Prerequisites: Flux (tested with v2.9.5: source-controller, kustomize-controller, helm-controller), a Git server holding the repository, and helm-controller started with `--feature-gates=DisableChartDigestTracking=true`. Without it, helm-controller appends the OCI digest to the chart version, so the chart version seen by the templates differs from Argo CD's:

```sh
kubectl -n flux-system patch deployment helm-controller --type json \
  -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--feature-gates=DisableChartDigestTracking=true"}]'
```

With `flux bootstrap`, add the same patch to `flux-system/kustomization.yaml`.

1. Set the Git URL and branch in `flux/sync.yaml` (and the paths `./gitops/...` in `flux/sync.yaml` and `flux/platform.yaml` if the layout is not under `gitops/`; then render with `scripts/render-flux.sh --path-prefix <prefix>`). Commit.
2. `kubectl apply -f gitops/flux/sync.yaml`

Flux then manages `flux/` itself, the platform values, the platform components (ordered by layer) and all projects. Deleting an instance directory uninstalls the release.

## Install with Argo CD

Prerequisites: Argo CD (tested with v3.4.2) with progressive syncs enabled on the ApplicationSet controller, needed by the layer ordering of platform components:

```sh
kubectl -n argocd patch configmap argocd-cmd-params-cm --type merge \
  -p '{"data":{"applicationsetcontroller.enable.progressive.syncs":"true"}}'
kubectl -n argocd rollout restart deployment argocd-applicationset-controller
```

1. Set the Git URL and branch in `argocd/root.yaml`, `argocd/platform-values.yaml`, `argocd/components.yaml` and `argocd/services.yaml` (and the `gitops/` prefix of the paths if the layout is elsewhere). Commit.
2. `kubectl apply -n argocd -f gitops/argocd/`

Argo CD then manages `argocd/` itself, the platform values, and one Application `<project>-<instance>` per platform component and per service instance. Each Application has two sources, the OCI chart and the Git repository, and lists the values layers in the order above. Deleting an instance directory deletes the Application and its resources.

## What the console writes

| Action in the console | Files written                                                                                               | Files deleted          |
| --------------------- | ----------------------------------------------------------------------------------------------------------- | ---------------------- |
| Deploy an instance    | `instance.yaml`, `values.yaml`, `helmrelease.yaml`, `kustomization.yaml`; `projects/<p>/kustomization.yaml` |                        |
| Edit parameters       | `values.yaml`                                                                                               |                        |
| Change the version    | `instance.yaml`, `helmrelease.yaml`                                                                         |                        |
| Delete an instance    | `projects/<p>/kustomization.yaml`                                                                           | the instance directory |
| Create a connection   | `projects/<p>/connections/<c>.yaml`, `projects/<p>/kustomization.yaml`                                      |                        |
| Delete a connection   | `projects/<p>/kustomization.yaml`                                                                           | the connection file    |
| Create a project      | `project.yaml`, `projects/<p>/kustomization.yaml`                                                           |                        |
| Delete a project      |                                                                                                             | `projects/<p>/`        |
| Edit the catalog      | `platform/catalog.yaml`                                                                                     |                        |

Each change is one commit, `okdp: <action> <project>/<instance> by <user>`, with a `Co-Authored-By: <name> <<email>>` trailer naming the logged-in user. The trailer needs the `email` claim in the access token (Keycloak `email` client scope, and `profile` for the name); without an email, the commit only names the user in its subject. The console never writes the platform components.
