# Setup of CX Portal & IAM for local development

![Version: 0.1.0](https://img.shields.io/badge/Version-0.1.0-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square)

This umbrella chart installs the helm charts of the [CX Portal](https://github.com/eclipse-tractusx/portal-cd/blob/portal-1.6.0/charts/portal/README.md) and of the [CX IAM](https://github.com/eclipse-tractusx/portal-iam) Keycloak instances ([centralidp](https://github.com/eclipse-tractusx/portal-iam/blob/centralidp-1.2.0/charts/centralidp/README.md) and [sharedidp](https://github.com/eclipse-tractusx/portal-iam/blob/sharedidp-1.2.0/charts/sharedidp/README.md)).

It's intended for the local setup of the those components in order to aid the local development. To integrate your local development, adapt the address values in the Values file for [Portal Frontend](./values.yaml#L23) and/or [Portal Backend](./values.yaml#L27).

This chart also sets up an additional [postgresql](https://artifacthub.io/packages/helm/bitnami/postgresql) instance to support the Portal Backend development as well as a [pgadmin4](https://artifacthub.io/packages/helm/runix/pgadmin4) instance for easy access to the deployed Postgres databases which are only available from within the Kubernetes cluster.

For detailed information about the default configuration values, please have a look at the [Values table](#values) and/or [Values file](./values.yaml).

## Usage

The following steps describe how to setup the LocalDev chart into the namespace 'localdev' of your started [**Minikube**](https://minikube.sigs.k8s.io/docs/start) cluster.

> **Note**
>
> In its current state of development, this chart as well as the following installation guide have been tested on Linux and Mac.
>
> **Linux** is the **preferred platform** to install this chart on as the network setup with Minikube is very straightforward on Linux.
>
> We plan to test the chart's reliability also on Windows and to update the installation guide accordingly.

> **Recommendations**
>
> Resources for Minikube
> | CPU(cores) | Memory(GB) |
> | :--------: | :--------: |
> |     2      |      6     |
>
>```bash
> minikube start --cpus=2 --memory 6gb
>```
>
> Use the dashboard provided by Minikube to get an overview about the deployed components:
>
> ```bash
> minikube dashboard
> ```

1. [Prepare self-signed TLS setup](#1-prepare-self-signed-tls-setup)
2. [Prepare network setup](#2-prepare-network-setup)
3. [Install from released chart or portal-cd repository](#3-install-from-released-chart-or-portal-cd-repository)
4. [Perform first login](#4-perform-first-login)

### 1. Prepare self-signed TLS setup

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```
```bash
helm install \
  cert-manager jetstack/cert-manager \
  --namespace localdev \
  --create-namespace \
  --version v1.13.0 \
  --set installCRDs=true
```

```bash
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-selfsigned-ca
  namespace: localdev
spec:
  isCA: true
  commonName: cx.local
  secretName: root-secret
  privateKey:
    algorithm: RSA
    size: 2048
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
    group: cert-manager.io
  subject:
    organizations:
      - CX
    countries:
      - DE
    provinces:
      - Some-State
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: my-ca-issuer
spec:
  ca:
    secretName: root-secret
EOF
```
See [cert-manager self-signed](https://cert-manager.io/docs/configuration/selfsigned) for reference.

### 2. Prepare network setup

In order to enable the local access via **ingress**, use the according addon for Minikube:

```bash
minikube addons enable ingress
```

Make sure that the **DNS** resolution for the hostnames is in place:

```bash
minikube addons enable ingress-dns
```
And execute installation step [3 Add the `minikube ip` as a DNS server](https://minikube.sigs.k8s.io/docs/handbook/addons/ingress-dns) for your OS:

```
domain example.org
nameserver 192.168.49.2
search_order 1
timeout 5
```
Replace 192.168.49.2 with your minikube ip.

To find out the IP address of your Minikube:

```bash
minikube ip
```

If afterwards at [step 4](#4-perform-first-login) your still facing DNS issues, add the following to your /etc/hosts file:

```
192.168.49.2    centralidp.example.org
192.168.49.2    sharedidp.example.org
192.168.49.2    portal.example.org
192.168.49.2    portal-backend.example.org
192.168.49.2    pgadmin4.example.org
```

Replace 192.168.49.2 with your minikube ip.

**Additional network setup** (for Mac only)

Install and start [Docker Mac Net Connect](https://github.com/chipmk/docker-mac-net-connect#installation).

We also recommend to execute the usage example after install to check proper setup.

If you're having issues with getting 'Docker Mac Net Connect' to work, we recommend to check out this issue: [#21](https://github.com/chipmk/docker-mac-net-connect/issues/21).

The tool is necessary due to [#7332](https://github.com/kubernetes/minikube/issues/7332).

### 3. Install from released chart or [portal-cd](https://github.com/eclipse-tractusx/portal-cd) repository

#### From the released chart

Install the chart with the release name 'local':

```bash
helm repo add tractusx-dev https://eclipse-tractusx.github.io/charts/dev
```

```bash
helm install local tractusx-dev/localdev-portal-iam --namespace localdev
```

To set your own configuration and secret values, install the helm chart with your own values file:

```bash
helm install -f your-values.yaml local tractusx-dev/localdev-portal-iam --namespace localdev
```

#### From [portal-cd](https://github.com/eclipse-tractusx/portal-cd) repository:

Make sure to clone the [portal-cd](https://github.com/eclipse-tractusx/portal-cd) repository beforehand.

Then change to the chart directory:

```bash
cd charts/localdev/
```
Download the chart dependencies:

```bash
helm dependency update
```

Install the chart with the release name 'local':

```bash
helm install local . --namespace localdev
```

To set your own configuration and secret values, install the helm chart with your own values file:

```bash
helm install local -f your-values.yaml . --namespace localdev
```

> **Note**
>
> It is to be expected that the pods for the **portal-migrations** job will run into an error a couple of times until the portal database is ready to take connections.
> The job will recreate pods until one run is successful.
>

### 4. Perform first login

Make sure to accept the risk of the self-signed certificates for the following hosts using the continue option:
- [centralidp.example.org/auth/](https://centralidp.example.org/auth/)
- [sharedidp.example.org/auth/](https://sharedidp.example.org/auth/)
- [portal-backend.example.org](https://portal-backend.example.org)
- [portal.example.org](https://portal.example.org)
- [pgadmin4.example.org](https://pgadmin4.example.org)

Then proceed with the login to the [portal](https://portal.example.org) to verify that everything is setup as expected.

Credentials to log into the initial example realm (CX-Operator):

```
cx-operator@cx.com
```

```
7XSXRwYLAm5kU2H
```

```mermaid
%%{
  init: {
    'flowchart': { 'diagramPadding': '10', 'wrappingWidth': '', 'nodeSpacing': '', 'rankSpacing':'', 'titleTopMargin':'10', 'curve':'basis'},
    'theme': 'base',
    'themeVariables': {
      'primaryColor': '#b3cb2d',
      'primaryBorderColor': '#ffa600',
      'lineColor': '#ffa600',
      'tertiaryColor': '#fff'
    }
  }
}%%
        graph TD
          classDef stroke stroke-width:2px
          classDef external fill:#4cb5f5,stroke:#4cb5f5,stroke-width:2px
          classDef addext fill:#4cb5f5,stroke:#b7b8b6,stroke-width:2px
          iam1(IAM: centralidp Keycloak):::stroke
          iam2(IAM: sharedidp Keycloak):::stroke
          portal(Portal):::stroke
          ps(Postgres):::external
          addpgadmin4(pgadmin 4):::addext
          ps --- iam1 & iam2 & portal
          ps -.- addpgadmin4
          subgraph Login Flow
            iam1 === portal
            iam1 === iam2
            end
          linkStyle 0,1,2 stroke:lightblue
          linkStyle 3 stroke:lightgrey
```

## Requirements

| Repository | Name | Version |
|------------|------|---------|
| https://charts.bitnami.com/bitnami | postgresportal(postgresql) | 12.12.x |
| https://eclipse-tractusx.github.io/charts/dev | centralidp | 2.0.0-alpha |
| https://eclipse-tractusx.github.io/charts/dev | portal | 1.7.0-alpha |
| https://eclipse-tractusx.github.io/charts/dev | sharedidp | 2.0.0-alpha |
| https://helm.runix.net | pgadmin4 | 1.17.x |

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| portal.enabled | bool | `true` |  |
| portal.portalAddress | string | `"https://portal.example.org"` | Set your local frontend to integrate into local development. |
| portal.portalBackendAddress | string | `"https://portal-backend.example.org"` | Set your local backend service to integrate into local development. Start port forwarding tunnel for database access, e.g.: 'kubectl port-forward service/portal-backend-postgresql-primary 5432:5432' |
| portal.replicaCount | int | `1` |  |
| portal.frontend.ingress.enabled | bool | `true` |  |
| portal.frontend.ingress.annotations."cert-manager.io/cluster-issuer" | string | `"my-ca-issuer"` |  |
| portal.frontend.ingress.annotations."nginx.ingress.kubernetes.io/rewrite-target" | string | `"/$1"` |  |
| portal.frontend.ingress.annotations."nginx.ingress.kubernetes.io/use-regex" | string | `"true"` |  |
| portal.frontend.ingress.annotations."nginx.ingress.kubernetes.io/enable-cors" | string | `"true"` |  |
| portal.frontend.ingress.annotations."nginx.ingress.kubernetes.io/cors-allow-origin" | string | `"https://*.example.org"` |  |
| portal.frontend.ingress.tls[0] | object | `{"hosts":["portal.example.org"],"secretName":"portal.example.org-tls"}` | Provide tls secret. |
| portal.frontend.ingress.tls[0].hosts | list | `["portal.example.org"]` | Provide host for tls secret. |
| portal.frontend.ingress.hosts[0].host | string | `"portal.example.org"` |  |
| portal.frontend.ingress.hosts[0].paths[0].path | string | `"/(.*)"` |  |
| portal.frontend.ingress.hosts[0].paths[0].pathType | string | `"ImplementationSpecific"` |  |
| portal.frontend.ingress.hosts[0].paths[0].backend.service | string | `"portal"` |  |
| portal.frontend.ingress.hosts[0].paths[0].backend.port | int | `8080` |  |
| portal.frontend.ingress.hosts[0].paths[1].path | string | `"/registration/(.*)"` |  |
| portal.frontend.ingress.hosts[0].paths[1].pathType | string | `"ImplementationSpecific"` |  |
| portal.frontend.ingress.hosts[0].paths[1].backend.service | string | `"registration"` |  |
| portal.frontend.ingress.hosts[0].paths[1].backend.port | int | `8080` |  |
| portal.frontend.ingress.hosts[0].paths[2].path | string | `"/((assets|documentation)/.*)"` |  |
| portal.frontend.ingress.hosts[0].paths[2].pathType | string | `"ImplementationSpecific"` |  |
| portal.frontend.ingress.hosts[0].paths[2].backend.service | string | `"assets"` |  |
| portal.frontend.ingress.hosts[0].paths[2].backend.port | int | `8080` |  |
| portal.backend.ingress.enabled | bool | `true` |  |
| portal.backend.ingress.name | string | `"portal-backend"` |  |
| portal.backend.ingress.annotations."cert-manager.io/cluster-issuer" | string | `"my-ca-issuer"` |  |
| portal.backend.ingress.annotations."nginx.ingress.kubernetes.io/use-regex" | string | `"true"` |  |
| portal.backend.ingress.annotations."nginx.ingress.kubernetes.io/enable-cors" | string | `"true"` |  |
| portal.backend.ingress.annotations."nginx.ingress.kubernetes.io/proxy-body-size" | string | `"8m"` |  |
| portal.backend.ingress.annotations."nginx.ingress.kubernetes.io/cors-allow-origin" | string | `"http://localhost:3000, https://*.example.org"` |  |
| portal.backend.ingress.tls[0] | object | `{"hosts":["portal-backend.example.org"],"secretName":"portal-backend.example.org-tls"}` | Provide tls secret. |
| portal.backend.ingress.tls[0].hosts | list | `["portal-backend.example.org"]` | Provide host for tls secret. |
| portal.backend.ingress.hosts[0].host | string | `"portal-backend.example.org"` |  |
| portal.backend.ingress.hosts[0].paths[0].path | string | `"/api/registration"` |  |
| portal.backend.ingress.hosts[0].paths[0].pathType | string | `"Prefix"` |  |
| portal.backend.ingress.hosts[0].paths[0].backend.service | string | `"registration-service"` |  |
| portal.backend.ingress.hosts[0].paths[0].backend.port | int | `8080` |  |
| portal.backend.ingress.hosts[0].paths[1].path | string | `"/api/administration"` |  |
| portal.backend.ingress.hosts[0].paths[1].pathType | string | `"Prefix"` |  |
| portal.backend.ingress.hosts[0].paths[1].backend.service | string | `"administration-service"` |  |
| portal.backend.ingress.hosts[0].paths[1].backend.port | int | `8080` |  |
| portal.backend.ingress.hosts[0].paths[2].path | string | `"/api/notification"` |  |
| portal.backend.ingress.hosts[0].paths[2].pathType | string | `"Prefix"` |  |
| portal.backend.ingress.hosts[0].paths[2].backend.service | string | `"notification-service"` |  |
| portal.backend.ingress.hosts[0].paths[2].backend.port | int | `8080` |  |
| portal.backend.ingress.hosts[0].paths[3].path | string | `"/api/provisioning"` |  |
| portal.backend.ingress.hosts[0].paths[3].pathType | string | `"Prefix"` |  |
| portal.backend.ingress.hosts[0].paths[3].backend.service | string | `"provisioning-service"` |  |
| portal.backend.ingress.hosts[0].paths[3].backend.port | int | `8080` |  |
| portal.backend.ingress.hosts[0].paths[4].path | string | `"/api/apps"` |  |
| portal.backend.ingress.hosts[0].paths[4].pathType | string | `"Prefix"` |  |
| portal.backend.ingress.hosts[0].paths[4].backend.service | string | `"marketplace-app-service"` |  |
| portal.backend.ingress.hosts[0].paths[4].backend.port | int | `8080` |  |
| portal.backend.ingress.hosts[0].paths[5].path | string | `"/api/services"` |  |
| portal.backend.ingress.hosts[0].paths[5].pathType | string | `"Prefix"` |  |
| portal.backend.ingress.hosts[0].paths[5].backend.service | string | `"services-service"` |  |
| portal.backend.ingress.hosts[0].paths[5].backend.port | int | `8080` |  |
| portal.backend.dotnetEnvironment | string | `"Development"` |  |
| portal.backend.keycloak.central.clientId | string | `"central-client-id"` |  |
| portal.backend.keycloak.central.clientSecret | string | `""` |  |
| portal.backend.keycloak.central.jwtBearerOptions.requireHttpsMetadata | string | `"false"` |  |
| portal.backend.keycloak.central.dbConnection | object | `{"password":""}` | Password for the user 'kccentral', defined at centralidp.secrets.postgresql.auth.existingSecret.password |
| portal.backend.keycloak.shared.clientId | string | `"shared-client-id"` |  |
| portal.backend.keycloak.shared.clientSecret | string | `""` |  |
| portal.backend.mailing.host | string | `"smtp.example.org"` |  |
| portal.backend.mailing.port | string | `"587"` |  |
| portal.backend.mailing.user | string | `"smtp-user"` |  |
| portal.backend.mailing.password | string | `""` |  |
| portal.backend.provisioning.sharedRealm.smtpServer.host | string | `"smtp.example.org"` |  |
| portal.backend.provisioning.sharedRealm.smtpServer.port | string | `"587"` |  |
| portal.backend.provisioning.sharedRealm.smtpServer.user | string | `"smtp-user"` |  |
| portal.backend.provisioning.sharedRealm.smtpServer.password | string | `""` |  |
| portal.backend.provisioning.sharedRealm.smtpServer.from | string | `"smtp@example.org"` |  |
| portal.backend.provisioning.sharedRealm.smtpServer.replyTo | string | `"smtp@example.org"` |  |
| portal.postgresql.auth.password | string | `""` | Password for the root username 'postgres'. Secret-key 'postgres-password'. |
| portal.postgresql.auth.portalPassword | string | `""` | Password for the non-root username 'portal'. Secret-key 'portal-password'. |
| portal.postgresql.auth.provisioningPassword | string | `""` | Password for the non-root username 'provisioning'. Secret-key 'provisioning-password'. |
| centralidp.enabled | bool | `true` |  |
| centralidp.keycloak.proxy | string | `"edge"` |  |
| centralidp.keycloak.nameOverride | string | `"centralidp"` |  |
| centralidp.keycloak.replicaCount | int | `1` |  |
| centralidp.keycloak.extraEnvVars[0].name | string | `"KEYCLOAK_SPI_TRUSTSTORE_FILE"` |  |
| centralidp.keycloak.extraEnvVars[0].value | string | `"/opt/bitnami/keycloak/certs/keycloak.truststore.jks"` |  |
| centralidp.keycloak.extraEnvVars[1].name | string | `"KEYCLOAK_SPI_TRUSTSTORE_PASSWORD"` |  |
| centralidp.keycloak.extraEnvVars[1].valueFrom.secretKeyRef.name | string | `"centralidp-spi"` |  |
| centralidp.keycloak.extraEnvVars[1].valueFrom.secretKeyRef.key | string | `"spi-truststore-password"` |  |
| centralidp.keycloak.extraEnvVars[2].name | string | `"KEYCLOAK_EXTRA_ARGS"` |  |
| centralidp.keycloak.extraEnvVars[2].value | string | `"-Dkeycloak.migration.action=import -Dkeycloak.migration.provider=singleFile -Dkeycloak.migration.file=/realms/CX-Central-realm.json -Dkeycloak.migration.strategy=IGNORE_EXISTING"` |  |
| centralidp.keycloak.extraVolumes[0].name | string | `"certificates"` |  |
| centralidp.keycloak.extraVolumes[0].secret.secretName | string | `"root-secret"` |  |
| centralidp.keycloak.extraVolumes[0].secret.defaultMode | int | `420` |  |
| centralidp.keycloak.extraVolumes[1].name | string | `"shared-certs"` |  |
| centralidp.keycloak.extraVolumes[1].emptyDir | object | `{}` |  |
| centralidp.keycloak.extraVolumes[2].name | string | `"themes"` |  |
| centralidp.keycloak.extraVolumes[2].emptyDir | object | `{}` |  |
| centralidp.keycloak.extraVolumes[3].name | string | `"realms"` |  |
| centralidp.keycloak.extraVolumes[3].emptyDir | object | `{}` |  |
| centralidp.keycloak.extraVolumeMounts[0].name | string | `"certificates"` |  |
| centralidp.keycloak.extraVolumeMounts[0].mountPath | string | `"/certs"` |  |
| centralidp.keycloak.extraVolumeMounts[1].name | string | `"shared-certs"` |  |
| centralidp.keycloak.extraVolumeMounts[1].mountPath | string | `"/opt/bitnami/keycloak/certs"` |  |
| centralidp.keycloak.extraVolumeMounts[2].name | string | `"themes"` |  |
| centralidp.keycloak.extraVolumeMounts[2].mountPath | string | `"/opt/bitnami/keycloak/themes/catenax-central"` |  |
| centralidp.keycloak.extraVolumeMounts[3].name | string | `"realms"` |  |
| centralidp.keycloak.extraVolumeMounts[3].mountPath | string | `"/realms"` |  |
| centralidp.keycloak.initContainers[0].name | string | `"init-certs"` |  |
| centralidp.keycloak.initContainers[0].image | string | `"docker.io/bitnami/keycloak:22.0.3-debian-11-r14"` |  |
| centralidp.keycloak.initContainers[0].imagePullPolicy | string | `"Always"` |  |
| centralidp.keycloak.initContainers[0].command[0] | string | `"/bin/bash"` |  |
| centralidp.keycloak.initContainers[0].args[0] | string | `"-ec"` |  |
| centralidp.keycloak.initContainers[0].args[1] | string | `"keytool -import -file \"/certs/tls.crt\" \\\n        -keystore \"/opt/bitnami/keycloak/certs/keycloak.truststore.jks\" \\\n        -storepass \"${KEYCLOAK_SPI_TRUSTSTORE_PASSWORD}\" \\\n        -noprompt"` |  |
| centralidp.keycloak.initContainers[0].env[0].name | string | `"KEYCLOAK_SPI_TRUSTSTORE_PASSWORD"` |  |
| centralidp.keycloak.initContainers[0].env[0].valueFrom.secretKeyRef.name | string | `"centralidp-spi"` |  |
| centralidp.keycloak.initContainers[0].env[0].valueFrom.secretKeyRef.key | string | `"spi-truststore-password"` |  |
| centralidp.keycloak.initContainers[0].volumeMounts[0].name | string | `"certificates"` |  |
| centralidp.keycloak.initContainers[0].volumeMounts[0].mountPath | string | `"/certs"` |  |
| centralidp.keycloak.initContainers[0].volumeMounts[1].name | string | `"shared-certs"` |  |
| centralidp.keycloak.initContainers[0].volumeMounts[1].mountPath | string | `"/opt/bitnami/keycloak/certs"` |  |
| centralidp.keycloak.initContainers[1].name | string | `"import"` |  |
| centralidp.keycloak.initContainers[1].image | string | `"tractusx/portal-iam:v1.2.0"` |  |
| centralidp.keycloak.initContainers[1].imagePullPolicy | string | `"Always"` |  |
| centralidp.keycloak.initContainers[1].command[0] | string | `"sh"` |  |
| centralidp.keycloak.initContainers[1].args[0] | string | `"-c"` |  |
| centralidp.keycloak.initContainers[1].args[1] | string | `"echo \"Copying themes...\"\ncp -R /import/themes/catenax-central/* /themes\necho \"Copying realms...\"\ncp -R /import/catenax-central/realms/* /realms\n"` |  |
| centralidp.keycloak.initContainers[1].volumeMounts[0].name | string | `"themes"` |  |
| centralidp.keycloak.initContainers[1].volumeMounts[0].mountPath | string | `"/themes"` |  |
| centralidp.keycloak.initContainers[1].volumeMounts[1].name | string | `"realms"` |  |
| centralidp.keycloak.initContainers[1].volumeMounts[1].mountPath | string | `"/realms"` |  |
| centralidp.keycloak.postgresql.nameOverride | string | `"centralidp-postgresql"` |  |
| centralidp.keycloak.ingress.enabled | bool | `true` |  |
| centralidp.keycloak.ingress.ingressClassName | string | `"nginx"` |  |
| centralidp.keycloak.ingress.hostname | string | `"centralidp.example.org"` |  |
| centralidp.keycloak.ingress.annotations."cert-manager.io/cluster-issuer" | string | `"my-ca-issuer"` |  |
| centralidp.keycloak.ingress.annotations."nginx.ingress.kubernetes.io/cors-allow-credentials" | string | `"true"` |  |
| centralidp.keycloak.ingress.annotations."nginx.ingress.kubernetes.io/cors-allow-methods" | string | `"PUT, GET, POST, OPTIONS"` |  |
| centralidp.keycloak.ingress.annotations."nginx.ingress.kubernetes.io/cors-allow-origin" | string | `"https://centralidp.example.org"` |  |
| centralidp.keycloak.ingress.annotations."nginx.ingress.kubernetes.io/enable-cors" | string | `"true"` |  |
| centralidp.keycloak.ingress.annotations."nginx.ingress.kubernetes.io/proxy-buffer-size" | string | `"128k"` |  |
| centralidp.keycloak.ingress.annotations."nginx.ingress.kubernetes.io/proxy-buffering" | string | `"on"` |  |
| centralidp.keycloak.ingress.annotations."nginx.ingress.kubernetes.io/proxy-buffers-number" | string | `"20"` |  |
| centralidp.keycloak.ingress.annotations."nginx.ingress.kubernetes.io/use-regex" | string | `"true"` |  |
| centralidp.keycloak.ingress.tls | bool | `true` |  |
| centralidp.secrets.auth.existingSecret.adminpassword | string | `""` | Password for the admin username 'admin'. Secret-key 'admin-password'. |
| centralidp.secrets.auth.spi.truststorePassword | string | `""` |  |
| sharedidp.enabled | bool | `true` |  |
| sharedidp.keycloak.proxy | string | `"edge"` |  |
| sharedidp.keycloak.nameOverride | string | `"sharedidp"` |  |
| sharedidp.keycloak.replicaCount | int | `1` |  |
| sharedidp.keycloak.extraEnvVars[0].name | string | `"KEYCLOAK_SPI_TRUSTSTORE_FILE"` |  |
| sharedidp.keycloak.extraEnvVars[0].value | string | `"/opt/bitnami/keycloak/certs/keycloak.truststore.jks"` |  |
| sharedidp.keycloak.extraEnvVars[1].name | string | `"KEYCLOAK_SPI_TRUSTSTORE_PASSWORD"` |  |
| sharedidp.keycloak.extraEnvVars[1].valueFrom.secretKeyRef.name | string | `"sharedidp-spi"` |  |
| sharedidp.keycloak.extraEnvVars[1].valueFrom.secretKeyRef.key | string | `"spi-truststore-password"` |  |
| sharedidp.keycloak.extraEnvVars[2].name | string | `"KEYCLOAK_EXTRA_ARGS"` |  |
| sharedidp.keycloak.extraEnvVars[2].value | string | `"-Dkeycloak.migration.action=import -Dkeycloak.migration.provider=dir -Dkeycloak.migration.dir=/realms -Dkeycloak.migration.strategy=IGNORE_EXISTING"` |  |
| sharedidp.keycloak.extraVolumes[0].name | string | `"certificates"` |  |
| sharedidp.keycloak.extraVolumes[0].secret.secretName | string | `"root-secret"` |  |
| sharedidp.keycloak.extraVolumes[0].secret.defaultMode | int | `420` |  |
| sharedidp.keycloak.extraVolumes[1].name | string | `"shared-certs"` |  |
| sharedidp.keycloak.extraVolumes[1].emptyDir | object | `{}` |  |
| sharedidp.keycloak.extraVolumes[2].name | string | `"themes-catenax-shared"` |  |
| sharedidp.keycloak.extraVolumes[2].emptyDir | object | `{}` |  |
| sharedidp.keycloak.extraVolumes[3].name | string | `"themes-catenax-shared-portal"` |  |
| sharedidp.keycloak.extraVolumes[3].emptyDir | object | `{}` |  |
| sharedidp.keycloak.extraVolumes[4].name | string | `"realms"` |  |
| sharedidp.keycloak.extraVolumes[4].emptyDir | object | `{}` |  |
| sharedidp.keycloak.extraVolumes[5].name | string | `"realm-secrets"` |  |
| sharedidp.keycloak.extraVolumes[5].secret.secretName | string | `"secret-sharedidp-example-realm"` |  |
| sharedidp.keycloak.extraVolumeMounts[0].name | string | `"certificates"` |  |
| sharedidp.keycloak.extraVolumeMounts[0].mountPath | string | `"/certs"` |  |
| sharedidp.keycloak.extraVolumeMounts[1].name | string | `"shared-certs"` |  |
| sharedidp.keycloak.extraVolumeMounts[1].mountPath | string | `"/opt/bitnami/keycloak/certs"` |  |
| sharedidp.keycloak.extraVolumeMounts[2].name | string | `"themes-catenax-shared"` |  |
| sharedidp.keycloak.extraVolumeMounts[2].mountPath | string | `"/opt/bitnami/keycloak/themes/catenax-shared"` |  |
| sharedidp.keycloak.extraVolumeMounts[3].name | string | `"themes-catenax-shared-portal"` |  |
| sharedidp.keycloak.extraVolumeMounts[3].mountPath | string | `"/opt/bitnami/keycloak/themes/catenax-shared-portal"` |  |
| sharedidp.keycloak.extraVolumeMounts[4].name | string | `"realms"` |  |
| sharedidp.keycloak.extraVolumeMounts[4].mountPath | string | `"/realms"` |  |
| sharedidp.keycloak.extraVolumeMounts[5].name | string | `"realm-secrets"` |  |
| sharedidp.keycloak.extraVolumeMounts[5].mountPath | string | `"/secrets"` |  |
| sharedidp.keycloak.initContainers[0].name | string | `"init-certs"` |  |
| sharedidp.keycloak.initContainers[0].image | string | `"docker.io/bitnami/keycloak:22.0.3-debian-11-r14"` |  |
| sharedidp.keycloak.initContainers[0].imagePullPolicy | string | `"Always"` |  |
| sharedidp.keycloak.initContainers[0].command[0] | string | `"/bin/bash"` |  |
| sharedidp.keycloak.initContainers[0].args[0] | string | `"-ec"` |  |
| sharedidp.keycloak.initContainers[0].args[1] | string | `"keytool -import -file \"/certs/tls.crt\" \\\n        -keystore \"/opt/bitnami/keycloak/certs/keycloak.truststore.jks\" \\\n        -storepass \"${KEYCLOAK_SPI_TRUSTSTORE_PASSWORD}\" \\\n        -noprompt"` |  |
| sharedidp.keycloak.initContainers[0].env[0].name | string | `"KEYCLOAK_SPI_TRUSTSTORE_PASSWORD"` |  |
| sharedidp.keycloak.initContainers[0].env[0].valueFrom.secretKeyRef.name | string | `"sharedidp-spi"` |  |
| sharedidp.keycloak.initContainers[0].env[0].valueFrom.secretKeyRef.key | string | `"spi-truststore-password"` |  |
| sharedidp.keycloak.initContainers[0].volumeMounts[0].name | string | `"certificates"` |  |
| sharedidp.keycloak.initContainers[0].volumeMounts[0].mountPath | string | `"/certs"` |  |
| sharedidp.keycloak.initContainers[0].volumeMounts[1].name | string | `"shared-certs"` |  |
| sharedidp.keycloak.initContainers[0].volumeMounts[1].mountPath | string | `"/opt/bitnami/keycloak/certs"` |  |
| sharedidp.keycloak.initContainers[1].name | string | `"import"` |  |
| sharedidp.keycloak.initContainers[1].image | string | `"tractusx/portal-iam:v1.2.0"` |  |
| sharedidp.keycloak.initContainers[1].imagePullPolicy | string | `"Always"` |  |
| sharedidp.keycloak.initContainers[1].command[0] | string | `"sh"` |  |
| sharedidp.keycloak.initContainers[1].args[0] | string | `"-c"` |  |
| sharedidp.keycloak.initContainers[1].args[1] | string | `"echo \"Copying themes-catenax-shared...\"\ncp -R /import/themes/catenax-shared/* /themes-catenax-shared\necho \"Copying themes-catenax-shared-portal...\"\ncp -R /import/themes/catenax-shared-portal/* /themes-catenax-shared-portal\necho \"Copying realm...\"\ncp -R /import/catenax-shared/realms/CX-Operator-realm.json /realms\ncp -R /import/catenax-shared/realms/master-realm.json /realms\necho \"Copying realm-secret...\"\ncp /secrets/CX-Operator-users-0.json /realms\n"` |  |
| sharedidp.keycloak.initContainers[1].volumeMounts[0].name | string | `"themes-catenax-shared"` |  |
| sharedidp.keycloak.initContainers[1].volumeMounts[0].mountPath | string | `"/themes-catenax-shared"` |  |
| sharedidp.keycloak.initContainers[1].volumeMounts[1].name | string | `"themes-catenax-shared-portal"` |  |
| sharedidp.keycloak.initContainers[1].volumeMounts[1].mountPath | string | `"/themes-catenax-shared-portal"` |  |
| sharedidp.keycloak.initContainers[1].volumeMounts[2].name | string | `"realms"` |  |
| sharedidp.keycloak.initContainers[1].volumeMounts[2].mountPath | string | `"/realms"` |  |
| sharedidp.keycloak.initContainers[1].volumeMounts[3].name | string | `"realm-secrets"` |  |
| sharedidp.keycloak.initContainers[1].volumeMounts[3].mountPath | string | `"/secrets"` |  |
| sharedidp.keycloak.postgresql.nameOverride | string | `"sharedidp-postgresql"` |  |
| sharedidp.keycloak.ingress.enabled | bool | `true` |  |
| sharedidp.keycloak.ingress.ingressClassName | string | `"nginx"` |  |
| sharedidp.keycloak.ingress.hostname | string | `"sharedidp.example.org"` |  |
| sharedidp.keycloak.ingress.annotations."cert-manager.io/cluster-issuer" | string | `"my-ca-issuer"` |  |
| sharedidp.keycloak.ingress.annotations."nginx.ingress.kubernetes.io/cors-allow-credentials" | string | `"true"` |  |
| sharedidp.keycloak.ingress.annotations."nginx.ingress.kubernetes.io/cors-allow-methods" | string | `"PUT, GET, POST, OPTIONS"` |  |
| sharedidp.keycloak.ingress.annotations."nginx.ingress.kubernetes.io/cors-allow-origin" | string | `"https://sharedidp.example.org"` |  |
| sharedidp.keycloak.ingress.annotations."nginx.ingress.kubernetes.io/enable-cors" | string | `"true"` |  |
| sharedidp.keycloak.ingress.annotations."nginx.ingress.kubernetes.io/proxy-buffer-size" | string | `"128k"` |  |
| sharedidp.keycloak.ingress.annotations."nginx.ingress.kubernetes.io/proxy-buffering" | string | `"on"` |  |
| sharedidp.keycloak.ingress.annotations."nginx.ingress.kubernetes.io/proxy-buffers-number" | string | `"20"` |  |
| sharedidp.keycloak.ingress.annotations."nginx.ingress.kubernetes.io/use-regex" | string | `"true"` |  |
| sharedidp.keycloak.ingress.tls | bool | `true` |  |
| sharedidp.secrets.auth.existingSecret.adminpassword | string | `""` | Password for the admin username 'admin'. Secret-key 'admin-password'. |
| sharedidp.secrets.auth.spi.truststorePassword | string | `""` |  |
| postgresportal.enabled | bool | `true` | Additional PostgreSQL for backend development; start port forwarding tunnel for database access, e.g.: 'kubectl port-forward local-portal-postgresql-primary-0 5432:5432' |
| postgresportal.nameOverride | string | `"portal-postgresql"` |  |
| postgresportal.auth.database | string | `"postgres"` |  |
| postgresportal.auth.port | int | `5432` |  |
| postgresportal.auth.existingSecret | string | `"secret-postgres-init-localdev"` |  |
| postgresportal.auth.password | string | `""` |  |
| postgresportal.auth.replicationPassword | string | `""` |  |
| postgresportal.auth.portalUser | string | `"portal"` |  |
| postgresportal.auth.provisioningUser | string | `"provisioning"` |  |
| postgresportal.auth.provisioningPassword | string | `""` |  |
| postgresportal.architecture | string | `"replication"` |  |
| postgresportal.audit.pgAuditLog | string | `"write, ddl"` |  |
| postgresportal.audit.logLinePrefix | string | `"%m %u %d "` |  |
| postgresportal.primary.initdb.scriptsConfigMap | string | `"configmap-postgres-init-localdev"` |  |
| postgresportal.primary.extraEnvVars[0].name | string | `"PORTAL_PASSWORD"` |  |
| postgresportal.primary.extraEnvVars[0].valueFrom.secretKeyRef.name | string | `"{{ .Values.auth.existingSecret }}"` |  |
| postgresportal.primary.extraEnvVars[0].valueFrom.secretKeyRef.key | string | `"portal-password"` |  |
| postgresportal.primary.extraEnvVars[1].name | string | `"PROVISIONING_PASSWORD"` |  |
| postgresportal.primary.extraEnvVars[1].valueFrom.secretKeyRef.name | string | `"{{ .Values.auth.existingSecret }}"` |  |
| postgresportal.primary.extraEnvVars[1].valueFrom.secretKeyRef.key | string | `"provisioning-password"` |  |
| pgadmin4.enabled | bool | `true` |  |
| pgadmin4.env.email | string | `"local@example.org"` |  |
| pgadmin4.ingress.enabled | bool | `true` |  |
| pgadmin4.ingress.ingressClassName | string | `"nginx"` |  |
| pgadmin4.ingress.annotations."cert-manager.io/cluster-issuer" | string | `"my-ca-issuer"` |  |
| pgadmin4.ingress.hosts[0].host | string | `"pgadmin4.example.org"` |  |
| pgadmin4.ingress.hosts[0].paths[0].path | string | `"/"` |  |
| pgadmin4.ingress.hosts[0].paths[0].pathType | string | `"Prefix"` |  |
| pgadmin4.ingress.tls[0].hosts[0] | string | `"pgadmin4.example.org"` |  |
| pgadmin4.ingress.tls[0].secretName | string | `"pgadmin4.example.org-tls"` |  |

Autogenerated with [helm docs](https://github.com/norwoodj/helm-docs)
