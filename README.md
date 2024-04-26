# Homelab

## K3s Setup on Raspberry Pi(s)

Using the `rpi-imager` tool:
- Select `Raspberry Pi OS 64-bit`
- Select the SD card as storage
- Set the hostname as `rp-i` where `i` is the index
- Enable password-based SSH for a user `pi`
- Configure the wireless LAN (note that Raspberry Pis currently require 2.4GHz channels)

Once written to the SD card, insert and boot the Raspberry Pi. It should boot and be assigned a dynamic IP address on the network. The router's admin console should list the connected devices and their associated IP addresses. With the IP address, you should be able to test the SSH connection by running `ssh pi@$IP` using the password set when imaging the SD card.

### Assign Static IP Addresses
Login to the router's admin console and amend the DHCP range (which may be referred to as a Pool LAN). My range was `192.168.0.2` to `192.168.0.255`. To make space for some static IP addresses, adjust the end value. For instance, `192.168.0.240` provides 15 IP addresses for static assignment.

To set the static addresses:
1. SSH to the Raspberry Pi
   ```shell
   ssh pi@$IP
   ```
2. Install `vim`
   ```shell
   sudo apt-get install vim
   ```
3. Edit `/etc/dhcpcd.conf` (or create it if it does not exist)
   ```shell
   sudo vim /etc/dhcpcd.conf
   ```
4. Add/edit the section for static addresses and add
   ```shell
   interface eth0
   static ip_address=192.168.0.240/24
   static routers=192.168.0.1
   static domain_name_servers=192.168.0.1
   ```

   In this case, set `192.168.0.240` to the static IP which you intend to set for the Raspberry Pi. Also, set `192.168.0.1` to the router's IP
5. Reboot the Raspberry Pi
   ```shell
   sudo reboot
   ```
6. After reboot, SSH to the static IP
   ```shell
   ssh pi@192.168.0.240
   ```
7. If this does not work, assign a static lease via the network router's admin console. Less elegant, but sure to work.
8. Setup `/etc/hosts` mappings on the local machine for each Raspberry Pi
   ```shell
   192.168.0.240 rpi-1
   ...
   192.168.0.xxx rpi-n
   ```

### Setup Password-less SSH (Optional)
This is just a quality of life option to prevent having to retype passwords when connecting to Raspberry Pis

1. Generate an SSH key on your local machine
   ```shell
   ssh-keygen
   ```
   Accept default options and don't enter a password. This will create a `~/.ssh/id_rsa` (private key) and `~/.ssh/id_rsa.pub` (public key)
2. Copy the contents of `~/.ssh/id_rsa.pub`
3. SSH to a Raspberry Pi and run:
   ```shell
   mkdir ~/.ssh
   touch ~/.ssh/authorized_keys
   chmod 0700 ~/.ssh
   chmod 0600 ~/.ssh/authorized_keys
   vim ~/.ssh/authorized_keys
   ```
   And paste the contents of the public key. Save the file
4. Repeat this for each Raspberry Pi

After this, you should be able to connect to any Raspberry Pi without entering a password

### Install K3s
1. Create a token which is used to add workers to the server:
   ```shell
   export K3S_TOKEN=$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | head -c 20)
   echo $K3S_TOKEN
   ```
2. Connect to the Raspberry Pi which will serve as the master node and execute:
   ```shell
   curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --token $K3S_TOKEN --write-kubeconfig-mode 644 --bind-address $STATIC_IP --disable servicelb --disable traefik" sh -s -
   ```

   Where `$STATIC_IP` is the IP previously configured for that device. The command itself installs K3s with a couple of flags passed. If you're working with an old Raspberry Pi as the single node in your K3s cluster, it can struggle to schedule everything. To alleviate this load, consider passing one or more [`--disable` flags](https://docs.k3s.io/installation/packaged-components?_highlight=disable#using-the---disable-flag).

   If the installation fails, it may be necessary to add the following to the end of the `/boot/cmdline.txt` before rebooting:
   ```shell
   cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1
   ```

3. Connect to the Raspberry Pi(s) which will serve as worker or agent node(s) and execute:
   ```shell
   curl -sfL https://get.k3s.io | K3S_URL=https://$SERVER_IP:6443 K3S_TOKEN=$K3S_TOKEN sh -
   ```

   To retrieve the `K3S_TOKEN`, connect to the server node and run:
   ```shell
   sudo cat /var/lib/rancher/k3s/server/node-token
   ```
   The `K3S_TOKEN` is the last digits after the `:server:` string. The `$STATIC_IP` is the static IP configured for the master node.

#### Uninstall K3s
To remove K3s from a server node, SSH to the Raspberry Pi and run:
```shell
/usr/local/bin/k3s-uninstall.sh
```

To remove K3s from a worker/agent node, SSH to the Raspberry Pi and run:
```shell
/usr/local/bin/k3s-agent-uninstall.sh
```

### Configure Kubectl
It may take a couple of runs of the playbook for it to successfully install. Once completed, `kubectl` can be configured on the local machine to connect to K3s

1. Transfer the `kubeconfig` from the master node to the local machine
   ```shell
   scp pi@rpi-1:/etc/rancher/k3s/k3s.yaml ~/.kube/config-rpi-k3s
   ```
2. In the `config-rpi-k3s` file, replace the value of the server field with the IP or name of your K3s server
3. Configure the `KUBECONFIG` environment variable (add to `~/.bashrc`)
   ```shell
   export KUBECONFIG="/home/$USER/.kube/config"
   export KUBECONFIG="$KUBECONFIG:/home/$USER/.kube/config-rpi-k3s"
   ```

   On my machine, the `KUBECONFIG` requires absolute paths rather than relative
4. Use `kubectl` to check the available config contexts
   ```shell
   kubectl config get-contexts
   ```

   Ensure the Raspberry Pi K3s context has a useful name. If not, change it with:
   ```shell
   kubectl config rename-context default rpi-k3s
   ```

### Deploy Metallb
Since we're doing this bare-metal, we need a load balancer implementation. For this, we can use `metallb`: 
```shell
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
```

To understand how this all operates, we can use the manifests in the `examples` folder.

Now that we have our own load balancer implementation, we need to create a pool of IP addresses which can be assigned as needed. The pool isn't enough, however. We also need to be able to advertise the assignment of any IPs to the router. This is what the `L2Advertisement` achieves. 
```shell
kubectl apply -f examples/metallb.yaml
```
This creates an address pool for the 14 addresses ranging from `192.168.0.241` to `192.168.0.255`.

Note that we also specify a node selector. This is important since in L2 mode, only one node is elected to announce the IP.

### Deploy Custom DNS
Now that we have a load balancer implementation with an pool of available IP addresses to assign, we can deploy a custom DNS server on a fixed IP address. We can then configure the router to point to this IP as the primary DNS server (with Google/Cloudflare/etc. as a secondary), which means that we can create DNS records specific to the local network. First things first, we apply the below manifest which creates a `cytopia` deployment with one DNS record behind a load balancer with the external IP `192.168.0.241` assigned.
```shell
kubectl apply -f examples/dns.yaml
```

The next step involves logging into the control plane on your router, and changing the primary DNS server to `192.168.0.241` (be sure to add a secondary/backup). This means that the first place your router checks for resolving any DNS request is this `cytopia` deployment.

Note that the deployment includes one DNS record for `exammple.home.lab`, pointing to the IP address `192.168.0.242`. This means that any DNS lookup for `example.home.lab` should always resolve to this IP on your local network. You can validate this by running the below command on your local machine:
```shell
dig example.home.lab +vc +short
```

To see this in action, apply the `nginx` deployment in the `examples` directory to create a simple `nginx` webserver behind a load balancer with the address `192.168.0.242`.
```shell
kubectl apply -f examples/nginx.yaml
```

Once this has been deployed you should be able to open your web browser and navigate to `example.home.lab` and see the default `nginx` landing page. Note that we are on a unencrypted HTTP connection. We could install cert-manager and get TLS configured for secure HTTPS, but this will be problematic. Deploying TLS requires a certificate authority (CA) which can validate that we own the host, but we do not own the `example.home.lab` host. The alternative is to issue a self-signed certificate, but no device on the local network will recognise that without additional manual configuration.

#### Custom DNS via Helm
The manifest that we have applied (`examples.dns.yaml`) is, as the directory name implies, an example. We can make this a bit more configurable by packaging it into a Helm chart. This has the advantage of allowing us to specify our DNS records in a neater fashion. We can use the following field in the `values.yaml` file to template records into a config map which is then used by the main deployment as environment variables:
For instance:
```yaml
dnsRecords:
- domain: 192.168.0.0
  address: "example.home.lab"
```

To begin with, we can see what resources the chart creates from our local machine with:
```shell
helm template --debug dns helm/charts/dns
```
Or:
```shell
helm upgrade --install dns helm/charts/dns --dry-run
```

We can then install the chart with:
```shell
helm upgrade --install dns helm/charts/dns
```

Finally, we can uninstall the Helm chart and all corresponding resources via:
```shell
helm uninstall dns
```

Installing a packaged Helm chart from our local machine is neat and all but, since this is a Git repository, we can do one better with GitOps.

### Deploy ArgoCD
[ArgoCD](https://argo-cd.readthedocs.io/en/stable/) is a GitOps tool for Kubernetes. The tool uses the contents of Git repositories as the source of truth for defining the desired application state. The state may be defined in a number of ways, ranging from Helm charts to directories of vanilla manifests.

ArgoCD automates the deployment of these states to a specified target environment. Once deployed, the ArgoCD controller continuously monitors running applications and compares the current, live state against the desired state as defined in the Git repository.

To install ArgoCD via Helm:
```shell
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install \
  argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --version 5.52.0 \
  --values helm/argocd-values.yaml 
```

Note that in the `argocd-values.yaml` we set the server to insecure. This is because on a local network deployment, managing TLS is problematic as we do not "own" the hosts we are using (instead, we are arbitrarily defining them in our custom DNS records). We also set the server's service as a `LoadBalancer` and assign the static IP we have in our custom DNS configuration.

Next, we need to login. The initial password for the `admin` account is auto-generated and stored as clear text in the field `password` in a secret called `argocd-initial-admin-secret`:
```shell
echo $(kubectl get secret argocd-initial-admin-secret -n argocd -o json | jq -r '.data.password') | base64 -d
```

Once logged in, it's worth updating the password in the "User Info" settings panel.

#### ArgoCD In Action

