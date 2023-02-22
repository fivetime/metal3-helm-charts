# Metal3 Helm chart

This chart is meant to manage the deployment of metal3 in a L3-Virtual Media setup through Helm.

## Prerequisites
There are two dependencies that are not managed through the metal3 chart because are related to applications that have a cluster-wide scope: `cert-manager` and a LoadBalancer Service provider such as `metallb` or `kube-vip`.

### cert-manager
In order to successfully deploy metal3 the cluster must have already installed the `cert-manager`.

You can install it through `helm` with:
```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.9.1 \
  --set installCRDs=true
```
, or via `kubectl` with:
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.9.1/cert-manager.yaml
```

### metallb
The Ironic services must be exposed in order to be reachable from the baremetal servers to be managed.
A way to do that is using `metallb` in `layer2` mode.

You can install it through `helm` with:
```bash
helm repo add metallb https://metallb.github.io/metallb
helm install \
  metallb metallb/metallb \
  --namespace metallb-system \
  --create-namespace
```
, or via `kubectl` with:
```bash
kubectl create namespace metallb-system
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.5/config/manifests/metallb-native.yaml -n metallb-system
```

Then you must configure the static IP address to be assigned to the Ironic svc.

In old versions of metallb, you can do it through a configMap:
```bash
cat <<EOF > metallb-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - <IRONIC_SVC_IP>-<IRONIC_SVC_IP>
EOF
kubectl apply -f metallb-config.yaml
```
Ensure that `IRONIC_SVC_IP` is the same IP configured in the `values.yaml file` in field `services.ironic.ironicIP`.

In the newer versions of metallb you can provide the IP pool configuration with:
```bash
cat <<EOF > metallb-config.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: ironic-pool
  namespace: metallb-system
spec:
  addresses:
  - <IRONIC_SVC_IP>-<IRONIC_SVC_IP>
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: ironic
  namespace: metallb-system
EOF
kubectl apply -f metallb-config.yaml
```

### kube-vip
Refer to the [official kube-vip documentation](https://kube-vip.io/docs/installation/) to install and configure it to serve LoadBalancer services.

It does not need any CRD to be configured to expose the Ironic services, it just exposes the chosen static IP configured in the `values.yaml` file in the `services.ironic.loadBalancerIP` field.

## Usage
Add the Helm chart repository:
```bash
helm repo add --username <username> --password <access_token> sylva-metal3 https://gitlab.com/api/v4/projects/43292746/packages/helm/stable
```
, then prepare your `values.yaml` file with the configuration of metal3 (in the next section there is the complete description of all the configurable parameters) and finally issue:
```bash
helm install metal3 sylva/metal3 -n metal3-system --values values.yaml --create-namespace
```

## Parameters
Below there is the full list of configuration parameters for the metal3 helm chart.

In most cases you should only need to provide `global.storageClass`, `services.ironic.ironicIP`, `httpProxy` and `httpsProxy`, and keep the defaults for other variables.

By default the chart deploys the `DIB ironic-python-agent images`, if you want to deploy `tinyIPA images` instead you can customize `images.ironicIPADownloader.repository` to `fedcicchiello/ironic-ipa-downloader` and `images.ironicIPADownloader.tag` to `tinyipa`.

### metal3 values.yaml

| Name                                         | Description                                                                                                                               | Value                                     |
| -------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| `global.storageClass`                        | Global storage class for dynamic provisioning                                                                                             | `""`                                      |
| `images.baremetalOperator.repository`        | baremetal-operator image repository                                                                                                       | `quay.io/metal3-io/baremetal-operator`    |
| `images.baremetalOperator.pullPolicy`        | baremetal-operator image pull policy                                                                                                      | `IfNotPresent`                            |
| `images.baremetalOperator.tag`               | baremetal-operator image tag                                                                                                              | `v0.1.1`                            |
| `images.ironic.repository`                   | ironic image repository                                                                                                                   | `quay.io/metal3-io/ironic`                |
| `images.ironic.pullPolicy`                   | ironic image pull policy                                                                                                                  | `IfNotPresent`                            |
| `images.ironic.tag`                          | ironic image tag                                                                                                                          | `capm3-v1.2.0`                            |
| `images.ironicIPADownloader.repository`      | ironic-ipa-downloader image repository                                                                                                    | `quay.io/metal3-io/ironic-ipa-downloader` |
| `images.ironicIPADownloader.pullPolicy`      | ironic-ipa-downloader image pull policy                                                                                                   | `IfNotPresent`                            |
| `images.ironicIPADownloader.tag`             | ironic-ipa-downloader image tag                                                                                                           | `latest`                                  |
| `persistence.ironic.storageClass`            | storageClass for the ironic shared volume                                                                                                 | `""`                                      |
| `persistence.ironic.size`                    | size of the ironic shared volume                                                                                                          | `1Gi`                                   |
| `persistence.ironic.accessMode`              | accessMode of the ironic shared volume PVC                                                                                                | `ReadWriteMany`                           |
| `auth.basicAuthEnabled`                      | ironic services basic authentication enabled                                                                                              | `true`                                    |
| `auth.ironicUsername`                        | ironic API service username                                                                                                               | `ironic`                                  |
| `auth.ironicPassword`                        | ironic API service password                                                                                                               | `changeme`                                |
| `auth.ironicInspectorUsername`               | ironic inspector service username                                                                                                         | `ironic`                                  |
| `auth.ironicInspectorPassword`               | ironic inspector service password                                                                                                         | `changeme`                                |
| `auth.tlsEnabled`                            | ironic services TLS enabled                                                                                                          | `true`                                |
| `ironicIPADownloaderBaseURI`                 | URI to download from the file `ironic-python-agent.tar`. If empty the default URI defined in the ironic-ipa-downloader image will be used | `""`                                      |
| `ironicIPAExtraHardwareCollectorEnabled`     | ironic-python-agent `extra-hardware` collector enabled. Important: must be set to false if tinyIPA images are used                        | `false`                                   |
| `httpProxy`                                  | URL of the HTTP proxy                                                                                                                     | `""`                                      |
| `httpsProxy`                                 | URL of the HTTPS proxy                                                                                                                    | `""`                                      |
| `noProxy`                                    | configuration of the noProxy                                                                                                              | `""`                                      |
| `services.ironic.type`                       | type of the ironic service                                                                                                                | `LoadBalancer`                            |
| `services.ironic.externalIPs`                | ExternalIPs statically assigned to the ironic service                                                                                                           | `["{{ .Values.services.ironic.ironicIP }}"]`                            |
| `services.ironic.ironicIP`                   | IP of the ironic service                                                                                                                  | `""`                                      |
| `services.ironic.ironicAPIPort`              | port of the ironic-api service                                                                                                            | `6385`                                    |
| `services.ironic.ironicInspectorPort`        | port of the ironic-inspector service                                                                                                      | `5050`                                    |
| `services.ironic.ironicHTTPPort`             | port of the ironic-http service                                                                                                           | `6180`                                    |
| `services.ironic.annotations`                | annotations of the ironic's service (tip: add `metallb.universe.tf/address-pool: <ironic-pool-name>` with metallb to select the address-pool for ironic)                                                                          | `{}`                                    |
| `mariadb.architecture`                       | MariaDB architecture (`standalone` or `replication`)                                                                                      | `replication`                             |
| `mariadb.auth.rootPassword`                  | Password for the `root` user. Ignored if existing secret is provided, otherwise automatically generated if empty.                                                                                              | `""`                                      |
| `mariadb.auth.ironicPassword`                | Password for the `ironic` user                                                                                                            | `changeme`                                |
| `mariadb.initdbScripts`                      | Dictionary of initdb scripts                                                                                                              |                                           |
| `mariadb.initdbScripts.primary.sql`          | SQL initialization script to be executed on the primary                                                                                   | `""`                                      |
| `mariadb.primary.configuration`              | MariaDB Primary configuration to be injected as ConfigMap                                                                                 | `""`                                      |
| `mariadb.primary.name`                       | Name of the primary database (eg primary, master, leader, ...)                                                                            | `primary`                                 |
| `mariadb.primary.persistence.enabled`        | Enable persistence on MariaDB primary replicas using a `PersistentVolumeClaim`. If false, use emptyDir                                    | `true`                                    |
| `mariadb.primary.persistence.size`           | MariaDB primary persistent volume size                                                                                                    | `1Gi`                                   |
| `mariadb.primary.persistence.storageClass`   | MariaDB primary persistent volume storage Class                                                                                           | `""`                                      |
| `mariadb.primary.service.type`               | MariaDB Primary Kubernetes service type                                                                                                   | `ClusterIP`                               |
| `mariadb.primary.service.ports.mysql`        | MariaDB Primary Kubernetes service port for MariaDB                                                                                       | `3306`                                    |
| `mariadb.primary.service.clusterIP`          | MariaDB Primary Kubernetes service clusterIP IP                                                                                           | `None`                                    |
| `mariadb.secondary.configuration`            | MariaDB Secondary configuration to be injected as ConfigMap                                                                               | `""`                                      |
| `mariadb.secondary.name`                     | Name of the secondary database (eg secondary, slave, ...)                                                                                 | `secondary`                               |
| `mariadb.secondary.persistence.size`         | MariaDB primary persistent volume size                                                                                                    | `1Gi`                                   |
| `mariadb.secondary.persistence.storageClass` | MariaDB primary persistent volume storage Class                                                                                           | `""`                                      |
| `mariadb.secondary.replicaCount`             | Number of MariaDB secondary replicas                                                                                                      | `1`                                       |
| `mariadb.secondary.service.type`             | MariaDB secondary Kubernetes service type                                                                                                 | `ClusterIP`                               |
| `mariadb.secondary.service.ports.mysql`      | MariaDB secondary Kubernetes service port for MariaDB                                                                                     | `3306`                                    |
| `mariadb.secondary.service.clusterIP`        | MariaDB secondary Kubernetes service clusterIP IP                                                                                         | `None`                                    |
| `serviceAccount.create`                      | Specifies whether a service account should be created                                                                                     | `true`                                    |
| `serviceAccount.annotations`                 | Annotations to add to the service account                                                                                                 | `{}`                                      |
| `serviceAccount.name`                        | The name of the service account to use.                                                                                                   | `baremetal-operator-controller-manager`   |
| `nodeSelector`                               | nodeSelector for metal3 Pods                                                                                                              | `{}`                                      |
| `tolerations`                                | tolerations for metal3 Pods                                                                                                               | `[]`                                      |
| `affinity`                                   | affinity for metal3 Pods                                                                                                                  | `{}`                                      |


#### Chart parameters auto-generation
The chart parameters table above has been generated with [bitnami-labs readme generator for helm](https://github.com/bitnami-labs/readme-generator-for-helm).

## CD pipeline and Helm repository channels
A CD pipeline is configured to automatically package the helm chart and publish it to the Gitlab hosted helm repository, you can find it in the left menu under `Packages and registries`->`Package Registry`.

The helm repository is structured into two channels:
1. `stable`: contains the helm charts built from the `main` branch at every new commit;
2. `devel`: contains the helm charts built from branches different from `main` at every new commit.

```bash
# Command to add the helm repo from the stable channel
helm repo add --username <username> --password <access_token> sylva-metal3 https://gitlab.com/api/v4/projects/43292746/packages/helm/stable

# Command to add the helm repo from the devel channel
helm repo add --username <username> --password <access_token> sylva-metal3-devel https://gitlab.com/api/v4/projects/43292746/packages/helm/devel
```

In order to use it properly at every change to the chart the field `version` into the [Chart.yaml file](Chart.yaml) should be incremented.

Use the `helm repo update` command to sync your local helm client to the latest charts built.
