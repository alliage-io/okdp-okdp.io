---
title: Connections and external secrets management
description: How to create, delete and edit connections, external secret stores and external secrets in the OKDP control plane.
---

## Connections

A connection is a Kubernetes object brought by the Kubocd controller. It serves as a binding object between two services or to a storage or database provider.

Here is an example in YAML format of a connection:

```yaml
apiVersion: kubocd.kubotal.io/v1alpha1
kind: Connection
metadata:
  name: demo-storage
  namespace: demo
spec:
  contract: s3
  description: Platform S3 store, shared with this project
  values:
    apiUrl: https://storage-default-api.okdp.sandbox
    internalUrl: http://storage-s3.default.svc.cluster.local:8333
    region: us-east-1
    pathStyle: true
    secretRef: creds-seaweedfs-s3
```

In the user interface, the connection section is found under `Project Panel` -> `Connections` and brings you to the following page:

![Connections](../../assets/connections.png)

### Connection types

You may choose between several types of connections. Two types concern external components mentioned in [installation requirements](/en/installation-requirements)

- `S3 object storage`
- `SQL Database`

And three OKDP services:

- `Hive metastore` for thrift connections to the Hive Metastore
- `Iceberg Rest Catalog` being a connection to Polaris
- `Trino`

Choosing between these different types determines the other parameters to enter in order to establish a connection.

### Creating connections

To create a connection with the user interface, you click on `+ Add connection`, give it a name, and choose a type. Then fill the mandatory fields and click on `Create`.

![Connection creation](../../assets/connection-creation.png)

Clicking on the connection gives you access to its details. The three dots on the right-hand side of the connection line give you the choice to edit or remove the connection.

Note: Created connections through the command line are also displayed in the user interface and may be manipulated from there on.

![Connection details](../../assets/connection-details.png)

## Add a secret store and add external secrets

The OKDP control plane enables you to access secrets stored inside a secret store and produce a Kubernetes secret. The underlying component is the External Secrets Operator, which must be installed. In order to accomplish this task, a connection to a secret store must first be established. Then, you are able to synchronize and access the desired secret within the secret store to use for OKDP services.

### Add a secret store

The OKDP control plane has been tested with Vault as a secret store, and it is taken as an example in this case.

This operation creates a `secretStore` object from the External Secrets controller.

To create a secret store in the OKDP control plane, go to `Project Panel` -> `Secrets` -> `Secret Stores`, then click on `+ Add secret store`.

Fill the mandatory fields:

- `Store-name`, a name you give for the secret store connection.
- `Server URL`, the secret store's URL.
- `Secret path`, the path within the secret store where the secrets of interest are stored.
- `Authentication`, choose between `Token` where you paste the secrets store's authentication token or `Kubernetes` where you use a Kubernetes secret.
- `CA bundle`, if necessary paste the encoded CA bundle.

Then, if you click on `Test Connection` to test the communication between the OKDP control plane and the secret store, and if it does work, click on `Create`.

![Add a secret store](../../assets/secret-store.png)

Here are the details of the secret store in YAML format that was just created, displayed with the `kubectl get secretStore -n demo vault-main -o yaml` command:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  creationTimestamp: "2026-09-03T08:10:37Z"
  generation: 1
  name: vault-main
  namespace: demo
  resourceVersion: "185959"
  uid: c9868bd6-6d9b-4465-abd8-6698de6906cb
spec:
  provider:
    vault:
      auth:
        tokenSecretRef:
          key: token
          name: vault-main-credentials
      path: secret/
      server: http://vault-main.vault.svc.cluster.local:8200
      version: v2
status:
  capabilities: ReadWrite
  conditions:
    - lastTransitionTime: "2026-09-03T08:10:37Z"
      message: store validated
      reason: Valid
      status: "True"
      type: Ready
```

### Add an external secret

To create a Kubernetes secret from a secret stored in the secret store through the OKDP control plane, go to the `External Secrets` section, which is the tab next to the `Secret Store` tab, and click on `Add external secret`.

Fill in the following fields:

- `Name` being the name of the external secret.
- `Secret Store` being a secret store connection that has been created as shown previously.
- `Refresh interval` being the elapsed time before synchronizing with the secret store.
- In the `Data Mappings` section:
  - `SECRET KEY` being the key to the secret's value.
  - `REMOTE KEY` being the external secret's name in the secret store.
  - `PROPERTY` being optional and adding information about the secret.
- `Kubernetes secret name` is by default the same as the external secret name but can be modified.

Then click on `CREATE`.

![Secret creation](../../assets/secret-creation.png)

The `externalSecret` object has been created as follows:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  creationTimestamp: "2026-09-03T13:49:51Z"
  generation: 2
  name: app-secret-example
  namespace: demo
  resourceVersion: "281555"
  uid: 972b7dd9-1fa5-412d-a312-7da6c76e925f
spec:
  data:
    - remoteRef:
        conversionStrategy: Default
        decodingStrategy: None
        key: example-secret
        metadataPolicy: None
      secretKey: secret_key
  refreshInterval: 1h
  secretStoreRef:
    kind: SecretStore
    name: vault-main
  target:
    creationPolicy: Owner
    deletionPolicy: Retain
    name: app-secret-example
status:
  binding:
    name: app-secret-example
  conditions:
    - lastTransitionTime: "2026-09-03T13:53:45Z"
      message: secret synced
      reason: SecretSynced
      status: "True"
      type: Ready
  refreshTime: "2026-09-03T13:53:45Z"
  syncedResourceVersion: 2-2f2c636d06132f67992c15a1daeb0a5f
```

The synchronization takes place, and once it is ready, the status should be `Synced`.

![Secrets view](../../assets/secrets-view.png)

Now, in your terminal you should see a secret in the namespace with the same Kubernetes secret name:

```sh
kubectl get secret -n demo app-secret-example
NAME                 TYPE     DATA   AGE
app-secret-example   Opaque   1      52s
```

Once you delete the external secret in the OKDP control plane, the Kubernetes object disappears as well.
