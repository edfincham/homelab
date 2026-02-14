# Homelab

Documenting my homelab setup as I go.

This documents my setup for a Kubernetes cluster running on Raspberry Pi hardware behind a Tailscale VPN. It includes automated GitOps deployments, secure ingress routing, automatic TLS certificate management, and automatic DNS updates.

## Table of Contents
- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Hardware Setup](#hardware-setup)
- [K3s Cluster Installation](#k3s-cluster-installation)
- [Core Infrastructure Components](#core-infrastructure-components)
- [GitOps with ArgoCD](#gitops-with-argocd)
- [Troubleshooting](#troubleshooting)

## Architecture Overview

The infrastructure stack includes:

- **K3s**: Lightweight Kubernetes distribution perfect for edge/IoT devices
- **Tailscale**: Zero-config VPN for secure remote access to cluster resources
- **Traefik**: Modern HTTP reverse proxy and load balancer for ingress routing
- **cert-manager**: Automated TLS certificate management using Let's Encrypt
- **ExternalDNS**: Automatic DNS record management in Cloudflare
- **Sealed Secrets**: Encrypted secrets safe to store in Git repositories
- **ArgoCD**: GitOps continuous delivery tool for Kubernetes

The cluster uses Tailscale for secure networking, Cloudflare for DNS management, and Let's Encrypt for automatic TLS certificate provisioning.

## Prerequisites

Before beginning, ensure you have the following tools installed on your local machine:

### Required Tools
- **Tailscale** - VPN client for secure cluster access
  - Installation: https://tailscale.com/download
  - Used for creating a secure mesh network between your devices and cluster nodes

- **kubectl** - Kubernetes command-line tool
  - Installation: https://kubernetes.io/docs/tasks/tools/
  - Version compatibility: Match to your K3s version (typically v1.28+)

- **kubeseal** - Client-side utility for Sealed Secrets
  - Installation: https://github.com/bitnami-labs/sealed-secrets#kubeseal
  - Used to encrypt secrets before committing to Git

- **Helm** - Kubernetes package manager
  - Installation: https://helm.sh/docs/intro/install/
  - Minimum version: 3.x

### Required Accounts
- **Tailscale account** - For VPN mesh networking
- **Cloudflare account** - For DNS management and API access
- **Domain name** - Registered domain managed by e.g. Cloudflare DNS

## Hardware Setup

### Raspberry Pi Preparation

This guide uses Raspberry Pi 4 Model B devices.

I've used Raspberry Pi 4B devices with 8GB RAM as the nodes in the cluster. One node is desginated as the control plane (or "master") node while the others are the "worker" nodes.

#### SD Card Imaging

Install the **Raspberry Pi Imager**. Ensure you use version 2.x or later as newer versions of the RPi OS use Cloud Init rather than the legacy `firstrun.sh` to persist your configuration. It took me a long time to figure this out.

1. **Select OS**: Choose `Raspberry Pi OS Lite 64-bit`
2. **Select Storage**: Choose your SD card
3. **Configure Settings** (click the gear icon):
   - **Hostname**: Set as `rpi-0`, `rpi-1`, `rpi-2`, etc. (incrementing for each node)
   - **Enable SSH**: Select "Use password authentication"
   - **Username**: `pi`
   - **Password**: Set a secure password
   - **Configure Wireless LAN**:
     - SSID: Your network name
     - Password: Your WiFi password
     - **Important**: Raspberry Pi 4 only supports 2.4GHz WiFi channels (not 5GHz)
     - Ensure your router has a 2.4GHz network enabled
4. **Write**: Flash the image to the SD card

#### Initial Boot and Network Discovery

1. Insert the SD card into your Raspberry Pi and power it on
2. The Pi should automatically connect to your WiFi network and be assigned an IP address
3. Locate the IP address using one of these methods:
   - Check your router's admin console for connected devices
   - Use `nmap` to scan your network: `nmap -sn 192.168.1.0/24`
4. Test SSH connectivity: `ssh pi@<IP_ADDRESS>`

### Tailscale Setup on Raspberry Pi Nodes

Tailscale creates a secure mesh VPN network, allowing your cluster to be accessible from anywhere without exposing services directly to the internet. Each node needs Tailscale installed.

On each Raspberry Pi node, run:

```shell
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Start Tailscale and authenticate
sudo tailscale up
```

The `tailscale up` command will output an authentication URL. Visit this URL in your browser to authorize the device in your Tailnet.

#### Creating Tailscale Auth Keys

For automated K3s setup, you'll need an auth key:

1. Navigate to Tailscale Admin Console: **Settings** → **Personal Settings** → **Keys** → **Auth keys**
2. Click **Generate auth key**
3. Configure the key:
   - Set an expiration time (or make it reusable)
   - Optional: Enable "Reusable" for multiple node setup
   - Optional: Tag the key with `tag:k3s` for organization
4. Copy the generated key (it starts with `tskey-auth-`)

This auth key will be used during K3s installation to automatically join nodes to your Tailnet.

## K3s Cluster Installation

K3s is a lightweight Kubernetes distribution built for IoT and edge computing. It's packaged as a single binary

### Generate Cluster Token

The K3s token is used to securely join worker nodes to the control plane. Generate a random secure token:

```shell
# Generate a random 20-character token
export K3S_TOKEN=$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | head -c 20)
echo $K3S_TOKEN
```

**Important**: Save this token securely. You'll need it to join any additional worker nodes to the cluster.

### Install K3s Control Plane (Master Node)

SSH into the Raspberry Pi that will serve as your master node (typically `rpi-0`):

```shell
ssh pi@<MASTER_IP>
```

Then execute the following installation script:

```shell
export K3S_TOKEN=<YOUR_GENERATED_TOKEN>
export KEY=<YOUR_TAILSCALE_AUTH_KEY>
export TAILSCALE_IP=$(tailscale ip | head -n 1 | xargs)

curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server \
  --token $K3S_TOKEN \
  --write-kubeconfig-mode 644 \
  --vpn-auth=name=tailscale,joinKey=$KEY \
  --node-external-ip=$TAILSCALE_IP \
  --disable traefik" sh -s -
```

#### Installation Options Explained

- `--token`: Authentication token for joining worker nodes
- `--write-kubeconfig-mode 644`: Makes kubeconfig readable by all users (needed for copying)
- `--vpn-auth`: Integrates K3s with Tailscale VPN
- `--node-external-ip`: Uses Tailscale IP for node communication (enables remote access)
- `--disable traefik`: Disables bundled Traefik (we'll install our own with custom config)

#### Verify Installation

Wait a few moments for K3s to start, then check the status:

```shell
systemctl status k3s

# Check node status
kubectl get nodes

# You should see output like:
# NAME     STATUS   ROLES                  AGE   VERSION
# rpi-0    Ready    control-plane,master   1m    v1.28.x+k3s1
```

If the installation fails, it may be necessary to add the following to the end of the `/boot/firmware/cmdline.txt` before rebooting:
```shell
cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1
```

Or inspect the journalctl:
```shell
journalctl -xeu k3s.service
```

### Configure kubectl on Local Machine

To manage your cluster from your local machine, you need to copy the kubeconfig file:
```shell
mkdir -p ~/.kube
scp pi@<MASTER_IP>:/etc/rancher/k3s/k3s.yaml ~/.kube/config-rpi-k3s
```

#### Update Kubeconfig Server Address

Now edit the copied config file. Specifically, change the `server` field from `https://127.0.0.1:6443` to your master node's Tailscale IP: `https://<MASTER_TAILSCALE_IP>:6443`. Use your editor of choice or just `sed`:

```shell
sed -i 's/127.0.0.1/<MASTER_TAILSCALE_IP>/g' ~/.kube/config-rpi-k3s
```

#### Install K3s on a Worker Node

Once the node has connected to the Tailnet (see [Tailscale Setup](#tailscale-setup-on-raspberry-pi-nodes) section), connect to the server node and retrieve the `K3S_TOKEN`:
```shell
sudo cat /var/lib/rancher/k3s/server/node-token
```

The `K3S_TOKEN` is the last digits after the `:server:` string.

You will also need the Tailscale IP of the server node:
```shell
tailscale ip | head -n 1 | xargs
```

And the Tailscale Auth key from before. Then connect to the worker node and execute:
```shell
curl -sfL https://get.k3s.io | K3S_URL=https://$TAILSCALE_IP:6443 \
    K3S_TOKEN=$K3S_TOKEN \
    INSTALL_K3S_EXEC="--vpn-auth=name=tailscale,joinKey=$KEY \
    --node-external-ip=$TAILSCALE_IP" sh -
```

#### Uninstall K3s
To remove K3s from a server node, SSH to the Raspberry Pi and run:
```shell
/usr/local/bin/k3s-uninstall.sh
```

To remove K3s from a worker/agent node, SSH to the Raspberry Pi and run:
```shell
/usr/local/bin/k3s-agent-uninstall.sh
```

## Core Infrastructure Components

### Create Namespaces

Namespaces are a key Kubernetes construct for logical isolation and security boundaries. Normally I like to create namespaces as part of the Helm deployment but in this case we are created sealed secrets prior to any Helm deployment. Accordingly, we must first create the relevent namespaces to store these secrets.

```shell
kubectl apply -f manifests/namespaces.yaml
```

This creates namespaces for:
- `traefik` - Ingress controller
- `cert-manager` - Certificate management
- `external-dns` - DNS automation
- `tailscale` - VPN operator

It may also create other namespaces for subsequent applications, but the above four are required for core deployments.

### Install Sealed Secrets

Sealed Secrets allows you to encrypt Kubernetes secrets so they can be safely stored in Git repositories. The secrets are encrypted with a public key and can only be decrypted by the sealed-secrets controller running in your cluster. First, let's install the sealed-secrets operator with Helm:

```shell
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update
helm install \
    sealed-secrets sealed-secrets/sealed-secrets \
    --namespace kube-system \
    --version 2.18.0 \
    --set-string fullnameOverride=sealed-secrets-controller
```

#### Verify Installation

```shell
kubectl get pods -n kube-system -l app.kubernetes.io/name=sealed-secrets

# Should show a running pod:
# NAME                                       READY   STATUS    RESTARTS   AGE
# sealed-secrets-controller-xxxxxxxxx-xxxxx  1/1     Running   0          1m
```

#### Using Sealed Secrets

To encrypt a secret:

1. Create a regular Kubernetes secret YAML file (don't commit this!)
2. Use `kubeseal` to encrypt it:
   ```shell
   kubeseal -f secrets/my-secret.yaml -w manifests/my-sealed-secret.yaml
   ```
3. The resulting `my-sealed-secret.yaml` is safe to commit to Git
4. Apply the sealed secret: `kubectl apply -f manifests/my-sealed-secret.yaml`
5. The controller automatically decrypts it into a regular Kubernetes secret

### Install Tailscale Operator

The Tailscale Kubernetes operator allows cluster services to join your Tailnet, making them securely accessible without public ingress.

#### Configure Tailscale ACLs

1. Navigate to [Tailscale Admin Console](https://login.tailscale.com/admin)
2. Go to **Access Controls** → **JSON Editor**
3. Add the following to the `tagOwners` section:

```json
{
  "tagOwners": {
    "tag:k8s-operator": [],
    "tag:k8s": ["tag:k8s-operator"]
  }
}
```

This allows the operator to create and manage tagged devices.

#### Create OAuth Client

1. Navigate to **Settings** → **Trust Credentials** → **+ Credential**
2. Select **OAuth**
3. Configure permissions:
   - **Devices** → Read/Write
   - **Keys** → Read/Write (for auth keys)
4. Add tags: `tag:k8s-operator`
5. Click **Generate** and save the **Client ID** and **Client Secret**

#### Create Tailscale Secret

Create a secret file `secrets/tailscale.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: operator-oauth
  namespace: tailscale
type: Opaque
stringData:
  client_id: <YOUR_OAUTH_CLIENT_ID>
  client_secret: <YOUR_OAUTH_CLIENT_SECRET>
```

Encrypt and apply the secret:

```shell
kubeseal -f secrets/tailscale.yaml -w manifests/tailscale.yaml
kubectl apply -f manifests/tailscale.yaml
```

#### Install Tailscale Operator via Helm

```shell
helm repo add tailscale https://pkgs.tailscale.com/helmcharts
helm repo update
helm install \
  tailscale tailscale/tailscale-operator \
  --namespace tailscale \
  --version 1.92.5
```

#### Verify Installation

```shell
kubectl get pods -n tailscale

# Should show the operator running:
# NAME                                  READY   STATUS    RESTARTS   AGE
# tailscale-operator-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
```

### Install cert-manager

cert-manager automates the management and issuance of TLS certificates from various sources, including Let's Encrypt. It ensures certificates are valid and up-to-date, automatically renewing them before expiration.

```shell
helm install \
  cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --version v1.19.2 \
  --namespace cert-manager \
  --set crds.enabled=true
```

The `--set crds.enabled=true` flag installs the Custom Resource Definitions (CRDs) needed for certificate management.

#### Verify Installation

```shell
kubectl get pods -n cert-manager

# Should show three running pods:
# NAME                                       READY   STATUS    RESTARTS   AGE
# cert-manager-xxxxxxxxxx-xxxxx              1/1     Running   0          1m
# cert-manager-cainjector-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
# cert-manager-webhook-xxxxxxxxxx-xxxxx      1/1     Running   0          1m
```

### Create Cloudflare API Token Secret

Multiple components need access to Cloudflare's API for DNS management:
- **cert-manager**: For DNS-01 ACME challenges (proves domain ownership for wildcard certificates)
- **external-dns**: For automatically creating/updating DNS records
- **traefik**: For DNS-based certificate validation

#### Create Cloudflare API Token

1. Log in to [Cloudflare Dashboard](https://dash.cloudflare.com)
2. Go to **My Profile** → **API Tokens** → **Create Token**
3. Use the **Edit zone DNS** template or create a custom token with:
   - **Permissions**:
     - Zone → DNS → Edit
     - Zone → Zone → Read
   - **Zone Resources**:
     - Include → All Zones (or specific zones you want to manage)
4. Click **Continue to summary** → **Create Token**
5. Copy the generated API token (it will only be shown once)

#### Create and Seal the Secret

Create a secret file `secrets/cloudflare.yaml` (don't commit this!):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare
  namespace: cert-manager
type: Opaque
stringData:
  api-token: <YOUR_CLOUDFLARE_API_TOKEN>
---
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare
  namespace: external-dns
type: Opaque
stringData:
  api-token: <YOUR_CLOUDFLARE_API_TOKEN>
---
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare
  namespace: traefik
type: Opaque
stringData:
  api-token: <YOUR_CLOUDFLARE_API_TOKEN>
```

Encrypt and apply:

```shell
kubeseal -f secrets/cloudflare.yaml -w manifests/cloudflare.yaml
kubectl apply -f manifests/cloudflare.yaml
```

### Create ClusterIssuer for Let's Encrypt

A ClusterIssuer is a cert-manager resource that represents a certificate authority. This creates an issuer for Let's Encrypt using DNS-01 validation with Cloudflare. It does this by using the Cloudflare API token secret which we just created.

```shell
kubectl apply -f manifests/cluster-issuer.yaml
```

Your `cluster-issuer.yaml` should look similar to:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: cloudflare
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: cert-manager-cloudflare
    solvers:
    - dns01:
        cloudflare:
          apiTokenSecretRef:
            name: cloudflare
            key: api-token
```

#### Verify ClusterIssuer

```shell
kubectl get clusterissuer

# Should show:
# NAME               READY   AGE
# cloudflare         True    30s
```

### Install Traefik Ingress Controller

Traefik is a modern HTTP reverse proxy and load balancer. While K3s includes Traefik, we disabled it during installation to use a custom configuration (principally, a .

```shell
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm install \
    traefik traefik/traefik \
    --version v38.0.2 \
    --namespace traefik \
    --values helm/traefik-values.yaml
```

#### Verify Installation

```shell
kubectl get pods -n traefik

# Should show Traefik pods running:
# NAME                       READY   STATUS    RESTARTS   AGE
# traefik-xxxxxxxxxx-xxxxx   1/1     Running   0          1m

# Check service
kubectl get svc -n traefik
```

### Install ExternalDNS

ExternalDNS automatically creates and updates DNS records in Cloudflare based on Kubernetes Ingress and Service resources. This eliminates manual DNS management.

```shell
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm repo update
helm install \
    external-dns external-dns/external-dns \
    --namespace external-dns \
    --version 1.20.0 \
    --values helm/externaldns-values.yaml
```

#### Verify Installation

```shell
kubectl get pods -n external-dns

# Should show:
# NAME                            READY   STATUS    RESTARTS   AGE
# external-dns-xxxxxxxxxx-xxxxx   1/1     Running   0          1m

# Check logs to verify Cloudflare connection
kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns
```

## GitOps with ArgoCD

ArgoCD is a declarative, GitOps continuous delivery tool for Kubernetes. It monitors Git repositories and automatically syncs the desired application state to your cluster.

### Install ArgoCD

```shell
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install \
  argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --version 9.3.0 \
  --values helm/argocd-values.yaml
```

#### ArgoCD Configuration

Key configurations in `helm/argocd-values.yaml`:

- **Insecure mode**: Since this is a local deployment accessed via Tailscale, TLS termination happens at Traefik, and ArgoCD server runs in insecure mode
- **Service type**: ClusterIP since we expose the UI via a Traefik Ingress
- **Ingress configuration**: Add Traefik annotations

### Access ArgoCD

#### Retrieve Initial Admin Password & Update in UI

The initial admin password is auto-generated and stored in a Kubernetes secret:

```shell
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d && echo
```

After first login, update the admin password:

1. Click on **User Info** in the ArgoCD UI (top left, user icon)
2. Click **Update Password**
3. Enter the current password and your new password
4. Save changes
