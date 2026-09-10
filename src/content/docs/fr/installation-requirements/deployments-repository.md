---
title: Dépôt de déploiement
description: Organisation du dépôt Git de déploiement d'OKDP, ses couches de valeurs, et l'installation avec Flux ou Argo CD.
sidebar:
  order: 2
---

Le dépôt de déploiement est le seul état souhaité d'une plateforme OKDP. La console (par l'intermédiaire du serveur du plan de contrôle) et les utilisateurs GitOps écrivent les mêmes fichiers ; Flux **ou** Argo CD les déploie. Helm rend la même charte avec les mêmes couches de valeurs avec les deux moteurs.

L'organisation de référence, ses scripts et la spécification exacte, à l'octet près, de chaque fichier se trouvent dans le [répertoire `gitops/` d'OKDP Sandbox](https://github.com/OKDP/okdp-sandbox/tree/main/gitops).

## Organisation

```
platform/
  platform-values.yaml            # {global: {okdp: {...}}} : première couche de valeurs de chaque release
  catalog.yaml                    # catalogue de services de la console
  connections/<name>.yaml         # {connections: {<name>: {...}}}, connexions externes des composants de plateforme
  kustomization.yaml              # généré : ConfigMaps des valeurs et des connexions de la plateforme (deux moteurs)
  components/<NN>-<name>/         # composants de plateforme ; NN = couche 00, 10, 20 ou 30
    instance.yaml  values.yaml    #   écrits à la main
    helmrelease.yaml  kustomization.yaml   # générés (Flux)
projects/<project>/
  project.yaml                    # {name, description, ...} ; namespace = <project>
  connections/<name>.yaml         # {connections: {<name>: {...}}}  (connexion externe)
  services/<instance>/
    instance.yaml                 # source neutre des deux moteurs
    values.yaml                   # paramètres de l'utilisateur uniquement
    helmrelease.yaml              # généré (Flux)
    kustomization.yaml            # généré (Flux)
  kustomization.yaml              # généré (Flux) : services + ConfigMaps des connexions
flux/                             # point d'entrée de Flux
argocd/                           # point d'entrée d'Argo CD
scripts/render-flux.sh            # instance.yaml -> fichiers Flux générés
scripts/check.sh                  # CI : fichiers générés à jour, formes et contrats valides
```

## Une instance de service

Une instance de service est un répertoire `projects/<project>/services/<instance>/` contenant deux fichiers écrits par une personne ou par la console.

`instance.yaml` indique la charte à déployer :

```yaml
name: hive # l'instance, égale au nom du répertoire
project: demo # le projet, égal au répertoire du projet et au namespace cible
service: hive-metastore # le nom de la charte
chart: oci://quay.io/okdp/platform-charts/hive-metastore
version: 4.0.1-1.0.1 # version exacte de la charte
connections: # fichiers de connexion à ajouter comme couches, dans cet ordre
  - demo-db-hive
  - demo-storage
```

Les règles, vérifiées par la console et par `scripts/check.sh` :

- exactement ces six clés ; `connections: []` s'il n'y en a aucune ;
- `name`, `project` et `service` sont des libellés DNS, `chart` est une référence `oci://` qui se termine par `/<service>` ;
- `version` est une version exacte (`4.0.1-1.0.1`), pas un intervalle ; mettez-la entre guillemets uniquement si YAML la lirait autrement que comme une chaîne (`"6.10"`) ;
- le nom de la release Helm est `<project>-<instance>` (ici `demo-hive`) : 53 caractères au plus, et unique parmi les projets et les composants de plateforme (le projet `a-b` avec l'instance `c` entre en collision avec le projet `a` avec l'instance `b-c`) ;
- chaque nom de `connections` est un fichier `projects/<project>/connections/<name>.yaml`.

`values.yaml` ne contient que les paramètres définis par l'utilisateur (`{}` s'il n'y en a aucun). Les valeurs par défaut de la charte s'appliquent au reste. Il ne contient jamais `global.okdp` ni `connections`.

```yaml
db: demo-db-hive
storage: demo-storage
s3SecretRef: creds-hive-metastore-s3
warehouseBucket: hive
```

`project.yaml` décrit le projet : `{name: demo, description: ...}`, où `name` est égal au nom du répertoire.

Les composants de plateforme (`platform/components/<NN>-<name>/`) ont les deux mêmes fichiers ; leur `project` est le namespace cible, la release reste `<project>-<name>`, et leurs `connections` désignent des fichiers de `platform/connections/` (par exemple `keycloak-db`, la base de données de Keycloak), dont le `secretRef` désigne un secret du namespace du composant. Une couche démarre lorsque tous les composants de la couche non vide précédente sont prêts.

## Couches de valeurs

Chaque release reçoit trois couches, dans cet ordre exact avec les deux moteurs (les couches suivantes l'emportent, les maps sont fusionnées, les listes sont remplacées, comme avec `helm -f a -f b`) :

1. `platform/platform-values.yaml`
2. chaque fichier de connexion listé dans `connections`, dans cet ordre : `projects/<project>/connections/<name>.yaml` pour un service, `platform/connections/<name>.yaml` pour un composant de plateforme
3. le `values.yaml` de l'instance

## Fichiers générés (Flux)

Flux a besoin d'une `HelmRelease` et d'un `OCIRepository` par instance, et de ConfigMaps portant les couches de valeurs. Ils sont générés à partir des fichiers `instance.yaml` par `scripts/render-flux.sh` (bash 4+ et yq v4), jamais modifiés à la main : `helmrelease.yaml` et `kustomization.yaml` de chaque instance, `projects/<project>/kustomization.yaml`, et, pour les administrateurs de la plateforme, `platform/kustomization.yaml` et `flux/components.yaml`. La console écrit des fichiers identiques à l'octet près, et `scripts/check.sh` échoue lorsqu'un fichier généré n'est pas à jour. Argo CD ignore ces fichiers.

Les HelmReleases, les OCIRepositories et les ConfigMaps de valeurs (`okdp-platform-values`, `conn-<project>-<name>`, `okdp-platform-conn-<name>`, `values-<project>-<instance>`) se trouvent tous dans le namespace `okdp-releases` ; la release Helm est stockée dans le namespace cible.

## Installer avec Flux

Prérequis : Flux (testé avec la v2.9.5 : source-controller, kustomize-controller, helm-controller), un serveur Git hébergeant le dépôt, et helm-controller démarré avec `--feature-gates=DisableChartDigestTracking=true`. Sans cette option, helm-controller ajoute le digest OCI à la version de la charte, si bien que la version vue par les templates diffère de celle d'Argo CD :

```sh
kubectl -n flux-system patch deployment helm-controller --type json \
  -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--feature-gates=DisableChartDigestTracking=true"}]'
```

Avec `flux bootstrap`, ajoutez le même patch à `flux-system/kustomization.yaml`.

1. Renseignez l'URL et la branche Git dans `flux/sync.yaml` (ainsi que les chemins `./gitops/...` dans `flux/sync.yaml` et `flux/platform.yaml` si l'organisation ne se trouve pas sous `gitops/` ; générez alors avec `scripts/render-flux.sh --path-prefix <prefix>`). Faites un commit.
2. `kubectl apply -f gitops/flux/sync.yaml`

Flux gère ensuite `flux/` lui-même, les valeurs de la plateforme, les composants de plateforme (dans l'ordre des couches) et tous les projets. Supprimer le répertoire d'une instance désinstalle la release.

## Installer avec Argo CD

Prérequis : Argo CD (testé avec la v3.4.2) avec les synchronisations progressives activées sur le contrôleur ApplicationSet, nécessaires à l'ordre des couches des composants de plateforme :

```sh
kubectl -n argocd patch configmap argocd-cmd-params-cm --type merge \
  -p '{"data":{"applicationsetcontroller.enable.progressive.syncs":"true"}}'
kubectl -n argocd rollout restart deployment argocd-applicationset-controller
```

1. Renseignez l'URL et la branche Git dans `argocd/root.yaml`, `argocd/platform-values.yaml`, `argocd/components.yaml` et `argocd/services.yaml` (ainsi que le préfixe `gitops/` des chemins si l'organisation se trouve ailleurs). Faites un commit.
2. `kubectl apply -n argocd -f gitops/argocd/`

Argo CD gère ensuite `argocd/` lui-même, les valeurs de la plateforme, et une Application `<project>-<instance>` par composant de plateforme et par instance de service. Chaque Application a deux sources, la charte OCI et le dépôt Git, et liste les couches de valeurs dans l'ordre ci-dessus. Supprimer le répertoire d'une instance supprime l'Application et ses ressources.

## Ce qu'écrit la console

| Action dans la console  | Fichiers écrits                                                                                              | Fichiers supprimés          |
| ----------------------- | ------------------------------------------------------------------------------------------------------------ | --------------------------- |
| Déployer une instance   | `instance.yaml`, `values.yaml`, `helmrelease.yaml`, `kustomization.yaml` ; `projects/<p>/kustomization.yaml` |                             |
| Modifier les paramètres | `values.yaml`                                                                                                |                             |
| Changer de version      | `instance.yaml`, `helmrelease.yaml`                                                                          |                             |
| Supprimer une instance  | `projects/<p>/kustomization.yaml`                                                                            | le répertoire de l'instance |
| Créer une connexion     | `projects/<p>/connections/<c>.yaml`, `projects/<p>/kustomization.yaml`                                       |                             |
| Supprimer une connexion | `projects/<p>/kustomization.yaml`                                                                            | le fichier de connexion     |
| Créer un projet         | `project.yaml`, `projects/<p>/kustomization.yaml`                                                            |                             |
| Supprimer un projet     |                                                                                                              | `projects/<p>/`             |
| Modifier le catalogue   | `platform/catalog.yaml`                                                                                      |                             |

Chaque modification est un commit, `okdp: <action> <project>/<instance> by <user>`. La console n'écrit jamais les composants de plateforme.
