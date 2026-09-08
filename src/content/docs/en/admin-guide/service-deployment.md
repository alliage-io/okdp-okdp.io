---
title: Service deployment
description: Service deployment using the OKDP control plane and using yaml files.
---

The OKDP control plane enables the administrator to deploy, delete, and monitor services, being its main goal.

A service in OKDP is a [KuboCD release](/en/installation/index#kubocd_release) hosted on an external repository which has been set in your context and containing a `helmRelease` as an underlying layer.

Service deployment in the user interface always follows the same pattern. On the instance page of the service, click on `Deploy` and then 3 stages follow:

1. Choose the instance name and KuboCD package version for the instance.
2. Configure the instance parameters.
3. Approve the recapitulation of the configuration.
4. You are returned to the instance page of the service, where you can check the instance status.
5. By clicking on the little eye icon on the right of your instance line, you are then redirected to a page with all the instance details and its deployed pods and their state, and you are able to observe them by clicking on the `Logs` button.

A full example is given with the first-mentioned service, Hive Metastore. For the other services except Trino, the deployment configuration is given in YAML format, which can be executed through a `kubectl apply` command.

Each instance of a service may be edited by clicking on the little crayon at the right of the instance line of the instance page or permanently deleted by clicking on the dustbin.

Note: Changing the version of the KuboCD package requires you to delete the current instance and deploy a new one.

### Order and dependencies of services

There is a dependency order between KuboCD releases representing a service or an external component which the administrator must pay attention to. For instance, almost all services need either or both a storage provider and a database server delivered as a KuboCD package under the dependency names `storage` and `database-server` marked in their metadata in order to deploy.

### Deploying Hive Metastore

Hive Metastore depends on a connection with an S3 storage provider and a database server. If none of them exist, the deployment will not be possible.

Otherwise, deploying a Hive Metastore instance is straightforward. In the `Data Catalog` click on `hive-metastore`, then on `+ Deploy`. Choose an instance name and a Hive Metastore KuboCD package version and click on `Next`.

![Hive1](../../assets/hive1.png)

Then, select a connection to the database base server and a connection to the S3 storage provider. If they do not exist, they can be created by clicking on `+ New connection` and following the steps described in [Connections and external secrets management](/en/admin-guide/connections-and-external-secrets-management#creating-connections). Enter the resource amounts and choose a Kubernetes secret for the S3 storage authentication as well as optionally a bucket which limits the Hive Metastore. Once it is done, click on `CREATE`.

![Hive2](../../assets/hive2.png)

Now check that the configuration meets your expectation and click on `Deploy instance`.

![Hive3](../../assets/hive3.png)

Wait for the `Status` to be `READY`.

![Hive4](../../assets/hive4.png)

Clicking on the little eye icon on the right gives you access to the instance details showing the deployed pods as well as their status, and you may access their logs.

![Hive5](../../assets/hive5.png)

Clicking on the little crayon icon or the dustbin icon enables you to edit or delete the instance.

Here is the same deployment shown in the screenshots in YAML format.

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

### Deploying Polaris

Polaris needs a connection and the credentials to the S3 storage and a connection to an SQL database. In addition, at least one realm must be configured.

Note that the Polaris principal is used through the management API. Polaris does not authenticate Keycloak clients; it keeps its own principals.

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

### Deploying Trino

Before deploying Trino, make sure a Hive Metastore instance /and/or a Polaris instance is deployed.

You have the choice to add [OPA (Open Policy Agent)](https://www.openpolicyagent.org) as an RBAC component for Trino. Ticking the box OPA spawns a pod with an OPA server container and a Kube-management container to read configmaps within the project namespace. If, in addition, you choose [OPAL (Open Poly Agent Layer)](https://docs.opal.ac), the OPA pod only contains an OPA server and no Kube-management container. Policies are then synchronized through OPAL composed of the following pods:

- OPAL server
- OPAL client
- Postgres database

Policy synchronization is in this case done with the git repository entered in the required field. If only the OPAL box is selected, neither OPA nor OPAL is deployed.

![Trino](../../assets/trino.png)

You may add Polaris catalogs and choose the sizing meaning the number of workers and their maximum resource consumption.

When you then click on `Next`, all parameters are summarized. If you agree, click on `Deploy instance`.

Here is a deployment example with only OPA and not OPAL:

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

### Deploying Spark History Server

Spark History Server only needs access to the S3 storage provider. Therefore, you only need to provide the S3 connection details. In JSON format write the OIDC mapping for the access to the Spark History Server.

Here is an example in YAML format:

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

### Deploying JupyterHub

The minimum specifications for JupyterHub are an S3 storage connection, an S3 storage credential secret, and an OIDC role mapping in order to avoid a 403 error for any user who tries to have access. Here is the minimal configuration in YAML format:

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

With this minimal configuration, S3 bucket access is unrestricted, PySpark lacks access to any Polaris catalog, and no Welcome notebook is included. Except for the Welcome notebook, these settings can all be adjusted for more advanced JupyterHub deployments.

Here is an example of how to limit JupyterHub to certain buckets:

```yaml
spec:
  parameters:
    fileBrowserLocations:
      - { name: bronze, uri: s3://bronze }
      - { name: silver, uri: s3://silver }
      - { name: gold, uri: s3://gold }
```

Here is an example of how to give PySpark access to a Polaris catalog:

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

A welcome notebook is written under the key `spec.parameters.welcomeNotebook` in JSON format.

### Deploying Superset

Superset needs two database-server connections. One to the database containing the data and the other one to the metadata. Without an OIDC mapping, every user lands in the Public category, and Superset refuses all.

Here is an example of a Superset deployment in YAML format:

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
