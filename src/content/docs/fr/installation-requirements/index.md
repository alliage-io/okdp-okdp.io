---
title: Prérequis d'installation
description: Lien vers le dépôt OKDP Sandbox pour une installation rapide, le modèle de déploiement d'OKDP et les composants nécessaires à un cluster OKDP.
sidebar:
  order: 1
---

[OKDP Sandbox](https://github.com/OKDP/okdp-sandbox) est une installation rapide d'un cluster OKDP. En suivant les instructions du fichier README.md, vous devriez avoir accès à tous les services OKDP actuels.

Le guide d'installation suivant explique comment OKDP déploie ses services et les dépendances obligatoires dont il a besoin pour disposer d'un cluster opérationnel.

## Modèle de déploiement

OKDP ne s'appuie que sur des outils standards : [Helm](https://helm.sh/), un moteur GitOps ([Flux](https://fluxcd.io/) ou [Argo CD](https://argo-cd.readthedocs.io/)), l'[External Secrets Operator](https://external-secrets.io/) et des objets natifs de Kubernetes. Il n'y a ni contrôleur OKDP, ni ressource personnalisée OKDP, ni base de données.

### Chartes Helm

Chaque service OKDP (Hive Metastore, Trino, JupyterHub, etc.) et chaque composant de la plateforme est une charte Helm, publiée dans `oci://quay.io/okdp/platform-charts`, `oci://quay.io/okdp/community-charts` ou `oci://quay.io/okdp/sandbox-charts`. Helm est le seul moteur de rendu : la console, Flux et Argo CD rendent tous la même charte avec les mêmes valeurs.

### Dépôt Git de déploiement

Un dépôt Git contient tout l'état souhaité de la plateforme : un répertoire par projet, un répertoire par instance de service contenant un fichier `instance.yaml` (quelle charte, quelle version) et un fichier `values.yaml` (les paramètres). Flux ou Argo CD déploie ce que décrit ce dépôt. La console OKDP écrit exactement les mêmes fichiers : déployer depuis la console ou écrire les fichiers à la main dans Git donne le même résultat.

L'organisation du dépôt et l'installation avec Flux ou Argo CD sont décrites dans [Dépôt de déploiement](/fr/installation-requirements/deployments-repository).

### Valeurs de la plateforme

Les valeurs communes à toutes les releases (suffixe d'ingress, fournisseur OIDC, proxy HTTP/HTTPS, émetteur de certificats, classes de stockage) se trouvent dans un seul fichier du dépôt de déploiement, `platform/platform-values.yaml`, sous la clé `global.okdp`. C'est la première couche de valeurs de chaque release. Le catalogue de services de la console est un fichier distinct, `platform/catalog.yaml`.

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

### Contrats et connexions

Un service accède à un autre service ou à un composant externe (stockage S3, serveur de base de données) au moyen d'une connexion. Une connexion suit un contrat, qui fixe ses champs : `database-server`, `s3`, `hive`, `iceberg-catalog` et `trino`. Les champs non secrets sont écrits dans Git ; les identifiants restent dans un secret Kubernetes. Voir [Gestion des connexions et des secrets externes](/fr/admin-guide/connections-and-external-secrets-management).

## Dépendances principales obligatoires

### Moteur GitOps et serveur Git

OKDP a besoin d'un serveur Git hébergeant le dépôt de déploiement, avec un accès en écriture pour la console, et de l'un des moteurs suivants :

- Flux (testé avec la v2.9.5), avec helm-controller démarré avec `--feature-gates=DisableChartDigestTracking=true` ;
- Argo CD (testé avec la v3.4.2), avec les synchronisations progressives activées sur le contrôleur ApplicationSet.

Ces deux réglages sont expliqués dans [Dépôt de déploiement](/fr/installation-requirements/deployments-repository).

### External Secrets Operator

L'[External Secrets Operator](https://external-secrets.io/) (ESO) est obligatoire. Les chartes OKDP ne génèrent jamais de mot de passe avec Helm : les mots de passe générés proviennent d'un générateur ESO `Password` et d'un `ExternalSecret`, et les identifiants partagés entre namespaces sont livrés par ESO. La plateforme utilise ESO 2.11 (API `external-secrets.io/v1` ; `v1beta1` n'est plus servie, les générateurs restent en `generators.external-secrets.io/v1alpha1`). La console s'en sert également pour synchroniser des secrets depuis un magasin de secret tel que Vault.

### Version de Kubernetes

Kubernetes 1.30 ou ultérieur est requis : la protection contre la suppression des objets critiques de la plateforme (libellé `okdp.io/protected: "true"`) est assurée par des ValidatingAdmissionPolicies, fournies par le composant de plateforme `tools`.

### Stockage

OKDP ne fournit pas de solution de stockage ; il est donc nécessaire de disposer au préalable d'un fournisseur de stockage compatible S3, tel que [SeaweedFS](https://github.com/seaweedfs/seaweedfs), [RustFS](https://rustfs.com/) ou [Ceph](https://ceph.io/en/), pour pouvoir utiliser la plateforme. OKDP communique avec le fournisseur de stockage via l’API S3. Tout fournisseur de stockage S3 prenant également en charge [STS](https://docs.aws.amazon.com/STS/latest/APIReference/Welcome.html), puisque Polaris l’utilise pour sécuriser ses catalogues, devrait être compatible avec OKDP ; la sandbox utilise SeaweedFS.

Un contrôleur de serveur de base de données tel que [CloudNativePG](https://cloudnative-pg.io/) doit également être fourni. Un contrôleur de serveur de base de données est un opérateur Kubernetes qui permet de gérer un `cluster` de bases de données afin de prendre en charge le cycle de vie des objets `database` de manière native dans Kubernetes.

Une fois le fournisseur de stockage et le serveur de base de données en place, les services OKDP y accèdent au moyen de connexions de contrat `s3` et `database-server`, expliquées dans [Gestion des connexions et des secrets externes](/fr/admin-guide/connections-and-external-secrets-management). Ces deux contrats sont toujours des connexions externes : un fichier de connexion du projet indique l'adresse du stockage ou de la base de données, et un secret du projet contient les identifiants. Les services qui en dépendent ne peuvent pas être déployés sans une telle connexion.

La procédure pour créer une vue de l’interface utilisateur du fournisseur de stockage est présentée dans le [guide de l’utilisateur](/fr/user-guide#panneau-de-projet).

### Contrôleur d’ingress

Un [contrôleur d’ingress](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) doit être installé pour accéder aux interfaces utilisateur de tous les services et au plan de contrôle OKDP lui-même. La sandbox utilise [ingress-nginx](https://github.com/kubernetes/ingress-nginx), bien que le projet soit arrêté : la 4.15.1 est sa dernière version et aucun correctif, y compris de sécurité, ne suivra. La migration vers un autre contrôleur d’ingress est un travail futur. La classe d'ingress utilisée par toutes les chartes est `global.okdp.ingress.className`.

### Fournisseur d’identité

OKDP fonctionne actuellement avec [Keycloak](https://www.keycloak.org/) et [KubeAuth](https://www.kubeauth.io/) pour l’authentification OIDC, et a été testé avec ces solutions. Veuillez consulter leur documentation pour plus d’informations. Le fournisseur est décrit une seule fois dans les valeurs de la plateforme (`global.okdp.oidc`) ; `clientProvisioning` indique comment les clients OIDC des services sont obtenus (secrets existants `existing`, enregistrement dynamique `dcr`, ou `kubauth`).

## Autres dépendances

La sandbox OKDP dépend actuellement d'autres composants, tels qu'un serveur DNS (CoreDNS), cert-manager et trust-manager, et d'autres outils, que vous pouvez consulter sur la page [stack](/fr/stack/okdp-1-0). Dans le dépôt de déploiement, ce sont des composants de plateforme installés par couches : `00` CRD et opérateurs, `10` infrastructure, `20` identité, stockage et bases de données, `30` plan de contrôle.
