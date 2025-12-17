# Trilium Notes (Secure Wrapper)

A comprehensive Helm chart that deploys **Trilium Notes** protected by **OAuth2-Proxy** for authentication, with an integrated **Remote Backup** system.

## Features

* **Trilium Notes**: Deploys the server-side application for hierarchical note-taking.
* **OAuth2 Authentication**: Disables internal Trilium auth in favor of a secure OAuth2-Proxy sidecar (compatible with Google, GitHub, GitLab, OIDC, etc.).
* **Automated Backups**: A CronJob that compresses the Trilium data directory and securely transfers it to a remote server via SSH.
* **Ingress Management**: Automated TLS certificate generation via Cert-Manager (Let's Encrypt).

## Prerequisites

* Kubernetes 1.19+
* Helm 3.2.0+
* [Cert-Manager](https://cert-manager.io/docs/) installed in the cluster (Optional if you have the issuer enabled).
* [Ingress-Nginx Controller](https://kubernetes.github.io/ingress-nginx/).
* A remote server accessible via SSH for storing backups. (Optional if you have backups enabled)

## Installation

### 1. Create the Backup SSH Secret
Before installing the chart, you must create a Kubernetes secret containing the private SSH key used to connect to your backup server.

```bash
# Generate a keypair if you haven't already
ssh-keygen -t rsa -b 4096 -f ./id_rsa_trilium

# Create the secret in your namespace
kubectl create secret generic trilium-backup-ssh-key \
  --from-file=id_rsa=./id_rsa_trilium \
  --namespace your-namespace
```
Note: Ensure the corresponding public key is added to ~/.ssh/authorized_keys on your remote backup server.

2. Add the repository (ideally the Github version)
```bash 
helm repo add <repo-name> oci://ghcr.io/jonnydeates/charts/secure-trilium-stack
helm repo add <repo-name> https://gitlab.jonnydeates.com/api/v4/projects/5/packages/helm/stable

# Example
helm repo add trilium-github oci://ghcr.io/jonnydeates/charts/secure-trilium-stack
helm repo add trilium-gitlab https://gitlab.jonnydeates.com/api/v4/projects/5/packages/helm/stable
```

3. Install the Chart
```bash
helm install trilium trilium-github/secure-trilium-stack \
  --set oauth2-proxy.config.clientID="YOUR_CLIENT_ID" \
  --set oauth2-proxy.config.clientSecret="YOUR_CLIENT_SECRET" \
  --set oauth2-proxy.config.cookieSecret="$(openssl rand -base64 32 | head -c 32 | base64)"
  --...Other values....
```

##  Configuration
The following table lists the configurable parameters of the chart and their default values.

### Global & Ingress
| Parameter                  | Description                                                       | Default                 |
|:---------------------------|:------------------------------------------------------------------|:------------------------|
| `email`                    | Email address used for Let's Encrypt registration                 | `jonnydeates@gmail.com` |
| `ingress.enabled`          | Enable Ingress resource                                           | `true`                  |
| `ingress.host`             | Domain name for the application                                   | `notes.example.com`     |
| `ingress.secretName`       | Ingress secretName for the Issuer                                 | `notes.example.com`     |
| `ingress.className`        | Ingress class name                                                | `nginx`                 |
| `ingress.cloudflareToken`  | Token for Cloudflare DNS challenges (if empty, then ignored)      | `CHANGE_ME...`          |
| `ingress.issuer.enabled`   | Ingress Issuer for the given namespace                            | `true`                  |
| `ingress.issuer.isStaging` | Wheather or not to use staging environment of lets encrypt        | `true`                  |
| `ingress.tls.enabled`      | Dictates if TlS should be enabled                                 | `true`                  |
| `ingress.annotations.*`    | Shared annotations between the public ingress and the private one | `{cert-manager.io/issuer: letsencrypt}`                    |

### Trilium (App)
| Parameter                                    | Description                                                          | Default               |
|:---------------------------------------------|:---------------------------------------------------------------------|:----------------------|
| `trilium.image.tag`                          | Trilium Docker image tag                                             | `v0.99.5`             |
| `trilium.image.repository`                   | Trilium Docker image repository                                      | `triliumnext/trilium` |
| `trilium.image.pullPolicy`                   | Pull policy for the image                                            | `IfNotPresent`        |
| `trilium.config.general.noAuthentication`    | Disable internal auth                                                | `true`                |
| `trilium.config.general.noBackup`            | Disable internal backup                                              | `false`               |
| `trilium.config.network.trustedReverseProxy` | Additional setting to trust a reverse proxy                          | `true`                |
| `trilium.persistence.existingClaim`                   | Uses existing Claim instead of creating one, if empty it creates one | ``                    |
| `trilium.persistence.size`                   | Size of the Persistent Volume Claim                                  | `5Gi`                 |
| `trilium.persistence.accessMode`           | Storage access mode                                                  | `"ReadWriteOnce"`     |
| `trilium.persistence.storageClass`           | Storage Class for PVC                                                | `""`                  |


### OAuth2 Proxy Configuration
| Parameter                                          | Description                                                  | Default                                                    |
|:---------------------------------------------------|:-------------------------------------------------------------|:-----------------------------------------------------------|
| `oauth2-proxy.config.clientID`                     | Client ID from your Identity Provider (Google, GitLab, etc.) | `CHANGE_ME...`                                             |
| `oauth2-proxy.config.clientSecret`                 | Client Secret from your Identity Provider                    | `CHANGE_ME...`                                             |
| `oauth2-proxy.config.cookieSecret`                 | Random 32-byte string for cookie encryption                  | `CHANGE_ME...`                                             |
| `oauth2-proxy.extraArgs.provider`                  | The OAuth provider type (e.g., `oidc`, `google`, `github`)   | `oidc`                                                     |
| `oauth2-proxy.extraArgs.oidc-issuer-url`           | The URL of the OIDC issuer (required if provider is `oidc`)  | `https://gitlab.example.com`                               |
| `oauth2-proxy.extraArgs.redirect-url`              | The URL of the Redirect from trilium                         | `https://notes.example.com/oauth2/callback`                |
| `oauth2-proxy.extraArgs.email-domain`              | Restrict login to specific email domains (`*` allows any)    | `*`                                                        |
| `oauth2-proxy.extraArgs.reverse-proxy`             | Sets the reverse proxy header on the redirect response       | `true`                                                     |
| `oauth2-proxy.extraArgs.cookie-secure`             | Sets the requirement for https for the cookie                | `true`                                                     |
| `oauth2-proxy.extraArgs.set-authorization-header`  | Sets the authorization header                                | `true`                                                     |
| `oauth2-proxy.extraArgs.pass-authorization-header` | Adds the authorization header to the redirect response       | `true`                                                     |
| `oauth2-proxy.extraArgs.pass-user-headers`         | Adds the user headers to the redirect response               | `true`                                                     |
| `oauth2-proxy.image.registry`                      | The registry to pull the oauth2 image from                   | `quay.io`                                                     |
| `oauth2-proxy.image.repository`                    | The repository of the oauth2 image                           | `oauth2-proxy/oauth2-proxy`                                                     |
| `oauth2-proxy.image.tag`                           | The repository tag of the oauth2 image                       | ``                                                     |
| `oauth2-proxy.ingress.enabled`                     | Enable Ingress for the auth callback path                    | `true`                                                     |
| `oauth2-proxy.ingress.annotations`                 | The annotations for the auth ingress                         | `{cert-manager.io/issuer: letsencrypt}`                    |
| `oauth2-proxy.ingress.ingressClassName`            | The classname of the ingress                                 | `nginx`                                                    |
| `oauth2-proxy.ingress.path`                        | The path of the ingress                                      | `/oauth2`                                                  |
| `oauth2-proxy.ingress.pathType`                    | The pathType of the ingress                                  | `Prefix`                                                   |
| `oauth2-proxy.ingress.hosts`                       | List of hosts for the auth ingress                           | `[notes.example.com]`                                      |
| `oauth2-proxy.ingress.tls`                         | List of tls for the auth ingress                             | `[{hosts: [notes.example.com ], secretName: letsencrypt}]` |

### Automated Backups
| Parameter                  | Description                                                                    | Default                      |
|:---------------------------|:-------------------------------------------------------------------------------|:-----------------------------|
| `backup.enabled`           | Enable the automated backup CronJob                                            | `true`                       |
| `backup.schedule`          | Cron expression for when backups run                                           | `0 3 * * *` (3:00 AM)        |
| `backup.remote.host`       | IP address or hostname of the remote backup server                             | `192.168.1.50`               |
| `backup.remote.user`       | SSH username on the remote server                                              | `ubuntu_user`                |
| `backup.remote.directory`  | Directory path on remote server where files are saved                          | `/home/ubuntu_user/backups`  |
| `backup.sshSecretName`     | Name of the Kubernetes Secret containing the SSH private key                   | `trilium-backup-ssh-key`     |
| `backup.sshKeyFileName`    | Key name inside the secret (usually `id_rsa`)                                  | `id_rsa`                     |
| `backup.podSelector.key`   | Label key used to find the running Trilium pod                                 | `app.kubernetes.io/instance` |
| `backup.podSelector.value` | Label value used to find the running Trilium pod                               | `trilium`                    |
| `backup.triliumPort`       | The port trilium is running on in its container                                | `8080`                       |
| `backup.triliumDataPath`   | The place where trilium stores its backups (usually `/home/node/trilium-data`) | `/home/node/trilium-data`    |
| `backup.image.repository`  | The cronjobs image repository                                                  | `alpine`                     |
| `backup.image.tag`         | The cronjobs image's tag                                                       | `latest`                     |
| `backup.image.pullpolicy`  | The cronjobs image's pullpolicy                                                | `IfNotPresent`               |

### Authentication Flow
1. User visits notes.example.com. 
2. Nginx Ingress intercepts the request and forwards it to OAuth2-Proxy. 
3. User authenticates with the provider (Google/GitLab/etc). 
4. Upon success, the request is forwarded to Trilium. 
5. Trilium accepts the connection (Internal Auth is disabled via noAuthentication: true).

### Troubleshooting Backups
If the backup job fails, check the logs of the pod spawned by the CronJob:
```
# List jobs
kubectl get jobs

# Get logs
kubectl logs job/trilium-backup-manual-xxxxx
```
Common issues:
- SSH Key permissions: Ensure the remote server accepts the key provided in the trilium-backup-ssh-key secret.
- Host Key Verification: The backup script typically runs ssh with -o StrictHostKeyChecking=no for simplicity, but ensure the network path is open.