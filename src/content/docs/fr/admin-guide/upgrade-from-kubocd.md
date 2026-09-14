---
title: Mise à niveau depuis KuboCD
description: Passer une plateforme OKDP déployée avec KuboCD à des chartes Helm décrites dans un dépôt Git de déploiement, réconcilié par Flux ou Argo CD.
---

Les versions précédentes d'OKDP déployaient leurs services avec [KuboCD](https://www.kubocd.io/) : une `Release` KuboCD par instance, un `Context` pour les valeurs de la plateforme et le catalogue de services, des objets `ClusterContract` et `Connection` pour les connexions. OKDP déploie désormais des chartes Helm décrites par des fichiers d'un [dépôt Git de déploiement](/fr/installation-requirements/deployments-repository), réconcilié par Flux ou Argo CD. Cette page liste ce qui change et l'ordre des opérations.

Il n'y a pas de mise à niveau sur place : chaque release Helm et la plupart des noms de ressources changent, si bien que chaque instance est installée de nouveau et démarre avec de nouveaux volumes. Les données conservées en dehors de l'instance (compartiments du stockage S3, bases de données atteintes par une connexion) restent où elles sont ; les données conservées dans les volumes d'une instance doivent être déplacées à la main.

Cette page couvre les services des projets. Les composants de plateforme (cert-manager, External Secrets Operator, CloudNativePG, Keycloak, etc.) sont eux aussi renommés, et plusieurs passent à de nouvelles versions majeures ; le plus simple est de les installer sur un nouveau cluster à partir de l'organisation de référence d'[OKDP Sandbox](https://github.com/OKDP/okdp-sandbox). Pour les mettre à niveau sur place, voir [Composants de plateforme](#composants-de-plateforme).

## Des objets KuboCD aux fichiers

| KuboCD                                                             | Désormais                                                                                                           |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------- |
| `Context` (valeurs de la plateforme)                               | `platform/platform-values.yaml`, sous `global.okdp`, mêmes clés : `.Context.X` devient `.Values.global.okdp.X`      |
| `serviceCatalog` du `Context`                                      | `platform/catalog.yaml`, `defaultRepository: oci://quay.io/okdp/platform-charts`                                    |
| `Release` `<project>-<instance>`                                   | `projects/<project>/services/<instance>/instance.yaml` et `values.yaml` (l'ancien `spec.parameters`)                |
| paquet `quay.io/okdp/platform-packages/<service>`, tag `4.0.1-p02` | charte `oci://quay.io/okdp/platform-charts/<service>`, version `4.0.1-1.0.1`                                        |
| objet `Connection`                                                 | `projects/<project>/connections/<name>.yaml` et un secret ; `secretRef: <name>` devient `secretRef: {name: <name>}` |
| connexion publiée par une release, `kcd-<release>-<output>`        | le nom de release `<project>-<instance>`                                                                            |
| `ClusterContract`                                                  | schémas de contrat de la charte `okdp-lib`                                                                          |
| état de la `Release`                                               | `HelmRelease` Flux ou `Application` Argo CD, et le ConfigMap descripteur `<release>-okdp`                           |
| `protected: true`, `kubocd-webhooks`                               | libellé `okdp.io/protected: "true"`, appliqué par les ValidatingAdmissionPolicies du composant `tools`              |
| dépendances entre releases                                         | aucune entre services ; couches `00`, `10`, `20`, `30` entre composants de plateforme                               |

## Ressources renommées

La release Helm d'une instance est `<project>-<instance>` (par exemple `demo-trino`), et il y a une release par instance au lieu d'une par module KuboCD : le suffixe `-main` disparaît de tous les noms.

- Services : Trino `<release>-trino-*` (auparavant `<release>-main-trino-*`), Hive Metastore `<release>-hive-metastore` (auparavant `<release>`), Polaris `<release>-polaris` (auparavant `<release>-main`), Spark History Server `<release>-spark-history-server` (auparavant `<release>-main`), Ollama `<release>` (auparavant `<release>-main`), Open WebUI `<release>-open-webui` (auparavant `<release>-main-open-webui`), cache et broker Celery de Superset, désormais Valkey, Service `<release>-redis` (auparavant le Redis `<release>-main-redis-headless`).
- Secrets générés : `creds-<release>-internal` devient `<release>-internal` (Trino, Airflow, Superset), et `creds-<release>-root` devient `<release>-root` (Polaris). Les secrets des clients OAuth gardent leur nom, `creds-<release>-oauth2`.
- Secrets TLS : `airflow-tls` devient `<release>-airflow-tls`, `polaris-console-tls` devient `<release>-polaris-console-tls`, et `okdp-ui-tls` de la console devient `<release>-tls`.
- Les persistent volume claims suivent le nom de la release : les nouvelles instances démarrent avec des volumes vides.
- Bases de données CloudNativePG du composant `cnpg-postgresql` : les objets `Database` sont `<release>-<database>` (auparavant `<database>`), et leurs connexions `<release>-<database>` (auparavant `kcd-<release>-<database>`).

## Données à déplacer à la main

- Ollama : les modèles sont téléchargés de nouveau.
- Open WebUI (`ollama-ui`) : les conversations et les comptes repartent de zéro ; copiez l'ancien volume de données pour les conserver.
- RustFS : son Service, ses volumes et ses secrets changent de nom ; copiez les données du stockage.
- Superset : le cache et le broker Celery sont désormais [Valkey](https://valkey.io/) (auparavant Redis avec un volume), inclus dans la charte et sans persistance : le cache et la file Celery repartent vides. Les Deployments de Superset sélectionnent aussi leurs pods sur de nouveaux libellés (charte Apache Superset 0.22.8) : ils sont recréés, et non mis à jour sur place, et leurs pods redémarrent.

## Utilisateurs et clients OIDC

- JupyterHub : le secret des cookies et les clés d'état d'authentification sont générés de nouveau par ESO (secret `<release>-hub-generated`) : les utilisateurs doivent se reconnecter.
- Polaris : les identifiants root sont générés dans `<release>-root` et conservés lorsque l'instance est supprimée. Un domaine déjà initialisé dans la base de données connaît les anciens identifiants de `creds-<release>-root` : gardez une copie de ce secret.

## kubauth est retiré

OKDP ne fournit plus kubauth : l'identité repose uniquement sur Keycloak. `clientProvisioning` accepte `existing` (clients créés au préalable dans Keycloak) ou `dcr` (enregistrement dynamique des clients) ; `kubauth` n'est plus une valeur valide.

Une plateforme qui utilise kubauth doit déplacer ses utilisateurs, ses groupes et ses clients OIDC vers Keycloak, et passer `clientProvisioning` à `existing` ou `dcr`, **avant** la mise à niveau. Aucun outil de migration n'est fourni : recréez-les dans Keycloak à la main ou avec vos propres scripts. Les utilisateurs se reconnectent ensuite, et les services reçoivent de nouveaux secrets clients. Une fois les utilisateurs et les groupes dans Keycloak, la page Identity de la console les gère de nouveau, à condition que le serveur du control plane dispose d'identifiants d'administration Keycloak (voir [Fournisseur d’identité](/fr/installation-requirements#fournisseur-didentité)).

Avec `dcr`, le client OIDC d'un service est enregistré dans Keycloak par un Job de son chart. La suppression du service ne retire pas ce client de Keycloak, avec Flux comme avec Argo CD : supprimez-le dans la console Keycloak.

## Changements de paramètres

- Les références à d'autres instances utilisent le nom de release : `kcd-demo-hive-metastore` devient `demo-hive`, `kcd-demo-polaris-catalog` devient `demo-polaris`, `kcd-demo-trino-endpoint` devient `demo-trino`. Trino a besoin de `warehouse` sur un catalogue d'une instance Polaris du projet.
- JupyterHub : le paramètre `connections` (connexions PySpark) est renommé `sparkConnections`, car `connections` contient désormais les connexions externes de la release.
- Les paramètres ne sont plus des templates. L'échappement KuboCD ``{{ `{{ ... }}` }}`` n'est plus nécessaire : écrivez la valeur telle qu'elle doit parvenir au service. JupyterHub remplace toujours `{{ .Context.ingress.suffix }}` (ou `{{ .Values.global.okdp.ingress.suffix }}`), `{{ .Release.Name }}` et `{{ .Release.Namespace }}` dans `welcomeNotebook`, `sparkDefaults` et les `properties` de `sparkConnections`.
- Spark History Server : OIDC est activé sauf si `global.okdp.oidc.enabled` vaut `false` (le paquet valait `false` par défaut lorsque la clé manquait).
- Serveur du plan de contrôle : `kubocdNamespace`, `releaseInterval` et `releaseTimeout` disparaissent ; les réglages Git sont sous `gitops` (URL du dépôt, branche, chemin, moteur, secret d'identifiants).

## Composants de plateforme

Les composants de l'organisation de référence utilisent des versions amont plus récentes que les paquets KuboCD. Si vous gardez le cluster et les mettez à niveau sur place au lieu d'en installer un nouveau, suivez les chemins de mise à niveau amont ci-dessous ; ne sautez pas directement aux nouvelles versions.

### External Secrets Operator

ESO passe de la 0.15 à la 2.11, et l'API de `external-secrets.io/v1beta1` à `external-secrets.io/v1` : ESO 2.x ne sert plus `v1beta1`, et toutes les chartes OKDP produisent désormais du `v1` (les générateurs `Password` restent en `generators.external-secrets.io/v1alpha1`).

1. Mettez ESO à niveau de la 0.15 vers la 0.16, la version qui sert à la fois `v1` et `v1beta1`.
2. Passez tous les manifestes ESO en `v1` : les chartes OKDP le font d'elles-mêmes, vos propres manifestes `SecretStore`, `ClusterSecretStore` et `ExternalSecret` (et tout ce qui les crée) doivent être modifiés. Les objets déjà stockés en `v1beta1` doivent être réécrits en `v1` (par exemple par une mise à jour de chaque objet) avant de retirer `v1beta1` du `status.storedVersions` des CRD d'ESO.
3. Mettez ESO à niveau vers la 2.x (l'organisation de référence installe la charte amont 2.11.0). ESO 2.0 a retiré les fournisseurs Alibaba et Device42.

### cert-manager

cert-manager passe de la 1.17 à la 1.21 (et trust-manager de la 0.16 à la 0.25). Montez d'une version mineure à la fois, 1.17 → 1.18 → 1.19 → 1.20 → 1.21, comme le recommande le projet cert-manager, et lisez les notes de chaque version : depuis la 1.18, `Certificate.spec.privateKey.rotationPolicy` vaut `Always` par défaut (une nouvelle clé privée à chaque renouvellement).

### Keycloak

Le composant Keycloak passe de la charte et de l'image Bitnami à la charte codecentric `keycloakx` avec l'image officielle (`quay.io/keycloak/keycloak` 26.7), exécutée en mode production. Il garde sa base de données : la même connexion `database-server`, Keycloak migre le schéma à son démarrage. Le sélecteur du StatefulSet diffère, la release ne peut donc pas être mise à niveau sur place : supprimez d'abord le StatefulSet Bitnami de la release Keycloak (les données sont dans la base), puis laissez le moteur installer la nouvelle charte. Le realm est appliqué par keycloak-config-cli, qui supprime les attributions de rôles et les réglages du profil utilisateur que le fichier du realm ne déclare pas : déclarez dans les valeurs du composant ce qui avait été accordé à la main (par exemple les rôles du compte de service de la console et les attributs utilisateur `comment` et `uid`).

### Ingress NGINX

ingress-nginx est arrêté en amont : la 4.15.1, qu'installe l'organisation de référence, est sa dernière version, et aucun correctif, y compris de sécurité, ne suivra. La migration vers un autre contrôleur d'ingress est un travail futur ; d'ici là, n'exposez pas l'ingress à des utilisateurs non fiables.

## Ordre des opérations

1. **Prérequis.** Vérifiez que l'External Secrets Operator 2.x fonctionne (API `external-secrets.io/v1`, voir [External Secrets Operator](#external-secrets-operator) pour le chemin depuis la 0.15), et préparez le moteur GitOps : Flux, sur lequel KuboCD fonctionne déjà, avec helm-controller démarré avec `--feature-gates=DisableChartDigestTracking=true`, ou Argo CD avec les synchronisations progressives activées. Kubernetes 1.30 ou ultérieur. Voir [Dépôt de déploiement](/fr/installation-requirements/deployments-repository).
2. **Valeurs de la plateforme.** Créez le dépôt de déploiement à partir de l'organisation de référence. Déplacez les valeurs du `Context` dans `platform/platform-values.yaml` sous `global.okdp` (mêmes clés), et son `serviceCatalog` dans `platform/catalog.yaml`, avec les versions des chartes.
3. **Fichiers des instances.** Convertissez chaque `Connection` en `projects/<project>/connections/<name>.yaml` (le contrat et les champs de `spec.values`, avec `secretRef: {name: ...}`), et chaque `Release` en `projects/<project>/services/<instance>/` : `instance.yaml` (le nom d'instance du libellé `okdp.io/instance-name`, le projet du namespace, le service du libellé `okdp.io/service`, la charte et sa version, les fichiers de connexion utilisés) et `values.yaml` (`spec.parameters`, avec les changements ci-dessus). Avec Flux, exécutez `scripts/render-flux.sh` ; puis `scripts/check.sh`.
4. **Sauvegardez** les données listées ci-dessus.
5. **Retirez KuboCD.** Supprimez les objets `Release` KuboCD, ce qui désinstalle leurs releases Helm, puis les objets `Connection`, `Context` et `ClusterContract`, le contrôleur KuboCD et ses CRD, et le composant `kubocd-webhooks`. Les anciennes et les nouvelles instances ne peuvent pas fonctionner côte à côte : elles servent les mêmes hôtes d'ingress.
6. **Démarrez le moteur** sur le dépôt de déploiement : `kubectl apply -f gitops/flux/sync.yaml`, ou `kubectl apply -n argocd -f gitops/argocd/`.
7. **Restaurez** les données, et prévenez les utilisateurs qu'ils doivent se reconnecter.
