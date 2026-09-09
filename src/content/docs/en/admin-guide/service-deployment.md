---
title: Service deployment
description: Service deployment using the OKDP control plane and using files in the deployments Git repository.
---

The OKDP control plane enables the administrator to deploy, delete, and monitor services, being its main goal.

A service in OKDP is a Helm chart plus a values file in the [deployments Git repository](/en/installation-requirements/deployments-repository). An instance of a service is a directory `projects/<project>/services/<instance>/` containing an `instance.yaml` file (the chart and its version) and a `values.yaml` file (the parameters). Flux or Argo CD deploys it as the Helm release `<project>-<instance>` in the project namespace. The console writes exactly these files, so you may deploy from the console or commit the files yourself: the result is the same.

Service deployment in the user interface always follows the same pattern. On the instance page of the service, click on `Deploy` and then 3 stages follow:

1. Choose the instance name and the chart version for the instance.
2. Configure the instance parameters.
3. Approve the recapitulation of the configuration.
4. You are returned to the instance page of the service, where you can check the instance status.
5. By clicking on the little eye icon on the right of your instance line, you are then redirected to a page with all the instance details and its deployed pods and their state, and you are able to observe them by clicking on the `Logs` button.

When you deploy, the console commits the files to Git and the instance shows the status `Pending` until the GitOps engine has picked the commit up. It then goes through `Installing` to `Ready`, or `Error` with the message of the engine. Only the parameters you set are written to `values.yaml`; the chart defaults apply to the others.

A full example is given with the first-mentioned service, Hive Metastore. For the other services except Trino, the deployment configuration is given as the two files of the instance, which you can commit to the deployments repository.

Each instance of a service may be edited by clicking on the little crayon at the right of the instance line of the instance page or permanently deleted by clicking on the dustbin. Editing the parameters rewrites `values.yaml`; choosing another version in the edit page rewrites `instance.yaml`, and the engine upgrades the release in place. Deleting an instance removes its directory from Git, and the engine uninstalls the release.

### Order and dependencies of services

Services of a project have no deployment order: each chart waits for what it needs and converges in any order. However, almost all services need a connection to a storage provider and/or a database server, which must exist before you deploy them: see [Connections and external secrets management](/en/admin-guide/connections-and-external-secrets-management). A service referring to another instance of the project (Trino to a Hive Metastore, Superset to Trino) names it by its release name `<project>-<instance>`.

### Deploying Hive Metastore

Hive Metastore depends on a connection with an S3 storage provider and a database server. If none of them exist, the deployment will not be possible.

Otherwise, deploying a Hive Metastore instance is straightforward. In the `Data Catalog` click on `hive-metastore`, then on `+ Deploy`. Choose an instance name and a Hive Metastore chart version and click on `Next`.

![Hive1](../../assets/hive1.png)

Then, select a connection to the database server and a connection to the S3 storage provider. If they do not exist, they can be created by clicking on `+ New connection` and following the steps described in [Connections and external secrets management](/en/admin-guide/connections-and-external-secrets-management#creating-connections). Enter the resource amounts and choose a Kubernetes secret for the S3 storage authentication as well as optionally a bucket which limits the Hive Metastore. Once it is done, click on `CREATE`.

![Hive2](../../assets/hive2.png)

Now check that the configuration meets your expectation and click on `Deploy instance`.

![Hive3](../../assets/hive3.png)

Wait for the `Status` to be `READY`.

![Hive4](../../assets/hive4.png)

Clicking on the little eye icon on the right gives you access to the instance details showing the deployed pods as well as their status, and you may access their logs.

![Hive5](../../assets/hive5.png)

Clicking on the little crayon icon or the dustbin icon enables you to edit or delete the instance.

Here is the same deployment shown in the screenshots, as files of the deployments repository. `projects/demo/services/hive/instance.yaml`:

```yaml
name: hive
project: demo
service: hive-metastore
chart: oci://quay.io/okdp/platform-charts/hive-metastore
version: 4.0.1-1.0.1
connections:
  - demo-db-hive
  - demo-storage
```

`projects/demo/services/hive/values.yaml`:

```yaml
db: demo-db-hive
storage: demo-storage
s3SecretRef: creds-hive-metastore-s3
warehouseBucket: hive
```

The release is `demo-hive`. It provides a `hive` connection named `demo-hive`, which Trino uses below.

### Deploying Polaris

Polaris needs a connection and the credentials to the S3 storage and a connection to a PostgreSQL database. It serves one realm, and may create Polaris principals in it.

Note that the Polaris principal is used through the management API. Polaris does not authenticate Keycloak clients; it keeps its own principals. The root credentials of the realm are generated once into the Secret `<release>-root` (here `demo-polaris-root`) and kept when the instance is deleted, since the realm lives on in the database.

`projects/demo/services/polaris/instance.yaml`:

```yaml
name: polaris
project: demo
service: polaris
chart: oci://quay.io/okdp/platform-charts/polaris
version: 1.3.0-incubating-2.0.0
connections:
  - demo-db-polaris
  - demo-storage
```

`projects/demo/services/polaris/values.yaml`:

```yaml
db: demo-db-polaris
storage: demo-storage
s3SecretRef: creds-polaris-s3
realm: sandbox
principals:
  - name: service-account-svc-polaris-api-admin
    roles: [service_admin, catalog_admin]
```

The release `demo-polaris` provides an `iceberg-catalog` connection named `demo-polaris`.

### Deploying Trino

Before deploying Trino, make sure a Hive Metastore instance and/or a Polaris instance is deployed.

You have the choice to add [OPA (Open Policy Agent)](https://www.openpolicyagent.org) as an RBAC component for Trino. Ticking the box OPA spawns a pod with an OPA server container and a Kube-management container to read configmaps within the project namespace. If, in addition, you choose [OPAL (Open Policy Administration Layer)](https://docs.opal.ac), the OPA pod only contains an OPA server and no Kube-management container. Policies are then synchronized through OPAL composed of the following pods:

- OPAL server
- OPAL client
- Postgres database

Policy synchronization is in this case done with the git repository entered in the required field. If only the OPAL box is selected, neither OPA nor OPAL is deployed.

![Trino](../../assets/trino.png)

You may add Polaris catalogs and choose the sizing meaning the number of workers and their maximum resource consumption.

When you then click on `Next`, all parameters are summarized. If you agree, click on `Deploy instance`.

Here is a deployment example with only OPA and not OPAL. The catalogs refer to the Hive Metastore and Polaris instances of the project by their release names; a catalog on a Polaris instance of the project must name its `warehouse`. `projects/demo/services/trino/instance.yaml`:

```yaml
name: trino
project: demo
service: trino
chart: oci://quay.io/okdp/platform-charts/trino
version: 480.0.0-1.0.1
connections:
  - demo-storage
```

`projects/demo/services/trino/values.yaml`:

```yaml
s3SecretRef: creds-trino-s3
hiveCatalogs:
  - name: bronze
    metastore: demo-hive
    storage: demo-storage
icebergCatalogs:
  - name: silver
    catalog: demo-polaris
    storage: demo-storage
    warehouse: silver
    oidcSecretRef: demo-polaris-root
  - name: gold
    catalog: demo-polaris
    storage: demo-storage
    warehouse: gold
    oidcSecretRef: demo-polaris-root
enableOPA: true
```

### Deploying Spark History Server

Spark History Server only needs access to the S3 storage provider. Therefore, you only need to provide the S3 connection details. In JSON format write the OIDC mapping for the access to the Spark History Server.

Here is an example, `projects/demo/services/spark-history/instance.yaml`:

```yaml
name: spark-history
project: demo
service: spark-history-server
chart: oci://quay.io/okdp/platform-charts/spark-history-server
version: 3.5.1-1.0.1
connections:
  - demo-storage
```

`projects/demo/services/spark-history/values.yaml`:

```yaml
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
```

### Deploying JupyterHub

The minimum specifications for JupyterHub are an S3 storage connection, an S3 storage credential secret, and an OIDC role mapping in order to avoid a 403 error for any user who tries to have access. Here is the minimal configuration, `projects/demo/services/jupyterhub/instance.yaml`:

```yaml
name: jupyterhub
project: demo
service: jupyterhub
chart: oci://quay.io/okdp/platform-charts/jupyterhub
version: 4.3.3-1.0.1
connections:
  - demo-storage
```

`projects/demo/services/jupyterhub/values.yaml`:

```yaml
cpu: 0.5
memoryGi: 1
oidcRoleMapping:
  admin_groups: [platform_admin]
  allowed_groups: [platform_admin]
s3SecretRef: creds-jupyterhub-s3
storage: demo-storage
```

With this minimal configuration, S3 bucket access is unrestricted, PySpark lacks access to any Polaris catalog, and no Welcome notebook is included. Except for the Welcome notebook, these settings can all be adjusted for more advanced JupyterHub deployments.

Here is an example of how to limit JupyterHub to certain buckets:

```yaml
fileBrowserLocations:
  - { name: bronze, uri: s3://bronze }
  - { name: silver, uri: s3://silver }
  - { name: gold, uri: s3://gold }
```

Here is an example of how to give PySpark access to a Polaris catalog, with the parameter `sparkConnections`. The placeholder `{{ .Values.global.okdp.ingress.suffix }}` is replaced by the ingress suffix of the platform; any other value is written as is.

```yaml
pyspark: [silver]
sparkConnections:
  - name: silver
    properties: |
      spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions
      spark.sql.catalog.silver=org.apache.iceberg.spark.SparkCatalog
      spark.sql.catalog.silver.type=rest
      spark.sql.catalog.silver.warehouse=silver
      spark.sql.catalog.silver.uri=https://polaris-demo.{{ .Values.global.okdp.ingress.suffix }}/api/catalog
      spark.sql.catalog.silver.oauth2-server-uri=https://keycloak.{{ .Values.global.okdp.ingress.suffix }}/realms/master/protocol/openid-connect/token
      spark.sql.catalog.silver.scope=profile
      spark.sql.catalog.silver.rest.auth.type=oauth2
      spark.sql.catalog.silver.token-refresh-enabled=true
      spark.sql.catalog.silver.header.X-Iceberg-Access-Delegation=vended-credentials
      spark.sql.catalog.silver.io-impl=org.apache.iceberg.io.ResolvingFileIO
      spark.sql.catalog.silver.header.Polaris-Realm=sandbox
      spark.sql.catalog.silver.client.region=us-east-1
      spark.sql.catalog.silver.s3.region=us-east-1
```

A welcome notebook is written under the parameter `welcomeNotebook` in JSON format.

### Deploying Superset

Superset needs two database-server connections. One to the database containing the data and the other one to the metadata. Without an OIDC mapping, every user lands in the Public category, and Superset refuses all. Its data sources may point at the Trino instance of the project by its release name.

Here is an example of a Superset deployment, `projects/demo/services/superset/instance.yaml`:

```yaml
name: superset
project: demo
service: superset
chart: oci://quay.io/okdp/platform-charts/superset
version: 6.0.0-1.1.0
connections:
  - demo-db-superset
  - demo-db-superset-examples
```

`projects/demo/services/superset/values.yaml`:

```yaml
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
  - { name: trino-bronze, trino: demo-trino, catalog: bronze }
  - { name: trino-silver, trino: demo-trino, catalog: silver }
  - { name: trino-gold, trino: demo-trino, catalog: gold }
```

### Deploying with Flux

With Flux, run `scripts/render-flux.sh` after adding, changing or removing an `instance.yaml` by hand, and commit the generated files with it (`helmrelease.yaml`, `kustomization.yaml`, `projects/<project>/kustomization.yaml`). The console does it for you. With Argo CD, the two files of the instance are enough.
