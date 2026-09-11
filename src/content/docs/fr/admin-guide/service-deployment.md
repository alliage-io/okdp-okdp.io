---
title: Déploiement de services
description: Déploiement de services à l'aide du plan de contrôle OKDP et de fichiers YAML.
---

Le plan de contrôle OKDP permet à l’administrateur de déployer, de supprimer et de surveiller des services, ce qui constitue son objectif principal.

Dans OKDP, un service est une [version KuboCD](/fr/installation-requirements#version-kubocd) hébergée sur un référentiel externe défini dans votre contexte et contenant un `helmRelease` comme couche sous-jacente.

Le déploiement d’un service dans l’interface utilisateur suit toujours le même schéma. Sur la page d’instance du service, cliquez sur `Deploy`, puis suivez les 3 étapes suivantes :

1. Choisissez le nom de l’instance et la version du paquet KuboCD pour celle-ci.
2. Configurez les paramètres de l’instance.
3. Validez le récapitulatif de la configuration.
4. Vous revenez à la page d’instance du service, où vous pouvez vérifier l’état de l’instance.
5. En cliquant sur la petite icône en forme d’œil à droite de la ligne correspondant à votre instance, vous êtes alors redirigé vers une page présentant tous les détails de l’instance, ainsi que ses pods déployés et leur état ; vous pouvez les consulter en cliquant sur le bouton `Logs`.

Un exemple complet est fourni avec le premier service mentionné, Hive Metastore. Pour les autres services, à l’exception de Trino, la configuration de déploiement est fournie au format YAML et peut être exécutée à l’aide de la commande `kubectl apply`.

Chaque instance d’un service peut être modifiée en cliquant sur le petit crayon situé à droite de la ligne de l’instance sur la page dédiée, ou supprimée définitivement en cliquant sur la corbeille.

Remarque : pour modifier la version du paquet KuboCD, vous devez supprimer l’instance actuelle et en déployer une nouvelle.

### Ordre et dépendances des services

Il existe un ordre de dépendance entre les versions de KuboCD représentant un service ou un composant externe, auquel l'administrateur doit prêter attention. Par exemple, pour être déployés, presque tous les services ont besoin soit d'un fournisseur de stockage, soit d'un serveur de base de données, soit des deux, fournis sous forme de package KuboCD sous les noms de dépendance `storage` et `database-server`, indiqués dans leurs métadonnées.

### Déploiement de Hive Metastore

Hive Metastore nécessite une connexion à un fournisseur de stockage S3 et à un serveur de base de données. Si aucun de ces éléments n'est présent, le déploiement ne sera pas possible.

Dans le cas contraire, le déploiement d'une instance Hive Metastore est très simple. Dans le `Data Catalog`, cliquez sur `hive-metastore`, puis sur `+ Déploy`. Choisissez un nom d’instance et une version du package KuboCD de Hive Metastore, puis cliquez sur `Next`.

![Hive1](../../assets/hive1.png)

Sélectionnez ensuite une connexion au serveur de base de données et une connexion au fournisseur de stockage S3. S’ils n’existent pas, vous pouvez les créer en cliquant sur `+ New connection` et en suivant les étapes décrites dans [Gestion des connexions et des secrets externes](/fr/admin-guide/connections-and-external-secrets-management#création-de-connexions). Saisissez les quantités de ressources et choisissez un secret Kubernetes pour l’authentification du stockage S3, ainsi que, de manière facultative, un compartiment (bucket) qui limite le Hive Metastore. Une fois cela fait, cliquez sur `CREATE`.

![Hive2](../../assets/hive2.png)

Vérifiez maintenant que la configuration correspond à vos attentes, puis cliquez sur `Déploy instance`.

![Hive3](../../assets/hive3.png)

Attendez que le `Status` passe à `Ready`.

![Hive4](../../assets/hive4.png)

En cliquant sur la petite icône en forme d’œil à droite, vous accédez aux détails de l’instance qui affichent les pods déployés ainsi que leur statut, et vous pouvez consulter leurs journaux.

![Hive5](../../assets/hive5.png)

En cliquant sur la petite icône en forme de crayon ou sur l’icône de la poubelle, vous pouvez modifier ou supprimer l’instance.

Voici le même déploiement que celui présenté dans les captures d’écran, au format YAML.

```yaml
apiVersion: kubocd.kubotal.io/v1alpha1
kind: Release
metadata:
  name: demo-hive
  namespace: demo
  labels:
    okdp.io/project: demo
    okdp.io/service: hive-metastore
    okdp.io/instance-name: hive
spec:
  description: Hive metastore of the demo project
  package:
    repository: quay.io/okdp/platform-packages/hive-metastore
    tag: 4.0.1-p02
    interval: 30m
    timeout: 10m
  parameters:
    db: demo-db-hive
    storage: demo-storage
    s3SecretRef: creds-hive-metastore-s3
    warehouseBucket: hive
  targetNamespace: demo
```

### Déploiement de Polaris

Polaris nécessite une connexion et les identifiants d'accès au stockage S3, ainsi qu'une connexion à une base de données SQL. De plus, au moins un domaine doit être configuré.

Notez que l'entité « principal » de Polaris est utilisée via l'API de gestion. Polaris n'authentifie pas les clients Keycloak ; il gère ses propres entités « principal ».

```yaml
apiVersion: kubocd.kubotal.io/v1alpha1
kind: Release
metadata:
  name: demo-polaris
  namespace: demo
  labels:
    okdp.io/project: demo
    okdp.io/service: polaris
    okdp.io/instance-name: polaris
spec:
  description: Polaris Iceberg catalog of the demo project
  package:
    repository: quay.io/okdp/platform-packages/polaris
    tag: 1.3.0-incubating-p06
    interval: 30m
    timeout: 10m
  parameters:
    db: demo-db-polaris
    storage: demo-storage
    s3SecretRef: creds-polaris-s3
    realms:
      - name: sandbox
        rootPrincipal:
          name: root
          credentialsSecret:
            name: creds-polaris-root-okdp-sandbox
            clientIdKey: client_id
            clientSecretKey: client_secret
        principals:
          - name: service-account-svc-polaris-api-admin
            clientId: svc-polaris-api-admin
            credentialsSecret:
              name: creds-polaris-oauth2-admin-okdp-sandbox
              clientIdKey: client_id
              clientSecretKey: client_secret
            roles: [service_admin, catalog_admin]
  targetNamespace: demo
```

### Déploiement de Trino

Avant de déployer Trino, assurez-vous qu’une instance de Hive Metastore et/ou une instance de Polaris est déployée.

Vous avez la possibilité d’ajouter [OPA (Open Policy Agent)](https://www.openpolicyagent.org) en tant que composant RBAC pour Trino. Cochez la case OPA pour créer un pod contenant un conteneur de serveur OPA et un conteneur de gestion Kube permettant de lire les configmaps au sein du namespace du projet. Si, en outre, vous choisissez [OPAL (Open Poly Agent Layer)](https://docs.opal.ac), le pod OPA ne contient qu’un serveur OPA et aucun conteneur Kubemanagement. Les politiques sont alors synchronisées via OPAL, composé des pods suivants :

- Serveur OPAL
- Client OPAL
- Base de données Postgres

La synchronisation des politiques s'effectue dans ce cas à partir du dépôt Git saisi dans le champ obligatoire. Si seule la case OPAL est cochée, ni OPA ni OPAL ne sont déployés.

![Trino](../../assets/trino.png)

Vous pouvez ajouter des catalogues Polaris et choisir le dimensionnement, c'est-à-dire le nombre de workers et leur consommation maximale de ressources.

Lorsque vous cliquez ensuite sur `Next`, tous les paramètres sont récapitulés. Si vous êtes d'accord, cliquez sur `Déploy instance`.

Voici un exemple de déploiement avec uniquement OPA et sans OPAL :

```yaml
apiVersion: kubocd.kubotal.io/v1alpha1
kind: Release
metadata:
  name: demo-trino
  namespace: demo
  labels:
    okdp.io/project: demo
    okdp.io/service: trino
    okdp.io/instance-name: trino
spec:
  description: Trino of the demo project, bronze plus Iceberg catalogs
  package:
    repository: quay.io/okdp/platform-packages/trino
    tag: 480.0.0-p21
    interval: 30m
    timeout: 10m
  parameters:
    s3SecretRef: creds-trino-s3
    hiveCatalogs:
      - name: bronze
        metastore: kcd-demo-hive-metastore
        storage: demo-storage
    icebergCatalogs:
      - name: iceberg
        catalog: kcd-demo-polaris-catalog
        storage: demo-storage
        warehouse: demo
        oidcSecretRef: creds-polaris-root-okdp-sandbox
      - name: silver
        catalog: kcd-demo-polaris-catalog
        storage: demo-storage
        warehouse: silver
        oidcSecretRef: creds-polaris-root-okdp-sandbox
      - name: gold
        catalog: kcd-demo-polaris-catalog
        storage: demo-storage
        warehouse: gold
        oidcSecretRef: creds-polaris-root-okdp-sandbox
    enableOPA: true
  targetNamespace: demo
```

### Déploiement de Spark History Server

Spark History Server a uniquement besoin d'accéder au fournisseur de stockage S3. Il vous suffit donc de fournir les informations de connexion S3. Rédigez au format JSON le mappage OIDC permettant l'accès à Spark History Server.

Voici un exemple au format YAML :

```yaml
apiVersion: kubocd.kubotal.io/v1alpha1
kind: Release
metadata:
  name: demo-spark-history
  namespace: demo
  labels:
    okdp.io/project: demo
    okdp.io/service: spark-history-server
    okdp.io/instance-name: spark-history
spec:
  description: Spark History Server of the demo project
  package:
    repository: quay.io/okdp/platform-packages/spark-history-server
    tag: 3.5.1-p07
    interval: 30m
    timeout: 10m
  parameters:
    storage: demo-storage
    s3SecretRef: creds-spark-history-s3
    roleMapping:
      admin_groups: [platform_admin]
      history_admin_groups: [platform_admin, auditor]
      modify_groups: [platform_admin, data_engineer]
      view_groups:
        [
          platform_admin,
          data_engineer,
          data_scientist,
          data_steward,
          business_analyst,
        ]
  targetNamespace: demo
```

### Déploiement de JupyterHub

La configuration minimale requise pour JupyterHub comprend une connexion au stockage S3, un secret d'authentification pour le stockage S3 et un mappage de rôles OIDC afin d'éviter une erreur 403 pour tout utilisateur tentant d'y accéder. Voici la configuration minimale au format YAML :

```yaml
apiVersion: kubocd.kubotal.io/v1alpha1
kind: Release
metadata:
  labels:
    okdp.io/instance-name: jupyterhub
    okdp.io/project: demo
    okdp.io/service: jupyterhub
  name: demo-jupyterhub
  namespace: demo
spec:
  description: jupyterhub for project demo
  package:
    interval: 30m0s
    repository: quay.io/okdp/platform-packages/jupyterhub
    tag: 4.3.3-p06
    timeout: 10m0s
  parameters:
    cpu: 0.5
    memoryGi: 1
    oidcRoleMapping:
      admin_groups: platform_admin
      allowed_groups: platform_admin
    s3SecretRef: creds-jupyterhub-s3
    storage: demo-storage
  targetNamespace: demo
```

Avec cette configuration minimale, l'accès aux compartiments S3 est illimité, PySpark ne peut accéder à aucun catalogue Polaris et aucun notebook de bienvenue n'est inclus. À l'exception du notebook de bienvenue, ces paramètres peuvent tous être modifiés pour des déploiements JupyterHub plus avancés.

Voici un exemple montrant comment limiter l'accès de JupyterHub à certains compartiments (buckets) :

```yaml
spec:
  parameters:
    fileBrowserLocations:
      - { name: bronze, uri: s3://bronze }
      - { name: silver, uri: s3://silver }
      - { name: gold, uri: s3://gold }
```

Voici un exemple illustrant comment autoriser PySpark à accéder à un catalogue Polaris :

```yaml
spec:
  parameters:
    pyspark: [silver]
    connections:
      - name: silver
        properties: |
          spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions
          spark.sql.catalog.silver=org.apache.iceberg.spark.SparkCatalog
          spark.sql.catalog.silver.type=rest
          spark.sql.catalog.silver.warehouse=silver
          spark.sql.catalog.silver.uri=https://polaris-demo.{{ .Context.ingress.suffix }}/api/catalog
          spark.sql.catalog.silver.oauth2-server-uri=https://keycloak.{{ .Context.ingress.suffix }}/realms/master/protocol/openid-connect/token
          spark.sql.catalog.silver.scope=profile
          spark.sql.catalog.silver.rest.auth.type=oauth2
          spark.sql.catalog.silver.token-refresh-enabled=true
          spark.sql.catalog.silver.header.X-Iceberg-Access-Delegation=vended-credentials
          spark.sql.catalog.silver.io-impl=org.apache.iceberg.io.ResolvingFileIO
          spark.sql.catalog.silver.header.Polaris-Realm=sandbox
          spark.sql.catalog.silver.client.region=us-east-1
          spark.sql.catalog.silver.s3.region=us-east-1
```

Un notebook de bienvenue est défini sous la clé `spec.parameters.welcomeNotebook` au format JSON.

### Déploiement de Superset

Superset nécessite deux connexions au serveur de base de données : l’une vers la base de données contenant les données, et l’autre vers celle contenant les métadonnées. En l’absence de mappage OIDC, tous les utilisateurs sont classés dans la catégorie « Public », et Superset les rejette tous.

Voici un exemple de déploiement de Superset au format YAML :

```yaml
apiVersion: kubocd.kubotal.io/v1alpha1
kind: Release
metadata:
  name: demo-superset
  namespace: demo
  labels:
    okdp.io/project: demo
    okdp.io/service: superset
    okdp.io/instance-name: superset
spec:
  description: Superset of the demo project
  package:
    repository: quay.io/okdp/platform-packages/superset
    tag: 6.0.0-p04
    interval: 30m
    timeout: 10m
  parameters:
    metadataDb: demo-db-superset
    examplesDb: demo-db-superset-examples
    oidcRoleMapping:
      platform_admin: [Admin]
      data_engineer: [Alpha, sql_lab]
      data_scientist: [Alpha, sql_lab]
      business_analyst: [Gamma]
      data_steward: [Gamma, sql_lab]
      auditor: [Gamma]
    datasources:
      - { name: trino-bronze, trino: kcd-demo-trino-endpoint, catalog: bronze }
      - { name: trino-silver, trino: kcd-demo-trino-endpoint, catalog: silver }
      - { name: trino-gold, trino: kcd-demo-trino-endpoint, catalog: gold }
  targetNamespace: demo
```
