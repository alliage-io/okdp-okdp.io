---
title: Prérequis d'installation
description: Lien vers le dépôt OKDP Sandbox pour une installation rapide et la description des composants prérequis à l'installation d'un cluster OKDP.
---

[OKDP Sandbox](https://github.com/OKDP/okdp-sandbox) est une installation rapide d'un cluster OKDP. En suivant les instructions du fichier README.md, vous devriez avoir accès à tous les services OKDP actuels.

Le guide d'installation suivant explique les dépendances obligatoires dont OKDP v1 a besoin pour disposer d'un cluster opérationnel.

## Objets KuboCD

[Kubocd](https://www.kubocd.io/) est une couche superposée à [Flux](https://fluxcd.io/) permettant de gérer les objets `helmRelease` liés aux [chartes helm](https://helm.sh/). Flux doit être installé avant le CRD KuboCD pour que ce dernier fonctionne.

### Version KuboCD

Une version KuboCD contient un ou plusieurs chartes Helm avec, si nécessaire, un ordre de dépendances entre eux, et remplace dynamiquement leur fichier `values.yaml` par des variables définies dans un objet `context` et/ou `clusterContract`.

### Contexte

Un contexte est un objet Kubernetes appartenant à l’environnement KuboCD. Il s’agit essentiellement d’un ensemble de variables définies auxquelles les versions KuboCD ont accès. L’objectif est de centraliser les variables utilisées dans plusieurs versions et de définir l’environnement OKDP, comme le suffixe d’ingress, l’adresse du proxy HTTP/HTTPS, etc. Le contexte contient également le catalogue de services.

### Contrat de cluster

Un contrat de cluster (`clusterContract` avec la commande kubectl) est également un objet Kubernetes appartenant à l’environnement KuboCD, qui définit des variables qui, contrairement au contexte, sont spécifiques à un service ou à un composant.

## Dépendances principales obligatoires

### Stockage

OKDP ne fournit pas de solution de stockage ; il est donc nécessaire de disposer au préalable d'un fournisseur de stockage compatible S3, tel que [SeaweedFS](https://github.com/seaweedfs/seaweedfs) ou [Ceph](https://ceph.io/en/), pour pouvoir utiliser la plateforme. OKDP communique avec le fournisseur de stockage via l’API S3. Tout fournisseur de stockage S3 prenant également en charge [STS](https://docs.aws.amazon.com/STS/latest/APIReference/Welcome.html), puisque Polaris l’utilise pour sécuriser ses catalogues, devrait être compatible avec OKDP ; ici, à des fins de démonstration, nous utilisons SeaweedFS.

Un contrôleur de serveur de base de données tel que [CloudNativePG](https://cloudnative-pg.io/) doit également être fourni. Un contrôleur de serveur de base de données est un opérateur Kubernetes qui vous permet de gérer un `cluster` de base de données afin de gérer le cycle de vie des objets `database` de manière native à Kubernetes.

Une fois que le fournisseur de stockage et le contrôleur de serveur de base de données sont en place, OKDP communique avec eux via les connexions décrites dans la section [Gestion des connexions et des secrets externes](/fr/admin-guide/connections-and-external-secrets-management).

La procédure à suivre pour créer une vue de l’interface utilisateur du fournisseur de stockage est décrite dans le [guide utilisateur](/fr/user-guide#project-panel).

Le fournisseur de stockage ainsi que le contrôleur de serveur de base de données doivent être fournis via un paquet kubocd dont les métadonnées comportent les balises `storage` ou `database-server`. Dans le cas contraire, certains services qui en dépendent ne pourront pas être déployés.

### Contrôleur d'ingress

Un [contrôleur d'ingress](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) doit être installé pour accéder aux interfaces utilisateur de tous les services et au plan de contrôle d'OKDP lui-même. Pour la version v1 d'OKDP, Nginx est utilisé, bien qu'il soit obsolète ; toutefois, une migration vers d'autres contrôleurs d'ingress sera effectuée ultérieurement.

### Fournisseur d'identité

OKDP fonctionne actuellement et a été testé avec [Keycloak](https://www.keycloak.org/) et [KubeAuth](https://www.kubeauth.io/) pour l'authentification OIDC. Veuillez vous reporter à leur documentation pour plus d'informations.

## Autres dépendances

OKDP Sandbox dépend actuellement d’autres composants tels qu’un serveur DNS, un gestionnaire de certificats et d’autres outils, que vous pouvez consulter sur la page [stack](/fr/stack/okdp-1-0).
