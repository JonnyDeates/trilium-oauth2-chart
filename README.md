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
* [Cert-Manager](https://cert-manager.io/docs/) installed in the cluster.
* [Ingress-Nginx Controller](https://kubernetes.github.io/ingress-nginx/).
* A remote server accessible via SSH for storing backups.

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

2. Install the Chart
```bash
helm install trilium ./trilium-chart \
  --set oauth2-proxy.config.clientID="YOUR_CLIENT_ID" \
  --set oauth2-proxy.config.clientSecret="YOUR_CLIENT_SECRET" \
  --set oauth2-proxy.config.cookieSecret="$(openssl rand -base64 32 | head -c 32 | base64)"
```

##  Configuration
The following table lists the configurable parameters of the chart and their default values.

### Global & Ingress
``` 
Parameter,Description,Default
email,Email address used for Let's Encrypt registration,jonnydeates@gmail.com
ingress.enabled,Enable Ingress resource,true
ingress.host,Domain name for the application,notes.example.com
ingress.className,Ingress class name,nginx
ingress.cloudflareToken,Token for Cloudflare DNS challenges (if using Cloudflare),CHANGE_ME...
```

### Trilium (App)
```
Parameter,Description,Default
trilium.image.tag,Trilium Docker image tag,v0.99.5
trilium.config.general.noAuthentication,Disable internal auth (relies on OAuth proxy),true
trilium.persistence.size,Size of the Persistent Volume Claim,5Gi
trilium.persistence.storageClass,Storage Class for PVC (empty uses default),""""""
```

### Automated Backups
```
Parameter,Description,Default
backup.enabled,Enable the backup CronJob,true
backup.schedule,Cron schedule for backups,0 3 * * * (3 AM)
backup.remote.host,IP/Hostname of the remote backup server,192.168.1.50
backup.remote.user,SSH user on remote server,ubuntu_user
backup.remote.directory,Target directory on remote server,/home/ubuntu_user/backups
backup.sshSecretName,Name of K8s secret containing SSH key,trilium-backup-ssh-key
backup.podSelector.value,Label value to find the running Trilium pod,trilium
```

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